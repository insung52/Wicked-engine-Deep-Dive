# Wicked Engine DX12 변경사항 분석 (2025년 9월~2026년 1월)

## 개요
- **분석 기간**: 2025년 9월 1일 ~ 2026년 1월 28일
- **대상 커밋**: DX12 관련 파일 수정 또는 공통 그래픽스 인터페이스 변경
- **목적**: VizMotive Engine 적용 검토
- **분석 대상**: 25개 (DX12 + Graphics 인터페이스)

---

## 커밋 목록 (시간순)

### 분석 대상 - DX12/Graphics 인터페이스 (25개)

| # | 날짜 | 커밋 | 제목 | 카테고리 |
|---|------|------|------|----------|
| 1 | 2025-10-22 | `0bdd7a2a` | Fix out of bounds crash in SRV and UAV subresource vectors | 버그수정 |
| 2 | 2025-10-26 | `f807d9f6` | added ability to choose gpu by vendor preference | 기능 |
| 3 | 2025-11-02 | `a6adfc12` | block allocator should respect alignment of items | 버그수정 |
| 4 | 2025-11-14 | `359ade8d` | optimizations for hair particle system | 최적화 |
| 5 | 2025-11-17 | `acf8006e` | compile optimizations | 최적화 |
| 6 | 2025-11-17 | `ccaaf209` | get rid of some undefined behaviour | 안정성 |
| 7 | 2025-11-20 | `93278bea` | removed virtual tables from gfx objects, struct size reductions | 최적화 |
| 8 | 2025-11-22 | `7a5ea2f6` | pooled shared ptr | 최적화 |
| 9 | 2025-11-23 | `e1dc87e4` | prevent calling destructor when original refcount was 0 | 버그수정 |
| 10 | 2025-11-23 | `16a19429` | fix typo in #1322 | 버그수정 |
| 11 | 2025-11-24 | `90e94ed4` | placement new is allowed for gfx objects | 리팩토링 |
| 12 | 2025-11-25 | `ffcc3abd` | custom shared_ptr updates | 리팩토링 |
| 13 | 2025-11-26 | `051bfb60` | handle self-assign | 버그수정 |
| 14 | 2025-11-26 | `973c2850` | made some internal structures final | 최적화 |
| 15 | 2025-11-26 | `cf6f3434` | some clang warning fixes | 빌드 |
| 16 | 2025-11-29 | `f0203679` | dx12: root descriptor can use buffer offset | 기능 |
| 17 | 2025-12-01 | `a5162e04` | added support for shader-readable swapchain | 기능 |
| 18 | 2025-12-06 | `7ecbc25a` | gfx allocators update | 업데이트 |
| 19 | 2025-12-14 | `be8c766e` | fixed critical issue with weak_ptr | 버그수정 |
| 20 | 2025-12-31 | `bf2f369b` | dx12 fix: DeleteSubresources didn't handle rtvs and dsvs | 버그수정 |
| 21 | 2026-01-11 | `3899e472` | Mac OS support | 플랫폼 |
| 22 | 2026-01-15 | `43b93085` | dx12, vulkan: using block allocator for command lists | 최적화 |
| 23 | 2026-01-17 | `5a067ed2` | console updates and fixes | 버그수정 |
| 24 | 2026-01-18 | `2e823bb2` | region copy support for CPU textures | 기능 |
| 25 | 2026-01-24 | `24824a1c` | texture upload improvements | 최적화 |

---

## 상세 분석

### #1. `0bdd7a2a` - Fix out of bounds crash in SRV and UAV subresource vectors VizMotive 적용 완료 ✅
**날짜**: 2025-10-22 | **카테고리**: 버그수정 | **우선순위**: 높음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | `GetDescriptorIndex()`에서 subresource 벡터 범위 검사 추가 |

#### 문제
- `subresources_srv[subresource]` 접근 시 범위 검사 없음
- 잘못된 인덱스 전달 시 out-of-bounds 크래시 발생 가능

#### 해결
```cpp
// 변경 전
return internal_state->subresources_srv[subresource].index;

// 변경 후
if (subresource >= (int)internal_state->subresources_srv.size())
    return -1;
return internal_state->subresources_srv[subresource].index;
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

**적용 내용** (`GraphicsDevice_DX12.cpp`):
- SRV subresource 벡터 범위 검사 추가
- UAV subresource 벡터 범위 검사 추가

---

### #2. `f807d9f6` - added ability to choose gpu by vendor preference ⏭️ 스킵
**날짜**: 2025-10-26 | **카테고리**: 기능 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h, wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | `GPUPreference` enum에 `Nvidia`, `Intel`, `AMD` 추가 |

#### 변경 내용
```cpp
// enum 확장
enum class GPUPreference
{
    Discrete,
    Integrated,
    Nvidia,    // 추가
    Intel,     // 추가
    AMD,       // 추가
};
```

#### 스킵 사유
- 멀티 GPU 시스템에서 특정 벤더 GPU 선택 기능
- VizMotive에서 현재 필요하지 않은 편의 기능
- 필요시 나중에 적용 가능

---

### #3. `a6adfc12` - block allocator should respect alignment of items ✅ 이미 적용됨
**날짜**: 2025-11-02 | **카테고리**: 버그수정 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h |
| 핵심 변경 | `BlockAllocator`에서 아이템 정렬(alignment) 보장 |

#### 변경 내용
```cpp
// 변경 후 - T의 정렬 요구사항 보장
struct Block
{
    struct alignas(alignof(T)) RawStruct
    {
        uint8_t data[sizeof(T)];
    };
    wi::vector<RawStruct> mem;
};
```

#### VizMotive 상태
- `EngineCore/Utils/Allocator.h:26-32`에 동일한 구조로 이미 구현됨

---

### #4. `359ade8d` - optimizations for hair particle system ✅ 이미 적용됨
**날짜**: 2025-11-14 | **카테고리**: 최적화 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h (+ hair particle 관련 파일들) |
| 핵심 변경 | Allocator에서 move semantics 최적화, const 정확성 |

#### DX12/Graphics 관련 변경
```cpp
// Move semantics 최적화
internal_state = std::move(other.internal_state);

// const 정확성
inline bool is_empty() const
```

#### VizMotive 상태
- Move 생성자/대입 연산자에 `std::move` 이미 사용
- `is_empty() const` 이미 적용됨

---

### #5. `acf8006e` - compile optimizations ⏭️ 스킵
**날짜**: 2025-11-17 | **카테고리**: 최적화 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | 빌드 설정, wiGraphicsDevice.h/DX12.h/Vulkan.h |
| 핵심 변경 | 예외 제거, RTTI 비활성화, `GetTag()` 함수 추가 |

#### Graphics 관련 변경
```cpp
// 디버깅용 식별자 함수 추가
virtual const char* GetTag() const { return ""; }
const char* GetTag() const override { return "[DX12]"; }
```

#### 스킵 사유
- 빌드 설정 변경은 VizMotive 프로젝트 구조가 다름
- `GetTag()`는 디버깅용 편의 기능
- 필요시 나중에 적용 가능

---

### #6. `ccaaf209` - get rid of some undefined behaviour VizMotive 적용 완료 ✅
**날짜**: 2025-11-17 | **카테고리**: 안정성 | **우선순위**: 높음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp, wiGraphicsDevice_Vulkan.cpp |
| 핵심 변경 | `to_internal()` 함수의 undefined behavior 수정 |

#### 문제
- `CopyTexture()`에서 `Texture*`를 `GPUBuffer*`로 C-style 캐스팅
- 타입이 다른 포인터 캐스팅은 undefined behavior

```cpp
// 변경 전 - undefined behavior!
auto src_internal = to_internal((const GPUBuffer*)src);  // src는 Texture*
auto dst_internal = to_internal((const GPUBuffer*)dst);  // dst는 Texture*
```

#### WickedEngine 해결 방식
템플릿 기반 `to_internal`으로 전체 리팩토링:
```cpp
template<typename T> struct DX12Type;
template<> struct DX12Type<Texture> { using type = Texture_DX12; };
// ...

