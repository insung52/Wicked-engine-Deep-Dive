# Appendix: DLL 경계 문제

> **DLL 기초 개념 참고** (exe/dll/lib 차이, export/import, inline 변수, 링킹): [09_dll_and_linking.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/09_dll_and_linking.md)

## DLL이란

**DLL(Dynamic Link Library)**은 런타임에 로드되는 별도의 실행 코드 덩어리다.
프로그램이 실행될 때 필요한 DLL 파일을 메모리에 올려서 사용한다.

```
단일 exe (WickedEngine):               다중 DLL (VizMotive):

┌──────────────────────────┐           ┌──────────────────────────────┐
│       program.exe        │           │         app.exe              │
│  (모든 코드가 한 덩어리)  │           │  (진입점, 사용자 코드)        │
└──────────────────────────┘           └──────┬──────────┬────────────┘
                                              │          │
                                     ┌────────▼──┐  ┌────▼──────────┐
                                     │Engine.dll │  │GBackendDX12   │
                                     │(EngineCore│  │.dll           │
                                     │렌더링 API)│  │(DX12 구현)    │
                                     └───────────┘  └───────────────┘
```

VizMotive의 DLL 구조:
- **Engine.dll** (`EngineCore/`) — 렌더링 추상화 레이어. `Texture`, `GPUBuffer` 같은 공개 타입 정의. 앱이 직접 사용하는 API.
- **GBackendDX12.dll** (`GraphicsBackends/`) — DX12 구현. `Texture_DX12`, `Resource_DX12` 같은 내부 구현 타입.
- **ShaderEngine.dll** — 셰이더 렌더러.

**핵심**: `Texture` 객체는 Engine.dll에 살고, 그 내부의 DX12 구현(`Texture_DX12`)은 GBackendDX12.dll에 산다.
`internal_state`(불투명 포인터)가 이 둘을 연결한다.

---

## 문제: C++ `inline` 변수는 DLL마다 별도 인스턴스

C++17의 `inline` 변수는 **같은 실행 파일 안에서는 하나의 인스턴스**만 존재한다.
하지만 DLL 경계를 넘으면 각 DLL마다 **별도의 인스턴스**가 생성된다.

```cpp
// Allocator.h (헤더에 inline으로 정의)
inline SharedBlockAllocator* block_allocators[256] = {};
```

```
단일 exe (WickedEngine):
  프로그램 전체에 block_allocators[] 가 딱 하나 ──── 모든 코드가 공유

다중 DLL (VizMotive에서 같은 방식 사용 시):
  Engine.dll이 로드될 때:
    → block_allocators[] 인스턴스 A 생성 (전부 nullptr)

  GBackendDX12.dll이 로드될 때:
    → block_allocators[] 인스턴스 B 생성 (전부 nullptr)
    → allocator 등록: 인스턴스 B[0] = &texture_allocator
```

두 DLL이 각자 자기 배열을 가지기 때문에 서로의 내용을 모른다.

---

## 문제 시나리오: WickedEngine 방식을 VizMotive에 그대로 쓰면

WickedEngine은 `shared_ptr` 안에 allocator_id(8비트)만 저장하고, 실제 allocator는 배열에서 조회한다:

```cpp
// WickedEngine 방식
struct shared_ptr {
    uint64_t handle;  // 상위 56비트: 포인터, 하위 8비트: allocator_id
};

SharedBlockAllocator* get_allocator() const {
    return block_allocators[handle & 0xFF];  // 전역 배열에서 조회
}
```

VizMotive에서 이걸 그대로 쓰면:

```
① GBackendDX12.dll에서 Texture_DX12 생성:
     → DX12.dll의 block_allocators[0] = &texture_allocator  (등록)
     → shared_ptr.handle = (ptr << 8) | 0                   (id=0 인코딩)

② Texture가 Engine.dll로 전달됨
     (tex.internal_state = shared_ptr{handle = ...})

③ Engine.dll 코드에서 Texture를 복사하거나 소멸시킬 때:
     allocator_id = handle & 0xFF = 0
     Engine.dll의 block_allocators[0] = nullptr  ← 비어있음!
                                                     (DLL마다 별도 배열!)
     → nullptr->inc_refcount() → 크래시
```

**이것이 "DLL 경계 문제"의 실체다.**
refcount 숫자 자체가 잘못 세지는 게 아니라, **allocator를 찾지 못하는 것**이 문제다.

---

## VizMotive의 소멸 순서와 문제와의 관계

### 실제 소멸 순서 코드

`VzEngineManager.cpp`의 `DeinitEngineLib()`:

