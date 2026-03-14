# VizMotive: Custom Allocator & Pooled shared_ptr 설계

**[Notebooklm pdf 문서 : allocator_shaerd_ptr](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/pdf/allocator_shared_ptr.pdf)**

## 개요

VizMotive는 GPU 리소스 객체(`Texture_DX12`, `Resource_DX12` 등)를 `std::shared_ptr`로 관리하고 있었다.
이를 커스텀 block allocator 기반 shared_ptr로 교체하는 작업이다.

최종 구현은 WickedEngine의 8-byte handle 방식을 기반으로, **교수님 피드백에 따라
`block_allocators[]`를 `Allocator.cpp`(Engine.dll 단일 인스턴스)로 분리하여 DLL 경계 문제를 해결**한 형태다.

> **배경 개념 참고**
> - `shared_ptr` / `weak_ptr` / control block: [05_smart_pointers_and_raii.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/05_smart_pointers_and_raii.md)
> - Block Allocator / 메모리 풀링: [appendix_block_allocator.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_block_allocator.md)
> - DLL 기초 (exe/dll/lib, export/import, inline 변수): [09_dll_and_linking.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/09_dll_and_linking.md)
> - DLL 경계 문제: [appendix_dll_boundary.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_dll_boundary.md)
> - C++ `static` 세 가지 용도: [10_static.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/10_static.md)

---

## 변경 전 상태 (VizMotive 원본)

### internal_state란

VizMotive는 `Engine.dll`과 `DX12.dll`이 분리된 다중 DLL 구조다.
`Texture`, `GPUBuffer` 같은 공개 리소스 타입은 Engine.dll에 정의되어 있고,
`Texture_DX12`, `Resource_DX12` 같은 실제 DX12 구현은 DX12.dll 내부에만 존재한다.

`internal_state`는 이 둘을 연결하는 **불투명 포인터(opaque pointer)** 다.

```
Engine.dll 영역 (공개):
  Texture {
      shared_ptr<void> internal_state ─────────────┐
      TextureDesc desc;                             │  void*로 타입 숨김
  }                                                 │
                                                    ↓
DX12.dll 영역 (내부):
  Texture_DX12 {
      ComPtr<ID3D12Resource> resource;
      SingleDescriptor rtv, dsv;
      allocationhandler→...;
  }
```

- DX12.dll의 `CreateTexture()`에서 `Texture_DX12` 객체를 생성하고 `internal_state`에 저장
- Engine.dll은 `internal_state`가 `void*`이기 때문에 DX12 타입을 알 필요가 없음
- DX12.dll 안에서만 `to_internal()` 함수로 `void*` → `Texture_DX12*`로 꺼냄

```cpp
// DX12.dll 내부에서만 사용하는 캐스팅 함수
Texture_DX12* to_internal(const Texture* param)
{
    return static_cast<Texture_DX12*>(param->internal_state.get());
}
```

이 패턴 덕분에 Engine.dll은 DX12를 전혀 알지 못한 채로 GPU 리소스를 보유하고 전달할 수 있다.

---

### GPU 리소스 관리 구조

```cpp
// GBackend.h
struct GraphicsDeviceChild
{
    std::shared_ptr<void> internal_state;
    inline bool IsValid() const { return internal_state != nullptr; }
};

// GraphicsDevice_DX12.h
std::shared_ptr<AllocationHandler> allocationhandler;
std::vector<std::unique_ptr<CommandList_DX12>> commandlists;

// GraphicsDevice_DX12.cpp
allocationhandler = std::make_shared<AllocationHandler>();

auto internal_state = std::make_shared<Texture_DX12>();
auto internal_state = std::make_shared<Resource_DX12>();
auto internal_state = std::make_shared<PipelineState_DX12>();
// ... 총 13곳
```

---

## 문제점

### 1. 힙 할당 오버헤드

> 힙 할당/단편화 개념: [appendix_block_allocator.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_block_allocator.md)

`std::make_shared<T>()` 호출 시마다 OS에 힙 메모리를 요청한다.

```
매 프레임:
  CreateTexture()  → std::make_shared<Texture_DX12>()  → 힙 할당 ①
  CreateBuffer()   → std::make_shared<Resource_DX12>() → 힙 할당 ②
  ...
  해제 시          → delete                             → 힙 해제, 단편화
```

- 할당/해제 비용: ~100-1000 cycles/회
- 메모리가 흩어짐 → 캐시 미스 증가
- 메모리 단편화 누적