template<typename T>
typename DX12Type<T>::type* to_internal(const T* param)
{
    return static_cast<typename DX12Type<T>::type*>(param->internal_state.get());
}
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

**적용 내용** (`GraphicsDevice_DX12.cpp`):
- VizMotive는 이미 `to_internal(const Texture*)` 오버로드가 있으므로 잘못된 캐스팅만 제거
- `Texture_DX12`가 `Resource_DX12`를 상속받아 `.resource` 멤버 접근 가능

```cpp
// 변경 후 - 올바른 오버로드 호출
auto src_internal = to_internal(src);  // Texture_DX12* 반환
auto dst_internal = to_internal(dst);  // Texture_DX12* 반환
```

---

### #7. `93278bea` - removed virtual tables from gfx objects, struct size reductions VizMotive 적용 완료 ✅
**날짜**: 2025-11-20 | **카테고리**: 최적화 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h, wiGraphicsDevice_DX12.cpp/h, wiGraphicsDevice_Vulkan.cpp/h |
| 핵심 변경 | 구조체 크기 최적화, virtual table 제거, 타입 크기 축소 |

#### 상세 변경 내용

##### 1. BindFlag 타입 축소
```cpp
// 변경 전
enum class BindFlag { ... };  // 기본 int (4바이트)

// 변경 후
enum class BindFlag : uint8_t { ... };  // 1바이트
```

##### 2. GPUBufferDesc 구조체 최적화
```cpp
// VizMotive 현재 (패딩 발생)
struct GPUBufferDesc {
    uint64_t size = 0;           // 8바이트
    Usage usage = Usage::DEFAULT; // 4바이트 + 4바이트 패딩
    Format format = Format::UNKNOWN;
    uint32_t stride = 0;
    uint64_t alignment = 0;      // ← uint64_t
    BindFlag bind_flags;
    ResourceMiscFlag misc_flags;
};

// WickedEngine 변경 후 (패딩 최소화)
struct GPUBufferDesc {
    uint64_t size = 0;
    uint32_t stride = 0;         // ← 순서 변경
    uint32_t alignment = 0;      // ← uint32_t로 축소
    Usage usage = Usage::DEFAULT;
    Format format = Format::UNKNOWN;
    BindFlag bind_flags;         // uint8_t
    ResourceMiscFlag misc_flags;
};
```

##### 3. TextureDesc 멤버 순서 재배치
```cpp
// WickedEngine - usage, bind_flags를 format 뒤로 이동
struct TextureDesc {
    Type type;
    Format format;
    Usage usage;        // ← 순서 변경
    BindFlag bind_flags; // ← 순서 변경 (uint8_t)
    uint32_t width, height, depth, array_size, mip_levels, sample_count;
    ClearValue clear;
    Swizzle swizzle;
    ResourceMiscFlag misc_flags;
    ResourceState layout;
};
```

##### 4. GraphicsDeviceChild 기본 클래스 제거
```cpp
// VizMotive 현재 - virtual 소멸자로 인한 vtable 오버헤드
struct GraphicsDeviceChild {
    std::shared_ptr<void> internal_state;
    virtual ~GraphicsDeviceChild() = default;  // vtable 8바이트
};
struct Sampler : public GraphicsDeviceChild { ... };

// WickedEngine 변경 후 - 각 구조체가 직접 멤버 보유
struct Sampler {
    std::shared_ptr<void> internal_state;
    inline bool IsValid() const { return internal_state != nullptr; }
    SamplerDesc desc;
};
```

##### 5. final 키워드 및 동적 할당 금지
```cpp
struct GPUBuffer final : public GPUResource {
    // 동적 할당 금지 - vtable 없이 스택/정적 할당만 허용
    static void* operator new(size_t) = delete;
    static void operator delete(void*) = delete;
    // ... placement new도 금지
};
```

##### 6. GetMinOffsetAlignment 반환 타입 변경
```cpp
// 변경 전
uint64_t GetMinOffsetAlignment(const GPUBufferDesc* desc) const;

// 변경 후
uint32_t GetMinOffsetAlignment(const GPUBufferDesc* desc) const;
```

##### 7. CopyResource 함수 타입 체크 추가
```cpp
// WickedEngine - Texture인지 명시적 체크
if (pDst->IsTexture() && pSrc->IsTexture()) {
    // texture copy logic
} else {
    // buffer copy logic
}
```

#### 예상 메모리 절약
| 구조체 | 절약량 (대략) |
|--------|--------------|
| BindFlag enum | 3바이트/사용처 |
| GPUBufferDesc | 4~8바이트 (alignment 타입 + 패딩) |
| TextureDesc | 패딩 감소 |
| GraphicsDeviceChild 상속 객체 | 8바이트/객체 (vtable 포인터) |

#### VizMotive 적용 고려사항
| 항목 | 난이도 | 영향 범위 |
|------|--------|----------|
| BindFlag : uint8_t | 낮음 | GBackend.h만 수정 |
| GPUBufferDesc.alignment uint32_t | 중간 | GBackend.h + DX12.h |
| 멤버 순서 재배치 | 낮음 | GBackend.h |
| GraphicsDeviceChild 제거 | 높음 | 전체 Graphics 객체 영향 |
| final + operator delete | 중간 | 동적 할당 코드 확인 필요 |

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

**적용 내용**:

| 파일 | 변경 내용 |
|------|----------|
| GBackend.h | BindFlag : uint8_t 타입 변경 |
| GBackend.h | GPUBufferDesc 멤버 순서 재배치 + alignment uint32_t |
| GBackend.h | TextureDesc 멤버 순서 재배치 |
| GBackend.h | GPUResource 멤버 재배치 + sparse_page_size uint32_t |
| GBackend.h | GraphicsDeviceChild 제거 - 8개 구조체에 직접 internal_state/IsValid() 추가 |
| GBackend.h | GPUBuffer, Texture, RaytracingAccelerationStructure에 final + operator delete |
| GBackendDevice.h | GetMinOffsetAlignment 반환 타입 uint32_t |
| GraphicsDevice_DX12.h | GetMinOffsetAlignment 구현 수정 |

