# VizMotive 적용: 구조체 메모리 최적화

---

## 개요

Graphics 객체 구조체의 불필요한 메모리 낭비를 제거한다.

- **변경 1**: `BindFlag` enum 크기 축소 (4바이트 → 1바이트)
- **변경 2**: `GPUBufferDesc` 멤버 순서 재배치 + `alignment` 타입 축소
- **변경 3**: `GraphicsDeviceChild` 기반 클래스 제거 (vtable 8바이트/객체 제거) ← 핵심
- **변경 4**: DX12 내부 구조체에 `final` 키워드 추가

---

## 이 브랜치의 시작 상태

이 작업은 `Topic-allocator-shared-ptr` 브랜치를 기반으로 한다.

해당 브랜치에서 `std::shared_ptr<void>` → `vz::allocator::shared_ptr<void>` 교체가 이미 완료됐다.
변경 3(GraphicsDeviceChild 제거)이 `vz::allocator::shared_ptr` 없이는 어렵기 때문에 이 순서가 중요하다.

```cpp
// Topic-allocator-shared-ptr 브랜치 현재 상태:
struct GraphicsDeviceChild
{
    vz::allocator::shared_ptr<void> internal_state;  // ← 이미 교체됨
    constexpr bool IsValid() const { return internal_state.IsValid(); }

    virtual ~GraphicsDeviceChild() = default;  // ← 아직 존재, vtable 원인
};
```

---

## 변경 1: BindFlag enum 크기 축소

**파일**: `EngineCore/GBackend/GBackend.h`

`BindFlag`는 8개 플래그를 가지며 최대값이 `1 << 7 = 128`이다. `uint8_t`(0~255)로 충분하지만
기저 타입을 명시하지 않으면 C++이 기본으로 `int`(4바이트)를 사용한다.

```cpp
// 변경 전
enum class BindFlag
{
    NONE = 0,
    VERTEX_BUFFER   = 1 << 0,
    INDEX_BUFFER    = 1 << 1,
    CONSTANT_BUFFER = 1 << 2,
    SHADER_RESOURCE = 1 << 3,
    RENDER_TARGET   = 1 << 4,
    DEPTH_STENCIL   = 1 << 5,
    UNORDERED_ACCESS = 1 << 6,
    SHADING_RATE    = 1 << 7,
};

// 변경 후
enum class BindFlag : uint8_t
{
    NONE = 0,
    VERTEX_BUFFER   = 1 << 0,
    INDEX_BUFFER    = 1 << 1,
    CONSTANT_BUFFER = 1 << 2,
    SHADER_RESOURCE = 1 << 3,
    RENDER_TARGET   = 1 << 4,
    DEPTH_STENCIL   = 1 << 5,
    UNORDERED_ACCESS = 1 << 6,
    SHADING_RATE    = 1 << 7,
};
```

절약: 3바이트 (`BindFlag`를 멤버로 가진 구조체마다).

---

## 변경 2: GPUBufferDesc 최적화

**파일**: `EngineCore/GBackend/GBackend.h`, `EngineCore/GBackend/GBackendDevice.h`

### 2-1. GPUBufferDesc 멤버 순서 재배치 + alignment 타입 축소

구조체 패딩은 각 멤버가 자신의 크기에 맞는 주소(alignment)에 위치해야 하는 규칙 때문에 생긴다.
큰 타입 먼저, 작은 타입 나중에 배치하면 패딩을 최소화할 수 있다.

```cpp
// 변경 전 (패딩 발생)
struct GPUBufferDesc
{
    uint64_t size = 0;                      // 8바이트, offset 0
    Usage usage = Usage::DEFAULT;           // 1바이트 (uint8_t enum)
    Format format = Format::UNKNOWN;        // 1바이트 (uint8_t enum)
    // 패딩 2바이트
    uint32_t stride = 0;                    // 4바이트, offset 4 (ok)
    uint64_t alignment = 0;                 // 8바이트 — 너무 큼!
    BindFlag bind_flags = BindFlag::NONE;   // 4바이트 (int, 변경 전)
    ResourceMiscFlag misc_flags = ResourceMiscFlag::NONE;
};
```

