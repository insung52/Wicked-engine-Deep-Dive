# VizMotive: Custom Allocator & Pooled shared_ptr 설계

## 개요

VizMotive는 GPU 리소스 객체(`Texture_DX12`, `Resource_DX12` 등)를 `std::shared_ptr`로 관리하고 있었다.
이를 커스텀 block allocator 기반 shared_ptr로 교체하는 작업이다.

이 문서는 WickedEngine의 구현을 참고하되, **VizMotive의 아키텍처 제약(다중 DLL 구조)에 맞게
독립적으로 설계를 검토하고 채택한 방식을 기록한다.**
WickedEngine이 선택한 방식과 다른 부분이 있으며, 각 선택의 이유를 함께 설명한다.

> **배경 개념 참고**
> - `shared_ptr` / `weak_ptr` / control block: [05_smart_pointers_and_raii.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/05_smart_pointers_and_raii.md)
> - Block Allocator / 메모리 풀링: [appendix_block_allocator.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_block_allocator.md)
> - DLL 경계 문제: [appendix_dll_boundary.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_dll_boundary.md)

---

## 변경 전 상태 (VizMotive 원본)

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

### 문제 2 — allocator 포인터 직접 저장 (VizMotive 독자 설계)

> DLL별 전역 배열 문제와 해결책 상세: [appendix_dll_boundary.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_dll_boundary.md)

**WickedEngine과 다른 지점**: shared_ptr 안에 allocator 포인터를 직접 저장한다.

```cpp
// WickedEngine: 8바이트, 전역 배열 의존
struct shared_ptr { uint64_t handle; };  // 상위 56비트: ptr, 하위 8비트: allocator_id

// VizMotive: 16바이트, allocator 직접 참조 (채택)
struct shared_ptr { T* ptr; SharedBlockAllocator* allocator; };
//                              ↑ DLL 어느 쪽에서 소멸해도 이 포인터를 직접 역참조
```

#### 왜 WickedEngine 방식(8바이트)을 채택하지 않는가

8바이트를 유지하면서 DLL-safe하게 만드는 방법도 이론적으로는 존재한다:

| 방법 | 내용 | 문제 |
|---|---|---|
| DLL-exported 레지스트리 | Engine.dll이 배열을 export, DX12.dll이 import | Engine.dll ↔ DX12.dll 강결합. 추상화 계층 붕괴 |
| 블록 헤더 역산 | 슬롯 포인터 마스킹으로 블록 헤더(allocator*) 찾기 | 블록 전체를 수십~수백KB 경계로 정렬해야 함 → 메모리 낭비 |

어느 방법도 VizMotive의 아키텍처(Engine.dll이 DX12를 모르는 추상화 레이어)와 맞지 않는다.

#### 16바이트 overhead는 실제로 문제인가

GPU 리소스 특성상 크기 차이는 무시 가능하다:

```
동시에 살아있는 GPU 리소스 수: ~1,000 ~ 10,000개
리소스 객체 자체 크기: 수백 바이트 (Texture_DX12, Resource_DX12 등)
shared_ptr 크기 차이:  8바이트 × 10,000개 = 80KB

GPU 애플리케이션 총 메모리: 수 GB
→ 80KB 추가는 무시 가능한 수준
```

**채택**: 16바이트 직접 포인터 방식. DLL-safe + 단순한 구현 + 의미 있는 성능 손실 없음.

---

## 변경 사항

### 1. EngineCore/Utils/Allocator.h 신규 추가

Block Allocator 기반 커스텀 shared_ptr 시스템 전체를 신규 파일로 작성.
네임스페이스: `vz::allocator`.

#### 1-1. SharedBlockAllocator 인터페이스

모든 타입별 allocator가 구현해야 하는 공통 인터페이스.

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

#### 1-2. shared_ptr / weak_ptr

> shared_ptr / weak_ptr / control block 개념 : [05_smart_pointers_and_raii.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/lan/c%2B%2B/05_smart_pointers_and_raii.md)

allocator 포인터를 멤버로 직접 보유.

```cpp
template<typename T>
struct shared_ptr
{
    T* ptr = nullptr;
    SharedBlockAllocator* allocator = nullptr;  // 직접 저장 (DLL 경계 안전)

    constexpr bool IsValid() const { return ptr != nullptr && allocator != nullptr; }

    // 복사: allocator->inc_refcount(ptr)
    // 소멸: allocator->dec_refcount(ptr)
    //       → refcount 0이면 ~T() 호출 + weak refcount 감소
    //       → weak refcount 0이면 메모리를 free_list로 반환
};

template<typename T>
struct weak_ptr
{
    T* ptr = nullptr;
    SharedBlockAllocator* allocator = nullptr;

    shared_ptr<T> lock()
    {
        if (!IsValid()) return {};
        // CAS: refcount가 0이 아닐 때만 원자적으로 증가
        // → inc 후 check 후 dec 패턴의 멀티스레드 race condition 방지
        if (allocator->try_inc_refcount(ptr))
        {
            shared_ptr<T> ret;
            ret.ptr = ptr;
            ret.allocator = allocator;
            return ret;
        }
        return {};
    }
};
```

#### 1-3. SharedBlockAllocatorImpl (빈번한 객체용 풀링)

> free_list, placement new, RawStruct 구조 상세: [appendix_block_allocator.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_block_allocator.md)