**참고**: CopyResource 함수 타입 체크는 VizMotive에서 별도 구현 불필요 (이미 #6에서 CopyTexture 수정됨)

---

### #8. `7a5ea2f6` - pooled shared ptr VizMotive 적용 완료 ✅
**날짜**: 2025-11-22 | **카테고리**: 최적화 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h, wiGraphics.h, wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | `std::shared_ptr` 대신 pooled block allocator 기반 custom shared_ptr 사용 |

#### 목적
- Graphics 객체의 `internal_state`가 빈번하게 생성/삭제됨
- `std::shared_ptr`는 매번 개별 힙 할당 → 메모리 단편화, 할당 오버헤드
- Block allocator를 사용하면 같은 타입 객체들이 연속 메모리에 풀링됨

#### WickedEngine 구현 구조

##### 1. SharedBlockAllocator 인터페이스
```cpp
struct SharedBlockAllocator
{
    virtual void init_refcount(void* ptr) = 0;
    virtual uint32_t get_refcount(void* ptr) = 0;
    virtual uint32_t inc_refcount(void* ptr) = 0;
    virtual uint32_t dec_refcount(void* ptr) = 0;
    virtual uint32_t get_refcount_weak(void* ptr) = 0;
    virtual uint32_t inc_refcount_weak(void* ptr) = 0;
    virtual uint32_t dec_refcount_weak(void* ptr) = 0;
};
```

##### 2. Custom shared_ptr 구조 (원본)
```cpp
template<typename T>
struct shared_ptr
{
    // handle = (ptr | allocator_id) - 하위 8비트에 allocator ID 인코딩
    uint64_t handle = 0;

    constexpr T* get_ptr() const { return (T*)(handle & (~0ull << 8ull)); }
    constexpr SharedBlockAllocator* get_allocator() const
    {
        return block_allocators[handle & 0xFF];
    }
};
```

##### 3. SharedBlockAllocatorImpl
```cpp
template<typename T, size_t block_size = 256>
struct SharedBlockAllocatorImpl final : public SharedBlockAllocator
{
    const uint8_t allocator_id = register_shared_block_allocator(this);

    // 256바이트 정렬된 블록 (하위 8비트가 0이 되도록)
    struct alignas(256) RawStruct
    {
        uint8_t data[sizeof(T)];
        std::atomic<uint32_t> refcount;
        std::atomic<uint32_t> refcount_weak;
    };

    std::vector<Block> blocks;
    std::vector<RawStruct*> free_list;
    SpinLock locker;
};
```

##### 4. make_shared 헬퍼
```cpp
template<typename T, size_t block_size = 256, typename... ARG>
inline shared_ptr<T> make_shared(ARG&&... args)
{
    return shared_block_allocator<T, block_size>->allocate(std::forward<ARG>(args)...);
}
```

#### VizMotive 적용 시 발생한 문제: DLL 경계

##### 문제 상황
1. `block_allocators[]` 배열이 `inline` 전역 변수로 선언됨
2. C++17에서 `inline` 변수는 각 DLL/EXE에서 별도로 인스턴스화됨
3. DX12 DLL에서 allocator 등록 → Engine DLL에서 lookup 실패

```
[DX12 DLL]                          [Engine DLL]
block_allocators[0] = allocator_A   block_allocators[0] = nullptr
block_allocators[1] = allocator_B   block_allocators[1] = nullptr
                  ↑                              ↑
                등록됨                         비어있음!
```

##### 증상
- `make_shared()`는 DX12 DLL에서 정상 작동
- Engine DLL에서 `get_allocator()` 호출 → nullptr 반환
- `IsValid()` 체크 후에도 크래시 발생

##### VizMotive 해결 방식
allocator pointer를 shared_ptr에 직접 저장:

```cpp
// VizMotive 수정 버전
template<typename T>
struct shared_ptr
{
    T* ptr = nullptr;
    SharedBlockAllocator* allocator = nullptr;  // 직접 저장

    constexpr bool IsValid() const
    {
        return ptr != nullptr && allocator != nullptr;
    }
    constexpr T* get_ptr() const { return ptr; }
    constexpr SharedBlockAllocator* get_allocator() const { return allocator; }
};

template<typename T>
struct weak_ptr
{
    T* ptr = nullptr;
    SharedBlockAllocator* allocator = nullptr;  // 직접 저장
    // ...
};
```

##### 트레이드오프
| 항목 | WickedEngine (handle) | VizMotive (ptr+allocator) |
|------|----------------------|---------------------------|
| 구조체 크기 | 8바이트 | 16바이트 |
| DLL 경계 | ❌ 문제 발생 | ✅ 정상 작동 |
| 복잡도 | 비트 연산 필요 | 직관적 |

- WickedEngine은 단일 실행파일이므로 DLL 경계 문제 없음
- VizMotive는 DLL 기반 아키텍처이므로 포인터 직접 저장 방식 채택

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

**수정 파일**:

| 파일 | 변경 내용 |
|------|----------|
| Allocator.h | SharedBlockAllocator 인터페이스 추가 |
| Allocator.h | shared_ptr<T> 템플릿 추가 (ptr + allocator 저장 방식) |
| Allocator.h | weak_ptr<T> 템플릿 추가 (ptr + allocator 저장 방식) |
| Allocator.h | SharedBlockAllocatorImpl<T> 구현 추가 |
| Allocator.h | make_shared<T> 헬퍼 함수 추가 |
| GBackend.h | `std::shared_ptr<void> internal_state` → `vz::allocator::shared_ptr<void> internal_state` |
| GBackend.h | `IsValid()` → `constexpr bool IsValid() const { return internal_state.IsValid(); }` |
| GraphicsDevice_DX12.cpp | `std::make_shared<*_DX12>()` → `vz::allocator::make_shared<*_DX12>()` |
| GraphicsDevice_DX12.cpp | `rootsig_desc_lifetime_extender` 타입 변경 |

---

### #9. `e1dc87e4` - prevent calling destructor when original refcount was 0 VizMotive 적용 완료 ✅
**날짜**: 2025-11-23 | **카테고리**: 버그수정 | **우선순위**: 높음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h |
| 핵심 변경 | `weak_ptr::lock()`에서 이중 소멸자 호출 방지 |

#### 문제 상황
`weak_ptr::lock()` 시나리오:
1. `inc_refcount()` 호출하여 refcount 증가
2. 원래 refcount가 0이었다면 (객체 이미 파괴됨)
3. `dec_refcount()`로 refcount 되돌림
4. `dec_refcount()`가 1→0 되면 소멸자 호출
5. **이미 파괴된 객체의 소멸자 재호출 → undefined behavior!**

```cpp
// 문제 코드
shared_ptr<T> lock()
{
    uint32_t old_strong = allocator->inc_refcount(ptr);
    if (old_strong == 0)
    {
        allocator->dec_refcount(ptr);  // ← 소멸자 다시 호출됨!
        return {};
    }
    // ...
}
```

#### 해결
`dec_refcount`에 `destruct_on_zero` 파라미터 추가:

```cpp
virtual uint32_t dec_refcount(void* ptr, bool destruct_on_zero = true) = 0;

// weak_ptr::lock()에서는 false 전달
allocator->dec_refcount(ptr, false);  // undo refcount, don't destruct

// SharedBlockAllocatorImpl 구현
uint32_t dec_refcount(void* ptr, bool destruct_on_zero) override
{
    uint32_t old = refcount.fetch_sub(1);
    if (old == 1)
    {
        if (destruct_on_zero) static_cast<T*>(ptr)->~T();  // 조건부 소멸
        dec_refcount_weak(ptr);
    }
    return old;
}
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| Allocator.h | `dec_refcount(void* ptr, bool destruct_on_zero = true)` 시그니처 변경 |
| Allocator.h | `shared_ptr::reset()`에서 `dec_refcount(ptr, true)` 호출 |
| Allocator.h | `weak_ptr::lock()`에서 `dec_refcount(ptr, false)` 호출 |
| Allocator.h | `SharedBlockAllocatorImpl::dec_refcount` 구현 수정 |

---

### #10. `16a19429` - fix typo in #1322 VizMotive 적용 완료 ✅
**날짜**: 2025-11-23 | **카테고리**: 버그수정 | **우선순위**: 높음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h |
| 핵심 변경 | `destruct_on_zero` 기본값 수정 |

#### 변경 내용
```cpp
// #9에서 잘못된 기본값
virtual uint32_t dec_refcount(void* ptr, bool destruct_on_zero = false) = 0;

// #10에서 수정 - 대부분의 경우 파괴해야 하므로 true가 기본값
virtual uint32_t dec_refcount(void* ptr, bool destruct_on_zero = true) = 0;
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

#9와 함께 올바른 기본값 `true`로 적용 완료.

---

### #11. `90e94ed4` - placement new is allowed for gfx objects VizMotive 적용 완료 ✅
**날짜**: 2025-11-24 | **카테고리**: 리팩토링 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h |
| 핵심 변경 | GPUBuffer, Texture, RaytracingAccelerationStructure에서 placement new 허용 |

#### 변경 내용
커밋 #7에서 동적 할당을 금지했으나, block allocator 등에서 placement new가 필요함:

```cpp
// 변경 전 (#7) - placement new도 금지
static void* operator new(size_t, void*) = delete;
static void* operator new[](size_t, void*) = delete;

// 변경 후 (#11) - placement new 허용
static void* operator new(size_t, void* p) noexcept { return p; }
static void* operator new[](size_t, void* p) noexcept { return p; }
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| GBackend.h | GPUBuffer - placement new 허용 |
| GBackend.h | Texture - placement new 허용 |
| GBackend.h | RaytracingAccelerationStructure - placement new 허용 |

---

### #12. `ffcc3abd` - custom shared_ptr updates VizMotive 부분 적용 ✅
**날짜**: 2025-11-25 | **카테고리**: 리팩토링 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h, wiGraphicsDevice_DX12.cpp/h 외 다수 |
| 핵심 변경 | 대규모 리팩토링 - 이름 변경 + 새 기능 추가 |

#### WickedEngine 변경 내용

##### 1. 이름 변경
```cpp
// 인터페이스
SharedBlockAllocator → SharedAllocator

// 구현체
SharedBlockAllocatorImpl → SharedBlockAllocator

// 전역 배열
block_allocators[] → shared_allocators[]
```

##### 2. 새로운 SharedHeapAllocator 추가
```cpp
// 단일 객체용 heap 할당자 (block allocator와 달리 풀링 없음)
template<typename T>
struct SharedHeapAllocator final : public SharedAllocator
{
    // new/delete로 개별 할당/해제
    void reclaim(void* ptr) { delete static_cast<RawStruct*>(ptr); }
};
```

##### 3. make_shared_single 함수 추가
```cpp
// 단일 객체용 - heap 기반
template<typename T, typename... ARG>
inline shared_ptr<T> make_shared_single(ARG&&... args);

// 비교: 기존 make_shared는 block allocator 기반 (풀링)
```

##### 4. DX12 변경
- `AllocationHandler` 생성: `std::make_shared` → `make_shared_single`
- 이유: `AllocationHandler`는 GraphicsDevice당 1개만 존재 → 풀링 불필요

#### VizMotive 적용 내용
WickedEngine의 이름 변경은 스킵 (VizMotive는 DLL 경계 대응으로 다른 구조 사용)

**적용한 부분**:
- `SharedHeapAllocator<T>` 추가
- `make_shared_single<T>()` 함수 추가
- `AllocationHandler`에 `make_shared_single` 적용

#### VizMotive 부분 적용 ✅
**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| Allocator.h | `SharedHeapAllocator<T>` 템플릿 추가 |
| Allocator.h | `make_shared_single<T>()` 함수 추가 |
| GraphicsDevice_DX12.h | `allocationhandler` 타입 → `vz::allocator::shared_ptr` |
| GraphicsDevice_DX12.cpp | 8개 internal struct의 `allocationhandler` 타입 변경 |
| GraphicsDevice_DX12.cpp | `make_shared_single<AllocationHandler>()` 사용 |

**스킵한 부분**:
- 이름 변경 (SharedBlockAllocator → SharedAllocator 등)
- VizMotive는 DLL 경계 대응을 위해 다른 구조 사용 중

---

### #13. `051bfb60` - handle self-assign ✅ 이미 적용됨
**날짜**: 2025-11-26 | **카테고리**: 버그수정 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h, wiFunction.h |
| 핵심 변경 | `PageAllocator::Allocation::operator=`에 self-assign 체크 추가 |

#### 변경 내용
```cpp
Allocation& operator=(const Allocation& other)
{
    if (&other == this) return *this;  // 추가
    Reset();
    // ...
}
```

#### VizMotive 상태
- `Allocator.h:161`에 이미 동일한 self-assign 체크 존재
- 추가 작업 불필요

---

### #14. `973c2850` - made some internal structures final VizMotive 적용 완료 ✅
**날짜**: 2025-11-26 | **카테고리**: 최적화 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp 외 |
| 핵심 변경 | internal struct에 `final` 키워드 추가 |

#### 변경 내용 (DX12 관련)
```cpp
// 변경 전
struct Texture_DX12 : public Resource_DX12
struct BVH_DX12 : public Resource_DX12

// 변경 후
struct Texture_DX12 final : public Resource_DX12
struct BVH_DX12 final : public Resource_DX12
```

#### 목적
- `final` 키워드로 컴파일러가 virtual function 호출을 devirtualize 가능
- 미세한 성능 최적화

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | `Texture_DX12 final` |
| GraphicsDevice_DX12.cpp | `BVH_DX12 final` |

---

### #15. `cf6f3434` - some clang warning fixes VizMotive 적용 완료 ✅
**날짜**: 2025-11-26 | **카테고리**: 빌드 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp 외 |
| 핵심 변경 | clang 컴파일러 경고 수정 |

#### DX12 변경 내용
```cpp
// switch문에 default case 추가 (clang 경고 방지)
switch (format)
{
    // ... cases ...
    case DXGI_FORMAT_BC7_UNORM_SRGB:
        return Format::BC7_UNORM_SRGB;
    default:  // 추가
        break;
}
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | `_ConvertFormat` switch문에 `default: break;` 추가 |

---

### #16. `f0203679` - dx12: root descriptor can use buffer offset VizMotive 적용 완료 ✅
**날짜**: 2025-11-29 | **카테고리**: 기능 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | root descriptor 바인딩 시 buffer offset 지원 |

#### 변경 내용
root descriptor는 DX12에서 buffer만 사용 가능하며, subresource의 offset을 GPU 주소에 더해줌:

```cpp
// SingleDescriptor에 buffer_offset 멤버 추가
struct SingleDescriptor
{
    // ...
    uint64_t buffer_offset = 0;
};

// SRV/UAV 바인딩 시 offset 적용
if (subresource >= 0)
{
    address += internal_state->subresources_srv[subresource].buffer_offset;
}

// CreateSubresource에서 offset 저장
descriptor.buffer_offset = offset;
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | `SingleDescriptor`에 `buffer_offset` 멤버 추가 |
| GraphicsDevice_DX12.cpp | SRV root descriptor 바인딩에 offset 적용 |
| GraphicsDevice_DX12.cpp | UAV root descriptor 바인딩에 offset 적용 |
| GraphicsDevice_DX12.cpp | SRV subresource 생성 시 `buffer_offset` 설정 |

---

### #17. `a5162e04` - added support for shader-readable swapchain VizMotive 적용 완료 ✅
**날짜**: 2025-12-01 | **카테고리**: 기능 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | SwapChain을 쉐이더에서 읽을 수 있도록 SRV 지원 추가 |

#### 변경 내용

##### 1. SwapChain_DX12 구조체 변경
```cpp
// 변경 전
std::vector<ComPtr<ID3D12Resource>> backBuffers;
std::vector<D3D12_CPU_DESCRIPTOR_HANDLE> backbufferRTV;
~SwapChain_DX12() {
    for (auto& x : backbufferRTV) allocationhandler->descriptors_rtv.free(x);
}

// 변경 후
std::vector<shared_ptr<Texture_DX12>> textures;
// 소멸자 제거 - Texture_DX12가 자체 정리 담당
```

##### 2. CreateSwapChain 변경
```cpp
// DXGI_USAGE_SHADER_INPUT 플래그 추가 (쉐이더 읽기 가능)
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT | DXGI_USAGE_SHADER_INPUT;

// 각 back buffer를 완전한 Texture_DX12 객체로 생성
for (uint32_t i = 0; i < internal_state->GetBufferCount(); ++i)
{
    auto texture_internal = make_shared<Texture_DX12>();
    texture_internal->allocationhandler = allocationhandler;
    internal_state->swapChain->GetBuffer(i, PPV_ARGS(texture_internal->resource));

    // RTV 생성
    texture_internal->rtv.handle = allocationhandler->descriptors_rtv.allocate();
    device->CreateRenderTargetView(texture_internal->resource.Get(), &rtv_desc, texture_internal->rtv.handle);

    // SRV 생성 (새로 추가)
    D3D12_SHADER_RESOURCE_VIEW_DESC srv_desc = {};
    srv_desc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
    srv_desc.Format = ...;
    srv_desc.Texture2D.MipLevels = 1;
    srv_desc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    texture_internal->srv.init(this, srv_desc, texture_internal->resource.Get());

    internal_state->textures.push_back(texture_internal);
}
```

##### 3. GetBackBuffer 변경
```cpp
// 변경 전 - 매번 새 객체 생성
Texture GraphicsDevice_DX12::GetBackBuffer(const SwapChain* swapchain)
{
    Texture result;
    result.internal_state = make_shared<Texture_DX12>();
    // 매번 새로 설정...
}

// 변경 후 - 미리 생성된 텍스처 반환
Texture GraphicsDevice_DX12::GetBackBuffer(const SwapChain* swapchain)
{
    Texture result;
    uint32_t index = internal_state->GetBufferIndex();
    result.internal_state = internal_state->textures[index];
    // desc 정보 설정
}
```

##### 4. RenderPassBegin/Present 변경
```cpp
// 모든 backBuffers[index] 참조를 textures[index]->resource로 변경
barrier.Transition.pResource = internal_state->textures[index]->resource.Get();

// 모든 backbufferRTV[index] 참조를 textures[index]->rtv.handle로 변경
RTV.cpuDescriptor = internal_state->textures[index]->rtv.handle;
```

#### 효과
- SwapChain back buffer를 쉐이더에서 SRV로 읽기 가능
- 포스트 프로세싱에서 back buffer를 입력 텍스처로 사용 가능
- 각 back buffer가 완전한 Texture_DX12 객체로 관리되어 일관된 리소스 처리

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | SwapChain_DX12: backBuffers + backbufferRTV → textures 통합 |
| GraphicsDevice_DX12.cpp | SwapChain_DX12 소멸자 제거 |
| GraphicsDevice_DX12.cpp | CreateSwapChain: DXGI_USAGE_SHADER_INPUT 추가 |
| GraphicsDevice_DX12.cpp | CreateSwapChain: SRV descriptor 생성 추가 |
| GraphicsDevice_DX12.cpp | GetBackBuffer: 미리 생성된 textures[index] 반환 |
| GraphicsDevice_DX12.cpp | RenderPassBegin: textures[index]->resource/rtv.handle 사용 |

---

### #18. `7ecbc25a` - gfx allocators update VizMotive 적용 완료 ✅
**날짜**: 2025-12-06 | **카테고리**: 업데이트 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | D3D12MemAlloc.h, D3D12MemAlloc.cpp, wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | D3D12MemAlloc 라이브러리 메이저 버전 업데이트 + Windows 11 최적화 플래그 |

#### 버전 변경
| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| D3D12MemAlloc | 2.1.0-development (2023-07-05) | 3.0.1 (2025-05-08) |

#### DX12 코드 변경
```cpp
// Windows 11에서 작은 committed buffer가 성능 문제를 일으킴
allocatorDesc.Flags |= D3D12MA::ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED;
```

#### 3.0.1 주요 개선점
- Windows 11 작은 버퍼 성능 문제 해결 플래그 추가
- 2년치 버그 수정 및 최적화
- 새로운 API 및 기능 호환성
- GPU upload heap 지원 개선
- Residency priority 지원

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| D3D12MemAlloc.h | 버전 2.1.0 → 3.0.1 전체 교체 |
| D3D12MemAlloc.cpp | 버전 2.1.0 → 3.0.1 전체 교체 |
| GraphicsDevice_DX12.cpp | `ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED` 플래그 추가 |

---

### #19. `be8c766e` - fixed critical issue with weak_ptr VizMotive 적용 완료 ✅
**날짜**: 2025-12-14 | **카테고리**: 버그수정 | **우선순위**: 높음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiAllocator.h |
| 핵심 변경 | weak_ptr::lock()의 race condition 완전 해결 |

#### 문제 상황 (커밋 #9, #10의 불완전한 해결책)
```
Thread A: inc_refcount() → 0 반환 (객체 이미 파괴됨)
Thread B: 객체가 재사용됨 (메모리 풀에서 다른 용도로 할당)
Thread A: dec_refcount(ptr, false) → 다른 객체의 refcount 감소!
```

기존 방식 (inc → check → dec)은 원자적이지 않아 race condition 발생 가능.

#### 해결책: try_inc_refcount()
```cpp
bool try_inc_refcount(void* ptr) override
{
    auto& ref = static_cast<RawStruct*>(ptr)->refcount;
    uint32_t expected = ref.load(std::memory_order_acquire);
    do {
        if (expected == 0) {
            return false;  // refcount가 0이면 실패
        }
    } while (!ref.compare_exchange_weak(expected, expected + 1,
        std::memory_order_acq_rel, std::memory_order_acquire));
    return true;  // 원자적으로 증가 성공
}
```

`compare_exchange_weak`로 refcount가 0이 아닐 때만 원자적으로 증가.

#### 변경 내용

##### 1. SharedBlockAllocator 인터페이스
```cpp
// destruct_on_zero 파라미터 제거 (더 이상 필요 없음)
virtual uint32_t dec_refcount(void* ptr) = 0;

// 새 함수 추가
virtual bool try_inc_refcount(void* ptr) = 0;
```

##### 2. weak_ptr::lock() 단순화
```cpp
// 변경 전 - race condition 있음
uint32_t old_strong = allocator->inc_refcount(ptr);
if (old_strong == 0) {
    allocator->dec_refcount(ptr, false);  // 위험!
    return {};
}

// 변경 후 - 원자적 동작
if (allocator->try_inc_refcount(ptr)) {
    shared_ptr<T> ret;
    ret.ptr = ptr;
    ret.allocator = allocator;
    return ret;
}
return {};
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| Allocator.h | `SharedBlockAllocator::try_inc_refcount()` 인터페이스 추가 |
| Allocator.h | `dec_refcount()`에서 `destruct_on_zero` 파라미터 제거 |
| Allocator.h | `weak_ptr::lock()`에서 `try_inc_refcount()` 사용 |
| Allocator.h | `SharedBlockAllocatorImpl::try_inc_refcount()` 구현 |
| Allocator.h | `SharedHeapAllocator::try_inc_refcount()` 구현 |

---

### #20. `bf2f369b` - dx12 fix: DeleteSubresources didn't handle rtvs and dsvs VizMotive 적용 완료 ✅
**날짜**: 2025-12-31 | **카테고리**: 버그수정 | **우선순위**: 높음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | Texture_DX12를 Resource_DX12로 병합하여 DeleteSubresources() 버그 수정 |

#### 문제 상황
```cpp
// 기존 구조
struct Resource_DX12 {
    void destroy_subresources() {
        // srv, uav만 정리 - rtv, dsv는 정리 안됨!
    }
};
struct Texture_DX12 : Resource_DX12 {
    SingleDescriptor rtv, dsv;  // 여기에 있어서 destroy_subresources()가 접근 못함
};
```

`DeleteSubresources()` 호출 시 RTV/DSV descriptor가 정리되지 않아 leak 발생.

#### 해결책: 클래스 병합
```cpp
struct Resource_DX12 {
    // 기존 멤버들...
    SingleDescriptor srv, uav;

    // Texture_DX12에서 이동
    SingleDescriptor rtv = {};
    SingleDescriptor dsv = {};
    std::vector<SingleDescriptor> subresources_rtv;
    std::vector<SingleDescriptor> subresources_dsv;
    std::vector<SubresourceData> mapped_subresources;

    void destroy_subresources() {
        srv.destroy();
        uav.destroy();
        rtv.destroy();      // 이제 정리됨
        dsv.destroy();      // 이제 정리됨
        // subresources도 모두 정리...
    }

    ~Resource_DX12() { /* virtual 제거 */ }
};
// Texture_DX12 클래스 삭제
```

#### 추가 변경
- `to_internal(const Texture*)` 반환 타입: `Texture_DX12*` → `Resource_DX12*`
- `SwapChain_DX12::textures`: `vector<shared_ptr<Texture_DX12>>` → `vector<shared_ptr<Resource_DX12>>`
- 모든 `make_shared<Texture_DX12>()` → `make_shared<Resource_DX12>()`

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | `Texture_DX12` 멤버들을 `Resource_DX12`로 이동 |
| GraphicsDevice_DX12.cpp | `destroy_subresources()`에서 RTV/DSV 정리 추가 |
| GraphicsDevice_DX12.cpp | `Texture_DX12` 클래스 삭제 |
| GraphicsDevice_DX12.cpp | 모든 `Texture_DX12` 참조를 `Resource_DX12`로 변경 |

---

### #21. `3899e472` - Mac OS support VizMotive 부분 적용 ✅
**날짜**: 2026-01-11 | **카테고리**: 플랫폼 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | 253개 파일 (158,000+ 줄 추가) |
| 핵심 변경 | Mac OS + Metal API 지원 |

#### DX12 관련 변경 (VizMotive 적용 대상)

##### 1. Fence 단순화 ✅ 적용
```cpp
// 변경 전 - 두 개의 fence 배열
ComPtr<ID3D12Fence> frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];
ComPtr<ID3D12Fence> frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];
// Signal(0) = in use, Signal(1) = free (cpu)
// Signal(FRAMECOUNT) (gpu)

// 변경 후 - 단일 fence + 값 추적
uint64_t frame_fence_values[BUFFERCOUNT] = {};
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];
// monotonically increasing value per buffer
```

**개선점:**
- 코드 단순화 (fence 2개 → 1개)
- 명확한 동기화 로직 (monotonic 값 사용)

##### 2. WriteTopLevelAccelerationStructureInstance nullptr 체크 ✅ 적용
```cpp
// 변경 전
tmp.AccelerationStructure = to_internal(instance->bottom_level)->gpu_address;

// 변경 후
D3D12_RAYTRACING_INSTANCE_DESC tmp = {};
if (instance != nullptr)
{
    auto internal_state = to_internal(instance->bottom_level);
    tmp.AccelerationStructure = internal_state->gpu_address;
    // ...
}
```

##### 3. 함수명 변경 ⏭️ 스킵
- `AlignTo` → `align`, `IsAligned` → `is_aligned`
- 스타일 차이일 뿐, VizMotive는 기존 이름 유지

#### VizMotive 부분 적용 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.h | `frame_fence_cpu/gpu` → `frame_fence` + `frame_fence_values` |
| GraphicsDevice_DX12.cpp | Fence 초기화 단순화 |
| GraphicsDevice_DX12.cpp | SubmitCommandLists fence signal/wait 로직 수정 |
| GraphicsDevice_DX12.cpp | WaitForGPU fence wait 로직 수정 |
| GraphicsDevice_DX12.cpp | `WriteTopLevelAccelerationStructureInstance` nullptr 체크 |

**스킵:**
- Mac OS / Metal API 관련 전체
- `AlignTo` → `align` 리네임

---

### #22. `43b93085` - dx12, vulkan: using block allocator for command lists VizMotive 적용 완료 ✅
**날짜**: 2026-01-15 | **카테고리**: 최적화 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphicsDevice_DX12.cpp, wiGraphicsDevice_DX12.h |
| 핵심 변경 | CommandList 메모리 풀링으로 할당 오버헤드 감소 |

#### 변경 내용
```cpp
// 변경 전 - 개별 힙 할당
std::vector<std::unique_ptr<CommandList_DX12>> commandlists;
commandlists.push_back(std::make_unique<CommandList_DX12>());
cmd.internal_state = commandlists[cmd_current].get();

// 변경 후 - 블록 할당자 풀링
BlockAllocator<CommandList_DX12, 64> cmd_allocator;
std::vector<CommandList_DX12*> commandlists;
commandlists.push_back(cmd_allocator.allocate());
cmd.internal_state = commandlists[cmd_current];
```

#### 효과
- 64개 단위로 메모리 블록 할당하여 단편화 방지
- CommandList 생성/삭제 시 힙 할당 오버헤드 감소
- 캐시 지역성 향상

#### VizMotive 추가 수정
WickedEngine 원본에는 없지만, VizMotive에서 소멸자에 정리 코드 추가 필요:
```cpp
GraphicsDevice_DX12::~GraphicsDevice_DX12()
{
    WaitForGPU();

    // Free command lists allocated by block allocator
    for (auto* cmdlist : commandlists)
    {
        cmd_allocator.free(cmdlist);
    }
    commandlists.clear();
    // ...
}
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.h | `cmd_allocator` + `commandlists` raw pointer 벡터로 변경 |
| GraphicsDevice_DX12.cpp | `cmd_allocator.allocate()` 사용 |
| GraphicsDevice_DX12.cpp | `.get()` 제거 |
| GraphicsDevice_DX12.cpp | 소멸자에 command list 정리 코드 추가 |

---

### #23. `5a067ed2` - console updates and fixes ⏭️ 스킵
**날짜**: 2026-01-17 | **카테고리**: 버그수정 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h, wiGraphicsDevice_DX12.cpp/h 외 |
| 핵심 변경 | 콘솔(Xbox) 관련 수정 + capability 정리 |

#### 주요 변경 내용

##### 1. `RENDERTARGET_AND_VIEWPORT_ARRAYINDEX_WITHOUT_GS` capability 제거
```cpp
// 이 기능: VS/MS에서 SV_RenderTargetArrayIndex 직접 설정 가능 여부
// 기존: 미지원 GPU를 위해 GS 에뮬레이션 코드 유지
// 변경: 현대 DX12 GPU는 모두 지원하므로 에뮬레이션 코드 삭제
```

WickedEngine은 다음도 함께 삭제:
- `GSTYPE_SHADOW_EMULATION`
- `GSTYPE_ENVMAP_EMULATION`
- `GSTYPE_SHADOW_TRANSPARENT_EMULATION`
- `GSTYPE_SHADOW_ALPHATEST_EMULATION`

##### 2. Enum 플래그 번호 재정렬
제거된 플래그로 인해 `GraphicsDeviceCapability` enum 번호 변경

##### 3. 콘솔(Xbox) 전용 변경
- Xbox용 HEVC GUID 추가
- Xbox swapchain 생성 시 `ApplyTextureCreationFlags` 호출
- Xbox Present/RenderPass 코드 수정

#### 스킵 사유
| 이유 | 설명 |
|------|------|
| 콘솔 전용 | 대부분 Xbox 관련 수정 |
| 호환성 유지 | VizMotive는 GS 에뮬레이션 코드 유지하여 구형 GPU 지원 |
| 기능적 이점 없음 | 코드 정리일 뿐, 성능/기능 개선 없음 |
| 리스크 | enum 재정렬로 인한 호환성 문제 가능 |

---

### #24. `2e823bb2` - region copy support for CPU textures VizMotive 적용 완료 ✅
**날짜**: 2026-01-18 | **카테고리**: 기능 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h, wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | CPU 텍스처(UPLOAD/READBACK) 복사 시 footprint 기반 위치 지정 지원 |

#### 배경
DX12에서 텍스처 복사는 두 가지 방식:
1. **GPU 텍스처 (DEFAULT)**: subresource index로 위치 지정
2. **CPU 텍스처 (UPLOAD/READBACK)**: 실제로는 버퍼이므로 footprint로 위치 지정

기존 코드는 모든 경우에 subresource index만 사용하여 CPU 텍스처 복사 시 문제 발생.

#### 변경 내용

##### 1. 헬퍼 함수 추가 (wiGraphics.h / GBackend.h)
```cpp
// Plane slice 계산 (depth/stencil 분리)
constexpr uint32_t GetPlaneSlice(ImageAspect aspect)
{
    switch (aspect)
    {
    case ImageAspect::COLOR:
    case ImageAspect::DEPTH:
    case ImageAspect::LUMINANCE:
        return 0;
    case ImageAspect::STENCIL:
    case ImageAspect::CHROMINANCE:
        return 1;
    default:
        break;
    }
    return 0;
}

// Subresource index 계산
constexpr uint32_t ComputeSubresource(uint32_t mip, uint32_t slice, uint32_t plane,
    uint32_t mip_count, uint32_t array_size)
{
    return mip + slice * mip_count + plane * mip_count * array_size;
}
```

##### 2. CopyTexture 함수 수정 (wiGraphicsDevice_DX12.cpp)
```cpp
void GraphicsDevice_DX12::CopyTexture(...)
{
    const TextureDesc& src_desc = src->GetDesc();
    const TextureDesc& dst_desc = dst->GetDesc();

    const UINT srcPlane = GetPlaneSlice(src_aspect);
    const UINT dstPlane = GetPlaneSlice(dst_aspect);
    const UINT src_subresource = D3D12CalcSubresource(...);
    const UINT dst_subresource = D3D12CalcSubresource(...);

    CD3DX12_TEXTURE_COPY_LOCATION src_location;
    CD3DX12_TEXTURE_COPY_LOCATION dst_location;

    if (src_desc.usage == Usage::UPLOAD && dst_desc.usage == Usage::DEFAULT)
    {
        // CPU (buffer) -> GPU (texture)
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_res, src_internal->footprints[src_subresource]);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_res, dst_subresource);
    }
    else if (src_desc.usage == Usage::DEFAULT && dst_desc.usage == Usage::READBACK)
    {
        // GPU (texture) -> CPU (buffer)
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_res, src_subresource);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_res, dst_internal->footprints[dst_subresource]);
    }
    else if (src_desc.usage == Usage::DEFAULT && dst_desc.usage == Usage::DEFAULT)
    {
        // GPU (texture) -> GPU (texture)
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_res, src_subresource);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_res, dst_subresource);
    }
    else
    {
        // CPU (buffer) -> CPU (buffer)
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_res, src_internal->footprints[src_subresource]);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_res, dst_internal->footprints[dst_subresource]);
    }
    // CopyTextureRegion 호출...
}
```

#### 4가지 복사 시나리오
| 시나리오 | src | dst | src 위치 | dst 위치 |
|----------|-----|-----|----------|----------|
| 업로드 | UPLOAD | DEFAULT | footprint | subresource |
| 리드백 | DEFAULT | READBACK | subresource | footprint |
| GPU→GPU | DEFAULT | DEFAULT | subresource | subresource |
| CPU→CPU | UPLOAD/READBACK | UPLOAD/READBACK | footprint | footprint |

#### 추가 개선: GetPlaneSlice 사용
기존 코드의 단순한 plane 계산:
```cpp
// 변경 전 - STENCIL만 처리
UINT srcPlane = src_aspect == ImageAspect::STENCIL ? 1 : 0;