### 2. DLL 경계 문제 — VizMotive 아키텍처의 제약

> DLL 경계 문제 상세: [appendix_dll_boundary.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_dll_boundary.md)

VizMotive는 `Engine.dll` + `DX12.dll` 등 **다중 DLL 구조**다.
GPU 리소스는 DX12.dll에서 생성되고, `internal_state`는 Engine.dll 코드에서 복사/소멸된다.
이때 shared_ptr가 allocator를 어떻게 찾는지가 핵심 문제다.

WickedEngine의 방식(참고용):
```cpp
// WickedEngine: 전역 배열로 allocator 조회
struct shared_ptr { uint64_t handle; };  // 상위 56비트: ptr, 하위 8비트: allocator ID
SharedBlockAllocator* get_allocator() const {
    return block_allocators[handle & 0xFF];  // inline 전역 배열 조회
}
```

```
VizMotive에서 이 방식을 그대로 쓰면:
  DX12.dll의 block_allocators[]  ← allocator 등록
  Engine.dll의 block_allocators[] ← 비어있음! (DLL마다 별도 인스턴스)

  → Engine.dll에서 dec_refcount 시 nullptr 역참조 → 크래시
```

#### Engine.dll에서 실제로 Texture를 복사/소멸하는가?

코드에서 직접 확인:

```cpp
// EngineCore/Components/TextureComponent.cpp:168
graphics::Texture texture = resource.GetTexture();  // value copy → inc_refcount 발생

// EngineCore/Components/GComponents.h:444, 674
graphics::Texture volumeMinMaxBlocks_ = {};  // 멤버 변수로 소유
graphics::Texture texture;                   // 컴포넌트 소멸 시 dec_refcount 발생
```

Engine.dll(EngineCore) 코드에서 `Texture`를 값 복사하거나 멤버로 소유하는 곳이 실제로 존재한다.
WickedEngine 방식은 이 시점에 `block_allocators[id]`를 Engine.dll에서 조회하므로 크래시가 발생한다.

---

## 설계 선택

### 문제 1 — Block Allocator 풀링 채택

> 상세 개념: [appendix_block_allocator.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_block_allocator.md)

같은 타입 객체를 256개씩 연속 메모리 블록에 미리 할당해두고, `free_list`로 O(1) 재사용.
이 부분은 WickedEngine과 동일한 방향이며, 대안이 없다.

```
std::shared_ptr 방식 (기존):
  ┌───┐     ┌───┐     ┌───┐
  │ A │     │ B │     │ C │   ← 개별 힙 할당, 흩어진 메모리
  └───┘     └───┘     └───┘

Block Allocator 방식 (채택):
  ┌───┬───┬───┬───┬───┬───┬───┬───┬─────────────────┐
  │ A │ B │ C │ D │ E │ F │ G │ H │ ... (256개)      │  ← 연속 메모리
  └───┴───┴───┴───┴───┴───┴───┴───┴─────────────────┘
     ↑
  free_list.pop() → O(1) 할당
  free_list.push() → O(1) 해제
```

### 문제 2 — WickedEngine handle 방식 + Allocator.cpp 분리 채택

> DLL별 전역 배열 문제 상세: [appendix_dll_boundary.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_dll_boundary.md)
> `static` 개념: [10_static.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/10_static.md)

WickedEngine의 `uint64_t handle` 방식(8바이트)을 그대로 채택하되,
**`block_allocators[]`의 위치를 `Allocator.h`(inline 전역)에서 `Allocator.cpp`(Engine.dll 단일 인스턴스)로 이동**하여 DLL 경계 문제를 해결한다.

#### 문제: inline 전역 변수의 DLL별 별도 인스턴스

```cpp
// 변경 전 Allocator.h — inline이라 DLL마다 별도 인스턴스 생성됨
inline SharedBlockAllocator* block_allocators[256] = {};
inline std::atomic<uint8_t> next_allocator_id{ 0 };
```

```
GBackendDX12.dll: block_allocators[] 인스턴스 A  ← allocator 등록
Engine.dll:       block_allocators[] 인스턴스 B  ← 비어있음!

Engine.dll에서 inc_refcount → block_allocators[id] 조회 → 인스턴스 B → nullptr → 크래시
```