```cpp
template<typename T, size_t block_size = 256>
struct SharedBlockAllocatorImpl final : public SharedBlockAllocator
{
    struct alignas(std::max(size_t(256), alignof(T))) RawStruct
    {
        uint8_t data[sizeof(T)];           // 객체 데이터
        std::atomic<uint32_t> refcount;
        std::atomic<uint32_t> refcount_weak;
    };

    struct Block { std::unique_ptr<RawStruct[]> mem; };
    std::vector<Block> blocks;
    std::vector<RawStruct*> free_list;
    vz::SpinLock locker;

    template<typename... ARG>
    inline shared_ptr<T> allocate(ARG&&... args)
    {
        locker.lock();
        if (free_list.empty())
        {
            // 새 블록 추가: 256개 연속 할당
            Block& block = blocks.emplace_back();
            block.mem.reset(new RawStruct[block_size]);
            for (size_t i = 0; i < block_size; ++i)
                free_list.push_back(block.mem.get() + i);
        }
        RawStruct* raw = free_list.back();
        free_list.pop_back();
        locker.unlock();

        new (raw) T(std::forward<ARG>(args)...);  // placement new: 생성자만 호출
        init_refcount(raw);

        shared_ptr<T> result;
        result.ptr = reinterpret_cast<T*>(raw);
        result.allocator = this;
        return result;
    }

    // dec_refcount: refcount 0 → ~T() + weak dec
    // dec_refcount_weak: weak 0 → reclaim() (free_list로 반환)
    // try_inc_refcount: CAS loop, expected==0이면 즉시 return false
};
```

#### 1-4. SharedHeapAllocator (단일 객체용)

`AllocationHandler`처럼 디바이스당 1개만 생성되는 객체에 블록 풀링을 쓰면
256개 슬롯 중 1개만 사용하고 나머지 255개가 낭비된다.
이런 객체는 개별 힙 할당 방식이 더 적합하다.

```cpp
template<typename T>
struct SharedHeapAllocator final : public SharedBlockAllocator
{
    struct RawStruct { uint8_t data[sizeof(T)]; ... };

    template<typename... ARG>
    inline shared_ptr<T> allocate(ARG&&... args)
    {
        RawStruct* raw = new RawStruct;  // 개별 힙 할당 (풀링 없음)
        new (raw) T(std::forward<ARG>(args)...);
        // ...
    }
    void reclaim(void* ptr) { delete static_cast<RawStruct*>(ptr); }
};
```

#### 1-5. 헬퍼 함수

```cpp
// 빈번하게 생성/삭제: 블록 풀링
template<typename T, size_t block_size = 256, typename... ARG>
inline shared_ptr<T> make_shared(ARG&&... args);

// 한 번만 생성/오래 유지: 힙 직접 할당
template<typename T, typename... ARG>
inline shared_ptr<T> make_shared_single(ARG&&... args);
```

---

### 2. GBackend.h 변경

Allocator.h에 시스템을 만들었으면, 실제 GPU 리소스 타입에 연결해야 한다.

#### 2-1. internal_state 타입 변경

```cpp
// 변경 전
struct GraphicsDeviceChild
{
    std::shared_ptr<void> internal_state;
    inline bool IsValid() const { return internal_state != nullptr; }
};

// 변경 후
struct GraphicsDeviceChild
{
    vz::allocator::shared_ptr<void> internal_state;
    constexpr bool IsValid() const { return internal_state.IsValid(); }
    //                                                     ^^^^^^^^^^
    //                              ptr != nullptr && allocator != nullptr 둘 다 체크
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

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| `internal_state` 타입 | `std::shared_ptr<void>` | `vz::allocator::shared_ptr<void>` |
| `IsValid()` | `internal_state != nullptr` | `internal_state.IsValid()` |
| `shared_ptr` 크기 | 16바이트 (ptr + control_block*) | 16바이트 (ptr + allocator*) |
| GPU 객체 할당 방식 | 힙 직접 할당 (`new`) | 블록 풀 (`free_list.pop()`) |
| `allocationhandler` 타입 (헤더) | `std::shared_ptr<AllocationHandler>` | `vz::allocator::shared_ptr<AllocationHandler>` |
| 내부 DX12 구조체 `allocationhandler` (×8) | `std::shared_ptr<AllocationHandler>` | `vz::allocator::shared_ptr<AllocationHandler>` |
| DX12 객체 생성 | `std::make_shared<T_DX12>()` | `vz::allocator::make_shared<T_DX12>()` |
| `AllocationHandler` 생성 | `std::make_shared<AllocationHandler>()` | `vz::allocator::make_shared_single<AllocationHandler>()` |
| CommandList 컨테이너 | `vector<unique_ptr<CommandList_DX12>>` | `BlockAllocator<CommandList_DX12, 64>` + `vector<CommandList_DX12*>` |
| CommandList 정리 | 자동 (unique_ptr 소멸) | 수동 (`cmd_allocator.free(p)` 루프) |
| placement new (`GPUBuffer`, `Texture` 등) | ❌ 금지 | ✅ 허용 |

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
    if (allocator->try_inc_refcount(ptr))  // 원자적: 0이 아니면 증가, 아니면 아무것도 안 함
    {
        shared_ptr<T> ret;
        ret.ptr = ptr;
        ret.allocator = allocator;
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