// 변경 후 - 더 많은 aspect 처리
const UINT srcPlane = GetPlaneSlice(src_aspect);
```

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GBackend.h | `GetPlaneSlice()` 헬퍼 함수 추가 |
| GBackend.h | `ComputeSubresource()` 헬퍼 함수 추가 |
| GraphicsDevice_DX12.cpp | `CopyTexture()` 4가지 usage 시나리오 처리 |
| GraphicsDevice_DX12.cpp | footprint 기반 copy location 지원 |

---

### #25. `24824a1c` - texture upload improvements VizMotive 적용 완료 ✅
**날짜**: 2026-01-24 | **카테고리**: 최적화 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h, wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | 텍스처 헬퍼 함수 추가 + CreateTexture 코드 정리 |

#### 변경 내용

##### 1. 새 헬퍼 함수 추가 (wiGraphics.h / GBackend.h)
```cpp
// TextureDesc에서 mip count 계산 (mip_levels=0이면 최대값 반환)
constexpr uint32_t GetMipCount(const TextureDesc& desc)
{
    return desc.mip_levels == 0 ? GetMipCount(desc.width, desc.height, desc.depth) : desc.mip_levels;
}

// 총 subresource 개수 계산
constexpr uint32_t GetTextureSubresourceCount(const TextureDesc& desc)
{
    const uint32_t mips = GetMipCount(desc);
    return desc.array_size * mips;
}
```

##### 2. CreateTextureSubresourceDatas() alignment 파라미터 (WickedEngine만)
```cpp
// GPU별 row pitch 정렬을 위한 alignment 파라미터 추가
inline void CreateTextureSubresourceDatas(const TextureDesc& desc, void* data_ptr,
    wi::vector<SubresourceData>& subresource_datas, uint32_t alignment = 1)
{
    // ...
    subresource_data.row_pitch = align((uint32_t)num_blocks_x * bytes_per_block, alignment);
    // ...
}
```
VizMotive에는 이 함수가 없으므로 스킵.

##### 3. DX12 CreateTexture 코드 정리
```cpp
// 변경 전 - mip_levels 계산이 늦게 수행됨
resourcedesc.MipLevels = desc->mip_levels;  // 원본 값 사용
// ... 중간 코드 ...
if (texture->desc.mip_levels == 0) {
    texture->desc.mip_levels = GetMipCount(width, height, depth);
}
internal_state->footprints.resize(desc->array_size * std::max(1u, desc->mip_levels));