```cpp
// 변경 후 (패딩 최소화)
struct GPUBufferDesc
{
    uint64_t size = 0;                      // 8바이트
    uint32_t stride = 0;                    // 4바이트
    uint32_t alignment = 0;                 // 4바이트 (uint64_t → uint32_t, 정렬값은 항상 작음)
    Usage usage = Usage::DEFAULT;           // 1바이트
    Format format = Format::UNKNOWN;        // 1바이트
    BindFlag bind_flags = BindFlag::NONE;   // 1바이트 (변경 1 적용)
    ResourceMiscFlag misc_flags = ResourceMiscFlag::NONE; // (크기 확인 후 배치)
    // 최소 패딩
};
```

> `ResourceMiscFlag`는 `1 << 10`까지 있어 `uint8_t`에 들어가지 않는다. 기저 타입 명시 없이 그대로 두거나 `uint16_t`로 지정할 수 있다.

### 2-2. GetMinOffsetAlignment 반환형 변경

`alignment` 필드가 `uint32_t`로 바뀌었으므로 이를 반환하는 함수도 맞춰야 한다.

**파일**: `EngineCore/GBackend/GBackendDevice.h`

```cpp
// 변경 전
virtual uint64_t GetMinOffsetAlignment(const GPUBufferDesc* desc) const = 0;

// 변경 후
virtual uint32_t GetMinOffsetAlignment(const GPUBufferDesc* desc) const = 0;
```

`GetMinOffsetAlignment`를 구현하는 `GraphicsDevice_DX12.cpp`의 오버라이드 함수도 동일하게 반환형 변경 필요.

---

## 변경 3: GraphicsDeviceChild 기반 클래스 제거

**파일**: `EngineCore/GBackend/GBackend.h`

### 왜 제거하는가?

`virtual ~GraphicsDeviceChild()` 한 줄이 있으면 C++이 모든 자식 객체에 **vptr(vtable 포인터) 8바이트**를 숨겨서 추가한다.
`GraphicsDeviceChild`를 상속하는 객체가 10개이고 씬에 수천 개 있다면 수백 KB가 낭비된다.

`vz::allocator::shared_ptr`이 이미 적용된 이 브랜치에서는 `GraphicsDeviceChild`를 완전히 제거하고
`internal_state`를 각 구조체에 직접 넣는 것이 가능하다.

### 현재 상속 구조

```
GraphicsDeviceChild  (internal_state, IsValid(), virtual ~)
├── Sampler
├── Shader
├── GPUResource      (type, mapped_data, mapped_size, sparse_page_size)
│   ├── GPUBuffer
│   └── Texture
├── VideoDecoder
├── GPUQueryHeap
├── PipelineState
├── SwapChain
└── RaytracingPipelineState
```

### 변경 후 구조

```
Sampler              (internal_state, IsValid() 직접 보유)
Shader               (internal_state, IsValid() 직접 보유)
GPUResource          (internal_state, IsValid() 직접 보유, type, mapped_data...)
├── GPUBuffer final  (final + operator delete 추가)
└── Texture final    (final + operator delete 추가)
VideoDecoder         (internal_state, IsValid() 직접 보유)
GPUQueryHeap         (internal_state, IsValid() 직접 보유)
PipelineState        (internal_state, IsValid() 직접 보유)
SwapChain            (internal_state, IsValid() 직접 보유)
RaytracingPipelineState (internal_state, IsValid() 직접 보유)
```

### 코드 변경

**GraphicsDeviceChild 삭제**, 각 구조체에 `internal_state`와 `IsValid()`를 직접 추가한다.

```cpp
// 변경 전
struct GraphicsDeviceChild
{
    vz::allocator::shared_ptr<void> internal_state;
    constexpr bool IsValid() const { return internal_state.IsValid(); }
    virtual ~GraphicsDeviceChild() = default;
};

struct Sampler : public GraphicsDeviceChild
{
    SamplerDesc desc;
    const SamplerDesc& GetDesc() const { return desc; }
};

struct GPUResource : public GraphicsDeviceChild
{
    enum class Type : uint8_t { ... } type = Type::UNKNOWN_TYPE;
    void* mapped_data = nullptr;
    size_t mapped_size = 0;
    size_t sparse_page_size = 0;
};

struct GPUBuffer : public GPUResource
{
    GPUBufferDesc desc;
    constexpr const GPUBufferDesc& GetDesc() const { return desc; }
};

struct Texture : public GPUResource
{
    TextureDesc desc;
    // ...
};
```