상세 흐름:
```
TextureComponent.cpp에서 graphics::Texture texture = resource.GetTexture();
  → shared_ptr 복사 → inc_refcount 호출
  → Engine.dll의 block_allocators[id] 조회  ← 비어있음!
  → 크래시  (정상 런타임 동작 중에 발생, DestroyAll 순서와 무관)
```

소멸 순서(`DestroyAll → backend Deinit`)가 올바르더라도,
런타임 도중 Engine.dll에서 Texture를 복사하는 순간 발생하는 크래시는 막을 수 없다.

#### 해결: block_allocators[]를 Allocator.cpp로 이동 + UTIL_EXPORT

`Allocator.cpp`를 Engine.dll에서 단 한 번 컴파일 → 배열이 Engine.dll 메모리에 **하나만** 존재.
3개의 `UTIL_EXPORT` 함수로 배열을 내보냄 → 모든 DLL이 같은 배열을 사용.

```
Engine.dll: block_allocators[] 하나만 존재 (Allocator.cpp에서 static으로 정의)
               ↑ 등록                      ↑ 조회
GBackendDX12.dll: register_shared_block_allocator() 호출 → 인스턴스 등록
Engine.dll:       get_shared_block_allocator(id) 호출  → 같은 배열에서 조회
→ 전부 같은 배열에 접근 → 크래시 없음
```

**`static`의 역할**: `Allocator.cpp`에서 `static SharedBlockAllocator* block_allocators[256]`으로 선언하면
이 심볼이 파일 외부로 노출되지 않는다(내부 링크). 외부에서는 오직 `UTIL_EXPORT` 함수를 통해서만 접근.
→ [10_static.md 파일 스코프 static 참고](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/10_static.md)

#### handle 인코딩 방식 (8바이트)

```cpp
// 상위 56비트: 포인터, 하위 8비트: allocator_id
uint64_t handle = reinterpret_cast<uint64_t>(ptr) | uint64_t(allocator_id);

// 조회:
T* get_ptr() const { return reinterpret_cast<T*>(handle & (~uint64_t(0xFF))); }
SharedBlockAllocator* get_allocator() const { return get_shared_block_allocator(uint8_t(handle)); }
bool IsValid() const { return get_ptr() != nullptr && get_allocator() != nullptr; }
```

전제 조건: `RawStruct`는 반드시 `alignas(256)` — 포인터 하위 8비트가 항상 0이어야
allocator_id를 OR로 끼워 넣을 수 있다.
`SharedBlockAllocatorImpl`(블록 풀)은 이미 `alignas(256)` 적용됨.
`SharedHeapAllocator`(힙 개별 할당)도 동일하게 `alignas(256)` 추가.

---

## 변경 사항

### 1. EngineCore/Utils/Allocator.h 변경

Block Allocator 기반 커스텀 shared_ptr 시스템.
네임스페이스: `vz::allocator`.

변경 이력이 2단계임:
- **1차 구현**: Allocator.h 신규 추가. `inline` 전역 변수 + 16바이트 직접 포인터 방식 (`T* ptr + SharedBlockAllocator* allocator`)
- **교수님 피드백 후 (최종)**: `inline` 전역 변수 제거 → `UTIL_EXPORT` 함수 선언으로 교체. `shared_ptr`: 16바이트 직접 포인터 → 8바이트 handle 인코딩으로 변경.

#### 1-1. SharedBlockAllocator 인터페이스

변경 없음. 모든 타입별 allocator가 구현해야 하는 공통 인터페이스.

```cpp
struct SharedBlockAllocator
{
    virtual void     init_refcount(void* ptr) = 0;
    virtual uint32_t get_refcount(void* ptr) = 0;
    virtual uint32_t inc_refcount(void* ptr) = 0;
    virtual uint32_t dec_refcount(void* ptr) = 0;
    virtual uint32_t get_refcount_weak(void* ptr) = 0;
    virtual uint32_t inc_refcount_weak(void* ptr) = 0;
    virtual uint32_t dec_refcount_weak(void* ptr) = 0;
    virtual bool     try_inc_refcount(void* ptr) = 0;
};
```

#### 1-2. UTIL_EXPORT 함수 선언 (inline 전역 변수 제거)

> 여기서 "변경 전"은 **원본 VizMotive가 아니라 1차 구현 상태**다.
> 원본 VizMotive에는 Allocator.h 자체가 없었으며, `std::shared_ptr<void>`를 사용했다.
> 1차 구현에서 Allocator.h를 신규 추가하면서 inline 전역 변수를 헤더에 뒀다가,
> 교수님 피드백으로 Allocator.cpp로 이동하는 것이 이 섹션의 변경이다.

