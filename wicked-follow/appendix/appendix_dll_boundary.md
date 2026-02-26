# Appendix: DLL 경계 문제

## DLL이란

**DLL(Dynamic Link Library)**은 런타임에 로드되는 별도의 실행 코드 조각이다.
여러 프로그램이 같은 DLL을 공유할 수 있고, DLL을 교체해도 프로그램 재컴파일이 불필요하다.

```
단일 exe 구조 (WickedEngine):           다중 DLL 구조 (VizMotive):
┌──────────────────────────┐            ┌─────────────┐  ┌──────────────┐
│       program.exe        │            │  Engine.dll  │  │  DX12.dll    │
│  (모든 코드가 한 덩어리)  │            │  (렌더링 API)│  │  (DX12 구현) │
└──────────────────────────┘            └──────┬───────┘  └──────┬───────┘
                                               │                 │
                                        ┌──────▼─────────────────▼──────┐
                                        │          app.exe               │
                                        └───────────────────────────────┘
```

VizMotive는 다중 DLL 구조를 사용한다:
- `Engine.dll` — 렌더링 추상화 레이어 (GBackend API)
- `DX12.dll` — DirectX 12 구현
- 각 DLL은 별도의 메모리 공간에 로드됨

---

## 문제: C++ `inline` 변수와 DLL

C++17의 `inline` 변수는 동일한 실행 파일 내에서는 **하나의 인스턴스만** 존재한다.
하지만 DLL 경계를 넘으면 각 DLL마다 **별도의 인스턴스**가 생성된다.

```cpp
// WickedEngine의 Allocator.h
inline SharedBlockAllocator* block_allocators[256] = {};  // ← inline 전역 배열
inline std::atomic<uint8_t> block_allocator_count{ 0 };
```

```
단일 exe (WickedEngine):
  program.exe 로드 시:
  block_allocators[] ─────────────────────────────── 하나만 존재
                           ↑
                모든 코드가 이 배열을 공유

다중 DLL (VizMotive에서 같은 방식 사용 시):
  Engine.dll 로드 시:
  block_allocators[] = { nullptr, ... }  ← 인스턴스 A (전부 비어있음)

  DX12.dll 로드 시:
  block_allocators[] = { nullptr, ... }  ← 인스턴스 B (전부 비어있음)
```

---

## 문제 시나리오

```
1. DX12.dll에서 Texture_DX12 allocator 등록:
   DX12.dll의 block_allocators[0] = &texture_allocator
   DX12.dll의 block_allocator_count = 1

2. DX12.dll에서 Texture 생성:
   shared_ptr handle = (texture_ptr << 8) | 0  ← allocator_id = 0 인코딩

3. Texture 객체가 Engine.dll로 전달됨
   (tex.internal_state = shared_ptr{handle = ...})

4. Engine.dll에서 tex.internal_state를 복사하거나 소멸시키려 할 때:
   allocator_id = handle & 0xFF = 0
   Engine.dll의 block_allocators[0] = nullptr  ← 비어있음!
                                                   ↑
   DX12.dll과 Engine.dll의 block_allocators[]는 서로 다른 인스턴스!

5. nullptr->inc_refcount() → 크래시
```

---

## VizMotive 해결책: allocator 포인터 직접 저장

전역 배열(DLL마다 별도 인스턴스)에 의존하는 대신,
`shared_ptr` 안에 allocator 포인터를 직접 저장한다.

```cpp
// WickedEngine: 8바이트, 전역 배열 의존
struct shared_ptr {
    uint64_t handle;  // ptr | allocator_id
    // allocator 접근: block_allocators[handle & 0xFF] (전역 배열)
};

// VizMotive: 16바이트, allocator 직접 참조
struct shared_ptr {
    T* ptr;
    SharedBlockAllocator* allocator;  // 포인터 직접 보유
    // allocator 접근: this->allocator (전역 배열 불필요)
};
```

```
DX12.dll에서 Texture 생성:
  shared_ptr.ptr       = &texture_slot
  shared_ptr.allocator = &DX12.dll내부의 texture_allocator  ← 포인터 직접 저장

Engine.dll에서 shared_ptr 소멸:
  this->allocator->dec_refcount(ptr)  ← 어느 DLL에서 호출해도
                                         DX12.dll의 allocator를 직접 가리킴
                                         전역 배열 조회 없음 → 정상 작동
```

---

## 트레이드오프

| | WickedEngine | VizMotive |
|---|---|---|
| `shared_ptr` 크기 | 8바이트 | 16바이트 |
| allocator 접근 | 전역 배열 인덱스 조회 | 포인터 직접 역참조 |
| DLL 경계 | ❌ 실패 | ✅ 정상 |
| 256바이트 정렬 요구 | 필요 (하위 8비트 ID 저장) | 불필요 |

10,000개 객체 기준 메모리 차이: 10,000 × 8바이트 = 약 80KB 추가.
그래픽스 엔진에서 무시 가능한 수준.