```cpp
// 변경 후 (GraphicsDeviceChild 삭제)

struct Sampler
{
    vz::allocator::shared_ptr<void> internal_state;
    constexpr bool IsValid() const { return internal_state.IsValid(); }

    SamplerDesc desc;
    const SamplerDesc& GetDesc() const { return desc; }
};

struct Shader
{
    vz::allocator::shared_ptr<void> internal_state;
    constexpr bool IsValid() const { return internal_state.IsValid(); }

    ShaderStage stage = ShaderStage::Count;
};

struct GPUResource
{
    vz::allocator::shared_ptr<void> internal_state;
    constexpr bool IsValid() const { return internal_state.IsValid(); }

    enum class Type : uint8_t { ... } type = Type::UNKNOWN_TYPE;
    void* mapped_data = nullptr;
    size_t mapped_size = 0;
    size_t sparse_page_size = 0;
    // (IsTexture, IsBuffer, IsAccelerationStructure 함수는 그대로 유지)
};

struct GPUBuffer final : public GPUResource   // final 추가!
{
    // 힙 할당 금지 (vtable 없으니 base* 통한 delete가 UB)
    static void* operator new(size_t) = delete;
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;
    // placement new는 허용 (BlockAllocator 등 직접 메모리 관리용)
    static void* operator new(size_t, void* p) noexcept { return p; }
    static void* operator new[](size_t, void* p) noexcept { return p; }

    GPUBufferDesc desc;
    constexpr const GPUBufferDesc& GetDesc() const { return desc; }
};

struct Texture final : public GPUResource    // final 추가!
{
    static void* operator new(size_t) = delete;
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;
    static void* operator new(size_t, void* p) noexcept { return p; }
    static void* operator new[](size_t, void* p) noexcept { return p; }

    TextureDesc desc;
    // ...
};

// VideoDecoder, GPUQueryHeap, PipelineState, SwapChain, RaytracingPipelineState도 동일하게:
struct VideoDecoder
{
    vz::allocator::shared_ptr<void> internal_state;
    constexpr bool IsValid() const { return internal_state.IsValid(); }

    VideoDesc desc;
    // ...
};

// (나머지도 동일 패턴)
```

### operator delete를 왜 막는가?

```
virtual ~가 없으므로:
    base* ptr = new GPUBuffer();
    delete ptr;  // UB! base의 소멸자만 호출됨

→ 애초에 new/delete를 금지해서 이 실수를 컴파일 에러로 막음.

GPUBuffer는 보통:
    GPUBuffer buf;           // 스택 변수
    GPUBuffer buf = {};      // 멤버 변수
처럼 사용하므로 힙 할당 자체가 필요 없음.
```

### final을 왜 추가하는가?

`GPUBuffer`와 `Texture`는 더 이상 상속할 계획이 없는 leaf 클래스다.
`final`을 붙이면 컴파일러가 이 사실을 알고 가상 함수 호출을 **직접 호출(devirtualization)**로 최적화할 수 있다.

> `GPUResource`에는 `final`을 붙이지 않는다. `GPUBuffer`와 `Texture`가 상속하고 있기 때문.

---

## 변경 4: DX12 내부 구조체 final 추가

**파일**: `GraphicsBackends/GraphicsDevice_DX12.cpp`

DX12 구현 내부의 `Texture_DX12`, `BVH_DX12`는 더 이상 파생 클래스가 없는 내부 구현 클래스다.
`final`을 추가해 컴파일러가 devirtualization을 수행할 수 있게 한다.

```cpp
// 변경 전
struct Texture_DX12 : public Resource_DX12
{
    // ...
};

struct BVH_DX12 : public Resource_DX12
{
    // ...
};

// 변경 후
struct Texture_DX12 final : public Resource_DX12
{
    // ...
};

struct BVH_DX12 final : public Resource_DX12
{
    // ...
};
```

---

## 전체 효과 요약

| 변경 | 파일 | 효과 |
|------|------|------|
| `BindFlag : uint8_t` | GBackend.h | 3바이트/사용처 절약 |
| `GPUBufferDesc` 멤버 순서 재배치 | GBackend.h | 패딩 감소 (~12바이트) |
| `alignment` uint64_t → uint32_t | GBackend.h, GBackendDevice.h | 4바이트 절약 |
| `GraphicsDeviceChild` 제거 | GBackend.h | **8바이트/객체 절약** (vtable 제거) |
| `GPUBuffer/Texture final` | GBackend.h | devirtualization 최적화 |
| `operator delete = delete` | GBackend.h | 잘못된 힙 할당 컴파일 에러로 방지 |
| `Texture_DX12/BVH_DX12 final` | GraphicsDevice_DX12.cpp | devirtualization 최적화 |

10,000개 객체 기준: vtable 제거만으로도 **~80KB** 절약.