```cpp
// 1차 구현 Allocator.h — inline이라 DLL마다 별도 인스턴스 (문제 원인)
inline SharedBlockAllocator* block_allocators[256] = {};
inline std::atomic<uint8_t> next_allocator_id{ 0 };
inline uint8_t register_shared_block_allocator(SharedBlockAllocator* allocator) { ... }
inline SharedBlockAllocator* get_shared_block_allocator(uint8_t id) { ... }

// 최종 Allocator.h — 선언만. 구현은 Allocator.cpp (Engine.dll 단일 인스턴스)
UTIL_EXPORT uint8_t register_shared_block_allocator(SharedBlockAllocator* allocator);
UTIL_EXPORT uint8_t get_shared_block_allocator_count();
UTIL_EXPORT SharedBlockAllocator* get_shared_block_allocator(uint8_t id);
```

`UTIL_EXPORT`는 `__declspec(dllexport)`의 매크로다.
이 함수들은 Engine.dll에서 내보내지며, GBackendDX12.dll을 포함한 모든 DLL이 import해서 사용한다.
→ [09_dll_and_linking.md UTIL_EXPORT 참고](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/09_dll_and_linking.md)

#### 1-3. shared_ptr / weak_ptr — 8바이트 handle로 변경

> shared_ptr / weak_ptr / control block 개념 : [05_smart_pointers_and_raii.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/05_smart_pointers_and_raii.md)

```cpp
template<typename T>
struct shared_ptr
{
    uint64_t handle = 0;
    // 상위 56비트: 포인터 주소 (alignas(256) → 하위 8비트 = 0)
    // 하위  8비트: allocator_id

    inline T* get_ptr() const
    {
        return reinterpret_cast<T*>(handle & (~uint64_t(0xFF)));
    }
    inline SharedBlockAllocator* get_allocator() const
    {
        return get_shared_block_allocator(uint8_t(handle));
    }
    bool IsValid() const { return get_ptr() != nullptr && get_allocator() != nullptr; }

    // 복사: get_allocator()->inc_refcount(get_ptr())
    // 소멸: get_allocator()->dec_refcount(get_ptr())
    //       → refcount 0이면 ~T() 호출 + weak refcount 감소
    //       → weak refcount 0이면 메모리를 free_list로 반환
};

template<typename T>
struct weak_ptr
{
    uint64_t handle = 0;

    inline T* get_ptr() const { return reinterpret_cast<T*>(handle & (~uint64_t(0xFF))); }
    inline SharedBlockAllocator* get_allocator() const
    {
        return get_shared_block_allocator(uint8_t(handle));
    }
    bool IsValid() const { return get_ptr() != nullptr && get_allocator() != nullptr; }

    shared_ptr<T> lock()
    {
        if (!IsValid()) return {};
        if (get_allocator()->try_inc_refcount(get_ptr()))
        {
            shared_ptr<T> ret;
            ret.handle = handle;
            return ret;
        }
        return {};
    }
};
```

1차 변경(직접 포인터)과 비교:
| | 1차 변경 (16바이트) | 변경 후 (8바이트) |
|---|---|---|
| 크기 | `T* ptr` + `SharedBlockAllocator* allocator` | `uint64_t handle` |
| 포인터 복원 | `ptr` 직접 사용 | `handle & ~0xFF` |
| allocator 조회 | `allocator` 직접 사용 | `get_shared_block_allocator(uint8_t(handle))` (UTIL_EXPORT 호출) |
| DLL 경계 | allocator*가 어느 DLL 메모리인지 무관 | Engine.dll의 단일 배열 조회 |

#### 1-4. SharedBlockAllocatorImpl — allocate() handle 설정

> free_list, placement new, RawStruct 구조 상세: [appendix_block_allocator.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_block_allocator.md)