```cpp
bool DeinitEngineLib()
{
    vzcompmanager::DestroyAll();           // ① 모든 컴포넌트/GPU 리소스 소멸
    graphicsBackend.pluginDeinitializer(); // ② GBackendDX12 소멸
    shaderEngine.pluginDeinitializer();    // ③ ShaderEngine 소멸
    // (DeinitEngineLib 반환 후 Engine.dll 자체 정리)
}
```

이것은 **DLL 파일 언로드 순서가 아니라, 각 서브시스템의 명시적 Deinit 함수 호출 순서**다.
교수님이 말씀하신 "소멸 순서: backend → shaderengine → engine"이 바로 이 ②③ 순서를 뜻한다.

### GPU 리소스 소멸(①)이 왜 먼저인가

```
① DestroyAll():
     Texture, GPUBuffer 등의 internal_state → 전부 .reset() → nullptr
     이 시점에 Texture_DX12의 refcount가 0이 되어 GBackendDX12.dll의 allocator가 해제

② graphicsBackend.pluginDeinitializer():
     GBackendDX12.dll의 GraphicsDevice 소멸
     이 시점엔 이미 internal_state가 없으므로 allocator는 안전하게 소멸 가능

③ shaderEngine.pluginDeinitializer():
     ShaderEngine.dll 정리
```

① → ② 순서 덕분에 **소멸 시점은 안전**하다. GPU 리소스가 먼저 정리되고 나서 backend가 소멸된다.

### 그런데 왜 WickedEngine 방식을 아직도 쓸 수 없는가

소멸 시점은 안전해도, 문제는 **프로그램 실행 중**에 발생한다.

Engine.dll 코드에서 Texture를 복사하는 순간 이미 크래시:

```cpp
// EngineCore/Components/TextureComponent.cpp:168
// (프로그램이 정상 동작하는 중, 아직 아무것도 소멸되지 않은 상태)
graphics::Texture texture = resource.GetTexture();
//                          ↑ Texture 복사 → shared_ptr 복사생성자 → inc_refcount 호출
//                            → Engine.dll의 block_allocators[id] 조회
//                            → 비어있음! → 크래시
```

DestroyAll()이 먼저 실행되든 나중에 실행되든, 평소 동작 중에 이미 터진다.

```
프로그램 실행 흐름:
  앱 시작 → [정상 동작 중] → 앱 종료
              ↑
              여기서 Texture 복사 → 크래시
              (DestroyAll/Deinit 순서와 무관)
```

---

## 해결책: allocator 포인터를 shared_ptr 안에 직접 저장

전역 배열 조회 대신, 생성 시점에 allocator 포인터를 shared_ptr 멤버로 직접 넣는다.

```cpp
// WickedEngine 방식 (8바이트): 전역 배열 의존 → DLL 경계에서 실패
struct shared_ptr {
    uint64_t handle;  // ptr | allocator_id
    // allocator 접근: block_allocators[handle & 0xFF]  ← 배열 조회
};

// VizMotive 방식 (16바이트): 직접 포인터 → DLL 어디서든 안전
struct shared_ptr {
    T* ptr;
    SharedBlockAllocator* allocator;  // GBackendDX12.dll 안의 allocator를 직접 가리킴
    // allocator 접근: this->allocator  ← 배열 조회 없음
};
```

```
GBackendDX12.dll에서 생성:
  shared_ptr.ptr       = &Texture_DX12_slot
  shared_ptr.allocator = &DX12.dll_내부_allocator  ← 포인터로 직접 저장

Engine.dll로 전달 후 복사/소멸:
  allocator->inc_refcount(ptr)  ← DX12.dll의 allocator를 직접 역참조
  allocator->dec_refcount(ptr)  ← 전역 배열 조회 없음 → 정상 동작
```

어느 DLL에서 소멸하더라도 `allocator` 포인터가 올바른 DLL의 allocator를 직접 가리키기 때문에 안전하다.

---

## 두 방식 비교

| | WickedEngine 방식 | VizMotive 방식 |
|---|---|---|
| `shared_ptr` 크기 | 8바이트 | 16바이트 |
| allocator 접근 | `block_allocators[id]` (전역 배열) | `this->allocator` (직접 포인터) |
| DLL 경계 | ❌ 크래시 | ✅ 안전 |
| 단일 exe | ✅ 정상 | ✅ 정상 |
| 메모리 오버헤드 | — | 8바이트/shared_ptr (GPU 리소스 특성상 무시 가능) |

> WickedEngine은 단일 exe 구조라 block_allocators[]가 하나만 존재한다. 그래서 문제없다.
> VizMotive는 다중 DLL 구조라 배열이 DLL마다 분리되어 문제가 생긴다.