// 변경 후 - mip_levels 먼저 계산
texture->desc.mip_levels = GetMipCount(texture->desc);  // 먼저 계산
resourcedesc.MipLevels = texture->desc.mip_levels;      // 계산된 값 사용
// ...
internal_state->footprints.resize(GetTextureSubresourceCount(texture->desc));
```

#### 개선점
- 코드 중복 제거 (mip_levels 계산 로직)
- 일관성 향상 (모든 곳에서 동일한 헬퍼 함수 사용)
- 가독성 향상

#### VizMotive 적용 완료 ✅
**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GBackend.h | `GetMipCount(const TextureDesc&)` 오버로드 추가 |
| GBackend.h | `GetTextureSubresourceCount(const TextureDesc&)` 추가 |
| GBackend.h | `ComputeTextureMemorySizeInBytes()`에서 `GetMipCount(desc)` 사용 |
| GraphicsDevice_DX12.cpp | mip_levels 계산을 resource 생성 전으로 이동 |
| GraphicsDevice_DX12.cpp | `footprints.resize()`에서 `GetTextureSubresourceCount()` 사용 |

---

## 분석 완료 요약

### 전체 커밋 현황 (25개)

| 상태 | 개수 | 커밋 번호 |
|------|------|-----------|
| ✅ 적용 완료 | 18개 | #1, #6, #7, #8, #9, #10, #11, #12, #14, #15, #16, #17, #18, #19, #20, #21, #22, #24, #25 |
| ✅ 이미 적용됨 | 3개 | #3, #4, #13 |
| ⏭️ 스킵 | 4개 | #2, #5, #23 |

### 주요 적용 내용

#### 버그 수정
- SRV/UAV 벡터 범위 검사 (#1)
- CopyTexture undefined behavior 수정 (#6)
- weak_ptr 이중 소멸자 호출 방지 (#9, #10)
- weak_ptr race condition 해결 (#19)
- DeleteSubresources RTV/DSV 정리 (#20)

#### 최적화
- vtable 제거, 구조체 크기 최적화 (#7)
- pooled shared_ptr (#8, #12)
- CommandList BlockAllocator 풀링 (#22)
- D3D12MemAlloc 3.0.1 업데이트 (#18)

#### 기능 추가
- root descriptor buffer offset 지원 (#16)
- shader-readable swapchain (#17)
- CPU 텍스처 region copy (#24)
- 텍스처 헬퍼 함수 (#25)

#### 코드 품질
- placement new 허용 (#11)
- internal struct final 키워드 (#14)
- clang 경고 수정 (#15)
- Fence 단순화 (#21)

---

## VizMotive 적용 우선순위

(분석 후 정리 예정)

---

## 적용 완료 커밋

| # | 커밋 | 내용 | 적용일 |
|---|------|------|--------|
| 1 | `0bdd7a2a` | SRV/UAV 벡터 범위 검사 | 2026-01-28 |
| 6 | `ccaaf209` | CopyTexture undefined behavior 수정 | 2026-01-28 |
| 7 | `93278bea` | vtable 제거, 구조체 크기 최적화 | 2026-01-28 |
| 8 | `7a5ea2f6` | pooled shared_ptr (DLL 경계 대응 수정) | 2026-01-28 |
| 9 | `e1dc87e4` | weak_ptr::lock() 이중 소멸자 호출 방지 | 2026-01-28 |
| 10 | `16a19429` | destruct_on_zero 기본값 수정 | 2026-01-28 |
| 11 | `90e94ed4` | placement new 허용 | 2026-01-28 |
| 12 | `ffcc3abd` | SharedHeapAllocator + make_shared_single (부분 적용) | 2026-01-28 |
| 14 | `973c2850` | Texture_DX12, BVH_DX12에 final 추가 | 2026-01-28 |
| 15 | `cf6f3434` | clang 경고 수정 (default case) | 2026-01-28 |
| 16 | `f0203679` | root descriptor buffer offset 지원 | 2026-01-28 |
| 17 | `a5162e04` | shader-readable swapchain (SRV 지원) | 2026-01-29 |
| 18 | `7ecbc25a` | D3D12MemAlloc 3.0.1 업데이트 + Win11 최적화 | 2026-01-29 |
| 19 | `be8c766e` | weak_ptr race condition 해결 (try_inc_refcount) | 2026-01-29 |
| 20 | `bf2f369b` | DeleteSubresources RTV/DSV 정리 (Texture_DX12 병합) | 2026-01-29 |
| 21 | `3899e472` | Fence 단순화 + RT nullptr 체크 (부분 적용) | 2026-01-29 |
| 22 | `43b93085` | CommandList BlockAllocator 풀링 | 2026-01-29 |
| 24 | `2e823bb2` | CPU 텍스처 region copy (footprint 지원) | 2026-01-29 |
| 25 | `24824a1c` | 텍스처 헬퍼 함수 + CreateTexture 정리 | 2026-01-29 |