```cpp
template<typename T, size_t block_size = 256>
struct SharedBlockAllocatorImpl final : public SharedBlockAllocator
{
    struct alignas(std::max(size_t(256), alignof(T))) RawStruct
    {
        uint8_t data[sizeof(T)];
        std::atomic<uint32_t> refcount;
        std::atomic<uint32_t> refcount_weak;
    };
    // alignas(256): 포인터 하위 8비트 = 0 → allocator_id를 OR로 끼워넣을 수 있음

    uint8_t allocator_id;
    // 생성 시 register_shared_block_allocator(this) 호출해서 ID 받음

    template<typename... ARG>
    inline shared_ptr<T> allocate(ARG&&... args)
    {
        // ... free_list에서 꺼내기, placement new ...

        shared_ptr<T> result;
        result.handle = reinterpret_cast<uint64_t>(raw) | uint64_t(allocator_id);
        //              ↑ alignas(256) 덕분에 하위 8비트 = 0 → OR 안전
        return result;
    }
};
```

#### 1-5. SharedHeapAllocator — alignas(256) 추가

`AllocationHandler`처럼 디바이스당 1개만 생성되는 객체용.

```cpp
template<typename T>
struct SharedHeapAllocator final : public SharedBlockAllocator
{
    struct alignas(256) RawStruct   // ← alignas(256) 추가 (handle 인코딩 전제 조건)
    {
        uint8_t data[sizeof(T)];
        std::atomic<uint32_t> refcount;
        std::atomic<uint32_t> refcount_weak;
    };

    uint8_t allocator_id;
    SharedHeapAllocator() { allocator_id = register_shared_block_allocator(this); }
    // 생성자에서 Engine.dll의 배열에 자신을 등록

    template<typename... ARG>
    inline shared_ptr<T> allocate(ARG&&... args)
    {
        RawStruct* raw = new RawStruct;       // 개별 힙 할당
        new (raw) T(std::forward<ARG>(args)...);
        init_refcount(raw);

        shared_ptr<T> result;
        result.handle = reinterpret_cast<uint64_t>(raw) | uint64_t(allocator_id);
        return result;
    }
    void reclaim(void* ptr) { delete static_cast<RawStruct*>(ptr); }
};
```

#### 1-6. 헬퍼 함수

```cpp
// 빈번하게 생성/삭제: 블록 풀링
template<typename T, size_t block_size = 256, typename... ARG>
inline shared_ptr<T> make_shared(ARG&&... args)
{
    static SharedBlockAllocatorImpl<T, block_size>* allocator
        = new SharedBlockAllocatorImpl<T, block_size>;
    // ↑ 함수 내 static: 처음 호출 시 한 번만 생성 (C++11 thread-safe)
    // 생성 시 register_shared_block_allocator() 호출 → Engine.dll 배열에 등록
    return allocator->allocate(std::forward<ARG>(args)...);
}

// 한 번만 생성/오래 유지: 힙 직접 할당
template<typename T, typename... ARG>
inline shared_ptr<T> make_shared_single(ARG&&... args)
{
    static SharedHeapAllocator<T>* allocator = new SharedHeapAllocator<T>;
    // ↑ SharedHeapAllocator 생성자에서 register_shared_block_allocator() 호출
    return allocator->allocate(std::forward<ARG>(args)...);
}
```

함수 내 `static` 사용: [10_static.md 함수 내 static 참고](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/10_static.md)

---

### 1'. EngineCore/Utils/Allocator.cpp 신규 추가

Engine.dll에서 단 한 번 컴파일. `block_allocators[]`의 **단일 인스턴스** 보관.

```cpp
#include "Allocator.h"

// static: 파일 외부로 심볼 노출 안 함 (internal linkage)
// → 외부에서는 오직 아래 UTIL_EXPORT 함수를 통해서만 접근 가능
static SharedBlockAllocator* block_allocators[256] = {};
static std::atomic<uint8_t> next_allocator_id{ 0 };

uint8_t register_shared_block_allocator(SharedBlockAllocator* allocator)
{
    uint8_t id = next_allocator_id.fetch_add(1);
    assert(id < 256);
    block_allocators[id] = allocator;
    return id;
}

uint8_t get_shared_block_allocator_count()
{
    return next_allocator_id.load();
}

SharedBlockAllocator* get_shared_block_allocator(uint8_t id)
{
    return block_allocators[id];
}
```

Engine_SOURCE.vcxitems에 추가 (빌드에 포함):
```xml
<ClCompile Include="$(MSBuildThisFileDirectory)Utils\Allocator.cpp" />
```

Engine_SOURCE.vcxitems.filters:
- `Allocator.h`: `Public (APIs managed by core)` (외부에 UTIL_EXPORT 함수 노출)
- `Allocator.cpp`: `Private` (배열 단일 인스턴스, 직접 접근 불가)

