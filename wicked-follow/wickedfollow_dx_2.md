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

(나머지 커밋 분석 예정)

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