---

### 2. GBackend.h 변경

Allocator.h에 시스템을 만들었으면, 실제 GPU 리소스 타입에 연결해야 한다.

#### 2-1. internal_state 타입 변경

3단계 변화가 있었다:

```cpp
// 원본 VizMotive
struct GraphicsDeviceChild
{
    std::shared_ptr<void> internal_state;
    inline bool IsValid() const { return internal_state != nullptr; }
};

// 1차 구현 (16바이트 직접 포인터 방식)
struct GraphicsDeviceChild
{
    vz::allocator::shared_ptr<void> internal_state;  // { T* ptr; allocator* allocator; }
    constexpr bool IsValid() const { return internal_state.IsValid(); }
    // IsValid() = ptr != nullptr && allocator != nullptr → constexpr 가능
};

// 최종 (교수님 피드백, 8바이트 handle 방식)
struct GraphicsDeviceChild
{
    vz::allocator::shared_ptr<void> internal_state;  // { uint64_t handle; }
    bool IsValid() const { return internal_state.IsValid(); }
    // ↑ constexpr 제거 이유:
    //   IsValid() → get_allocator() → get_shared_block_allocator() 호출
    //   get_shared_block_allocator()는 UTIL_EXPORT 외부 함수 → constexpr 불가 (C3615)
};
```

변경 대상: `GraphicsDeviceChild`, `GPUResource`, `Sampler`, `Shader`, `PipelineState`,
`RaytracingPipelineState`, `SwapChain` 등 `internal_state` 멤버를 가진 모든 구조체.

#### 2-2. placement new 허용

Block Allocator는 내부적으로 `new (ptr) T()` (placement new)로 객체를 생성한다.
이전에 vtable 제거 작업(커밋 #7)에서 모든 `operator new`를 `= delete`로 막았는데,
placement new까지 함께 막혀버렸다. placement new만 다시 허용한다.

```cpp
// 변경 전 (placement new도 금지된 상태)
struct GPUBuffer final : public GPUResource
{
    static void* operator new(size_t) = delete;
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;
    static void* operator new(size_t, void*) = delete;    // ← BlockAllocator에서 필요한데 막혀있음
    static void* operator new[](size_t, void*) = delete;
};

// 변경 후
struct GPUBuffer final : public GPUResource
{
    static void* operator new(size_t) = delete;         // 힙 할당 금지 유지
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;
    static void* operator new(size_t, void* p) noexcept { return p; }    // ✅ 허용
    static void* operator new[](size_t, void* p) noexcept { return p; }
};
```

변경 대상: `GPUBuffer`, `Texture`, `RaytracingAccelerationStructure`

---

### 3. GraphicsDevice_DX12.h + .cpp 변경

`internal_state`가 `vz::allocator::shared_ptr<void>`로 바뀌었으므로,
이 타입을 사용하는 모든 DX12 관련 코드도 함께 수정해야 한다.

#### 3-1. GraphicsDevice_DX12.h — allocationhandler 타입 변경

```cpp
// 변경 전
std::shared_ptr<AllocationHandler> allocationhandler;

// 변경 후
vz::allocator::shared_ptr<AllocationHandler> allocationhandler;
```

#### 3-2. GraphicsDevice_DX12.cpp — 내부 구조체 allocationhandler 타입 변경

`AllocationHandler`를 공유 소유하는 내부 DX12 구조체들도 동일하게 변경.

```cpp
// 변경 전 (8개 구조체 공통 패턴)
struct Resource_DX12
{
    std::shared_ptr<GraphicsDevice_DX12::AllocationHandler> allocationhandler;
};

// 변경 후
struct Resource_DX12
{
    vz::allocator::shared_ptr<GraphicsDevice_DX12::AllocationHandler> allocationhandler;
};
```

변경 대상: `SingleDescriptor`, `Resource_DX12`, `Sampler_DX12`, `QueryHeap_DX12`,
`PipelineState_DX12`, `RTPipelineState_DX12`, `SwapChain_DX12`, `VideoDecoder_DX12`

#### 3-3. GraphicsDevice_DX12.cpp — make_shared 호출 변경

```cpp
// 변경 전
allocationhandler = std::make_shared<AllocationHandler>();
// AllocationHandler는 디바이스당 1개 → 블록 풀링(256슬롯 중 1개만 사용) 낭비
// → make_shared_single 사용

// 변경 후
allocationhandler = vz::allocator::make_shared_single<AllocationHandler>();
```

```cpp
// 변경 전 (나머지 DX12 내부 타입들 — 13곳)
auto internal_state = std::make_shared<Texture_DX12>();
auto internal_state = std::make_shared<Resource_DX12>();
// ...

// 변경 후 (블록 풀링 사용)
auto internal_state = vz::allocator::make_shared<Texture_DX12>();
auto internal_state = vz::allocator::make_shared<Resource_DX12>();
// ...
```

전체 변경 목록:

| 기존 | 변경 후 |
|------|---------|
| `std::make_shared<AllocationHandler>()` | `vz::allocator::make_shared_single<AllocationHandler>()` |
| `std::make_shared<SwapChain_DX12>()` | `vz::allocator::make_shared<SwapChain_DX12>()` |
| `std::make_shared<Resource_DX12>()` × 4 | `vz::allocator::make_shared<Resource_DX12>()` |
| `std::make_shared<Texture_DX12>()` × 2 | `vz::allocator::make_shared<Texture_DX12>()` |
| `std::make_shared<PipelineState_DX12>()` × 2 | `vz::allocator::make_shared<PipelineState_DX12>()` |
| `std::make_shared<Sampler_DX12>()` | `vz::allocator::make_shared<Sampler_DX12>()` |
| `std::make_shared<QueryHeap_DX12>()` | `vz::allocator::make_shared<QueryHeap_DX12>()` |
| `std::make_shared<BVH_DX12>()` | `vz::allocator::make_shared<BVH_DX12>()` |
| `std::make_shared<RTPipelineState_DX12>()` | `vz::allocator::make_shared<RTPipelineState_DX12>()` |
| `std::make_shared<VideoDecoder_DX12>()` | `vz::allocator::make_shared<VideoDecoder_DX12>()` |

---

### 4. CommandList 블록 풀링

CommandList도 같은 이유로 블록 풀링이 유리하다.
매 프레임 여러 커맨드 리스트가 생성/재사용되므로 힙 할당 오버헤드가 누적된다.
shared_ptr가 아닌 raw pointer + BlockAllocator 직접 사용.

#### 4-1. GraphicsDevice_DX12.h 변경

```cpp
// 변경 전
std::vector<std::unique_ptr<CommandList_DX12>> commandlists;

// 변경 후
vz::allocator::BlockAllocator<CommandList_DX12, 64> cmd_allocator;
std::vector<CommandList_DX12*> commandlists;  // raw pointer (shared_ptr 불필요)
```

#### 4-2. GraphicsDevice_DX12.cpp — BeginCommandList() 변경

```cpp
// 변경 전
if (cmd_current >= commandlists.size())
    commandlists.push_back(std::make_unique<CommandList_DX12>());

CommandList cmd;
cmd.internal_state = commandlists[cmd_current].get();  // .get() 필요

// 변경 후
if (cmd_current >= commandlists.size())
    commandlists.push_back(cmd_allocator.allocate());   // O(1) 풀 할당

CommandList cmd;
cmd.internal_state = commandlists[cmd_current];         // raw pointer 직접 사용
```

#### 4-3. GraphicsDevice_DX12.cpp — 소멸자에 정리 코드 추가

`unique_ptr`는 벡터 소멸 시 자동으로 소멸자를 호출하지만,
raw pointer는 그렇지 않다. 명시적으로 정리해야 한다.

```cpp
// 소멸자에 추가 (WaitForGPU() 직후)
for (auto* cmdlist : commandlists)
{
    cmd_allocator.free(cmdlist);  // ~CommandList_DX12() 호출 + free_list 반환
}
commandlists.clear();
// cmd_allocator 자체는 이후 자동 소멸 (블록 메모리 해제)
```

> `cmd_allocator.free()` 누락 시: `CommandList_DX12`의 소멸자가 호출되지 않아
> `ID3D12GraphicsCommandList` 등 COM 객체가 해제되지 않는 리소스 누수 발생.

---

## 변경 전후 비교 표

> 3단계 변화 기준. "← 동일"은 해당 단계에서 변경 없음을 뜻한다.

| 항목 | 원본 VizMotive | 1차 구현 | 최종 (교수님 피드백) |
|------|---------------|---------|---------------------|
| `internal_state` 타입 | `std::shared_ptr<void>` | `vz::allocator::shared_ptr<void>` | ← 동일 |
| `IsValid()` | `internal_state != nullptr` | `constexpr bool IsValid()` | `bool IsValid()` (`constexpr` 제거) |
| `shared_ptr` 크기 | 16바이트 (ptr + control_block*) | 16바이트 (T* ptr + allocator*) | **8바이트 (uint64_t handle)** |
| `block_allocators[]` 위치 | 없음 | Allocator.h `inline` 전역 (DLL별 별도 인스턴스) | Allocator.cpp `static` (Engine.dll 단일 인스턴스) |
| allocator 조회 방식 | 표준 shared_ptr 내부 | `allocator` 포인터 직접 역참조 | `get_shared_block_allocator()` (UTIL_EXPORT 호출) |
| GPU 객체 할당 방식 | 힙 직접 할당 (`std::make_shared`) | 블록 풀 (`free_list.pop()`) | ← 동일 |
| `allocationhandler` 타입 | `std::shared_ptr<AllocationHandler>` | `vz::allocator::shared_ptr<AllocationHandler>` | ← 동일 |
| DX12 객체 생성 | `std::make_shared<T_DX12>()` | `vz::allocator::make_shared<T_DX12>()` | ← 동일 |
| `AllocationHandler` 생성 | `std::make_shared<AllocationHandler>()` | `vz::allocator::make_shared_single<AllocationHandler>()` | ← 동일 |
| CommandList 컨테이너 | `vector<unique_ptr<CommandList_DX12>>` | `BlockAllocator<>` + `vector<CommandList_DX12*>` | ← 동일 |
| CommandList 정리 | 자동 (unique_ptr 소멸) | 수동 (`cmd_allocator.free(p)` 루프) | ← 동일 |
| placement new (`GPUBuffer` 등) | ❌ 금지 | ✅ 허용 | ← 동일 |
| `SharedHeapAllocator::RawStruct` | 없음 | 기본 정렬 | `alignas(256)` 추가 (handle 인코딩 전제 조건) |

---

## 보충: weak_ptr 버그 수정 과정

커스텀 weak_ptr::lock()은 두 개의 버그를 거쳐 최종 구현에 도달했다.
실제 코드에는 최종 버전만 반영된다.

```
커밋 #9: 이중 소멸자 호출 수정 (단일 스레드)
  문제: lock() 내부에서 inc → refcount 0 확인 → dec 할 때, dec가 ~T() 재호출
  해결: dec_refcount(ptr, destruct_on_zero=false)
  한계: 멀티스레드에서 inc~dec 사이에 메모리 재사용 가능 → race condition 미해결

커밋 #10: destruct_on_zero 기본값 수정 (false → true)

커밋 #19: race condition 완전 해결
  문제: inc와 check 사이에 메모리가 반환/재할당되면 다른 객체의 refcount를 건드림
  해결: try_inc_refcount() — CAS로 "0이 아닐 때만 원자적으로 증가"
  부수: destruct_on_zero 파라미터 불필요해져서 제거
```

최종 `weak_ptr::lock()`:
```cpp
shared_ptr<T> lock()
{
    if (!IsValid()) return {};
    if (get_allocator()->try_inc_refcount(get_ptr()))  // 원자적: 0이 아니면 증가, 아니면 아무것도 안 함
    {
        shared_ptr<T> ret;
        ret.handle = handle;
        return ret;
    }
    return {};  // 실패 시 롤백 불필요
}
```

---

## 참고: WickedEngine 구현 커밋 (분석 참고용)

아래 문서들은 WickedEngine이 어떻게 구현했는지를 커밋 단위로 분석한 기록이다.
VizMotive의 구현 방식과 다를 수 있으며, 설계 결정의 배경을 이해하는 참고 자료로 활용한다.

- [dx2 #8 pooled shared_ptr](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/08_7a5ea2f6_pooled_shared_ptr.md)
- [#9 weak_ptr double destruct](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/09_e1dc87e4_weak_ptr_double_destruct.md)
- [#11 placement new](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/11_90e94ed4_placement_new_allowed.md)
- [#12 SharedHeapAllocator](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/12_ffcc3abd_shared_heap_allocator.md)
- [#19 weak_ptr race condition](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/19_be8c766e_weak_ptr_race_condition.md)
- [#22 CommandList BlockAllocator](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/22_43b93085_block_allocator_commandlist.md)
- [dx2 #3 alignas](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/03_a6adfc12_block_allocator_alignment.md)
