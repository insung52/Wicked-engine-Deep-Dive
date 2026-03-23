# VizMotive 적용: Texture 관련 개선

**[Notebooklm pdf 문서 : texture](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/pdf/texture.pdf)**

## 개요

WickedEngine의 텍스처 처리 개선(topic_texture.md)을 참고하여,
VizMotive에 필요한 변경 사항을 분석하고 적용한 기록.

> **배경 개념 참고**
> - 텍스처 전체 흐름 (파일→업로드→렌더→화면), Mipmap, Footprint, Subresource, BC 압축, Descriptor, Barrier:
>   [part5_texture_lifecycle.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part5_texture_lifecycle.md)
> - 힙 종류, 디스크립터, 리소스 상태 기초:
>   [part4_resource_management.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part4_resource_management.md)

적용 항목 4가지:

| # | 항목 | 파일 | 문제 |
|---|------|------|------|
| 1 | BC 블록 계산 수정 | GBackend.h | mip별 블록 수 계산 버그 |
| 2 | Texture_DX12 병합 | GraphicsDevice_DX12.cpp | DeleteSubresources RTV/DSV 누수 |
| 3 | Shader-readable SwapChain | GraphicsDevice_DX12.cpp | back buffer를 SRV로 쓸 수 없음 |
| 4 | 헬퍼 함수 추가 | GBackend.h / .cpp | mip 계산 중복, subresource count 없음 |

> **이미 적용된 항목 (master 기준으로 확인됨)**
> - CopyTexture UPLOAD/READBACK footprint 처리: 이미 master에 존재
> - GetMipCount(width, height, depth): 이미 master에 존재

---

## 1. BC 블록 계산 수정

> **BC 압축 개념**: [part5_texture_lifecycle.md §5.7](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part5_texture_lifecycle.md#57-bc-block-compressed-텍스처)

### 문제

`GBackend.h:1871`의 `ComputeTextureMemorySizeInBytes()`에 버그가 있다.

Block Compressed(BC) 텍스처란, 4×4 픽셀 단위로 데이터를 묶어 압축하는 포맷이다.
(BC1~BC7, DXT1~DXT5 등)

BC 텍스처의 각 mip 레벨 크기를 계산하려면, mip 레벨의 픽셀 수를 먼저 구한 뒤
그걸 4로 나눈 블록 수를 계산해야 한다.

```
예: 원본 텍스처 width = 10 pixels, block_size = 4
mip 0: 10 pixels → ceil(10/4) = 3 blocks
mip 1:  5 pixels → ceil(5/4)  = 2 blocks
mip 2:  2 pixels → ceil(2/4)  = 1 block  (1 미만은 1로 올림)
```

**현재 코드 (잘못됨):**

```cpp
// GBackend.h:1876-1884
const uint32_t num_blocks_x = desc.width / pixels_per_block;  // base mip의 블록 수
const uint32_t num_blocks_y = desc.height / pixels_per_block; //   (한 번만 계산)

for (uint32_t mip = 0; mip < mips; ++mip)
{
    const uint32_t width  = std::max(1u, num_blocks_x >> mip); // ❌ 블록 수를 shift
    const uint32_t height = std::max(1u, num_blocks_y >> mip); //    나머지 픽셀 누락!
```

문제점: base mip의 블록 수를 먼저 구하고, 그걸 mip별로 오른쪽 시프트(`>> mip`)한다.

```
예: width = 10, pixels_per_block = 4
num_blocks_x = 10 / 4 = 2   (버림! 나머지 2픽셀 무시됨)
mip 0: 2 >> 0 = 2 blocks    ← 실제로는 3 blocks여야 함
mip 1: 2 >> 1 = 1 block     ← 실제로는 2 blocks여야 함
```

나머지 픽셀이 첫 나눗셈에서 이미 사라진다.

**올바른 코드:**

각 mip별로 픽셀 수를 먼저 계산하고, 그 픽셀 수에 대해 ceiling division을 적용한다.

```cpp
for (uint32_t mip = 0; mip < mips; ++mip)
{
    const uint32_t mip_width  = std::max(1u, desc.width  >> mip); // mip별 픽셀 수
    const uint32_t mip_height = std::max(1u, desc.height >> mip);
    const uint32_t depth      = std::max(1u, desc.depth  >> mip);

    // ceiling division: 나머지 픽셀도 블록 하나로 올림
    const uint32_t width  = (mip_width  + pixels_per_block - 1) / pixels_per_block;
    const uint32_t height = (mip_height + pixels_per_block - 1) / pixels_per_block;
    size += width * height * depth * bytes_per_block;
}
```

### 수정 방법

`GBackend.h`의 `ComputeTextureMemorySizeInBytes` 함수를 수정한다.

**변경 전 (GBackend.h:1871-1891):**

```cpp
constexpr size_t ComputeTextureMemorySizeInBytes(const TextureDesc& desc)
{
    size_t size = 0;
    const uint32_t bytes_per_block = GetFormatStride(desc.format);
    const uint32_t pixels_per_block = GetFormatBlockSize(desc.format);
    const uint32_t num_blocks_x = desc.width / pixels_per_block;
    const uint32_t num_blocks_y = desc.height / pixels_per_block;
    const uint32_t mips = desc.mip_levels == 0 ? GetMipCount(desc.width, desc.height, desc.depth) : desc.mip_levels;
    for (uint32_t layer = 0; layer < desc.array_size; ++layer)
    {
        for (uint32_t mip = 0; mip < mips; ++mip)
        {
            const uint32_t width = std::max(1u, num_blocks_x >> mip);
            const uint32_t height = std::max(1u, num_blocks_y >> mip);
            const uint32_t depth = std::max(1u, desc.depth >> mip);
            size += width * height * depth * bytes_per_block;
        }
    }
    size *= desc.sample_count;
    return size;
}
```

**변경 후:**

```cpp
constexpr size_t ComputeTextureMemorySizeInBytes(const TextureDesc& desc)
{
    size_t size = 0;
    const uint32_t bytes_per_block = GetFormatStride(desc.format);
    const uint32_t pixels_per_block = GetFormatBlockSize(desc.format);
    const uint32_t mips = desc.mip_levels == 0 ? GetMipCount(desc.width, desc.height, desc.depth) : desc.mip_levels;
    for (uint32_t layer = 0; layer < desc.array_size; ++layer)
    {
        for (uint32_t mip = 0; mip < mips; ++mip)
        {
            const uint32_t mip_width  = std::max(1u, desc.width  >> mip);
            const uint32_t mip_height = std::max(1u, desc.height >> mip);
            const uint32_t depth      = std::max(1u, desc.depth  >> mip);
            const uint32_t width  = (mip_width  + pixels_per_block - 1) / pixels_per_block;
            const uint32_t height = (mip_height + pixels_per_block - 1) / pixels_per_block;
            size += width * height * depth * bytes_per_block;
        }
    }
    size *= desc.sample_count;
    return size;
}
```

`num_blocks_x`, `num_blocks_y` 두 변수를 제거하고, 루프 안에서 mip별로 픽셀→블록 수를 계산한다.

> **BC가 아닌 일반 포맷**은 `pixels_per_block = 1`이므로 ceiling division이 있어도 결과가 동일하다.
> 따라서 BC/non-BC 구분 없이 같은 공식을 쓸 수 있다.

---

## 2. DeleteSubresources RTV/DSV 누수 수정

> **SRV/RTV/DSV 개념**: [part5_texture_lifecycle.md §5.9](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part5_texture_lifecycle.md#59-descriptor뷰--같은-텍스처를-여러-방식으로-쓰기)

### 문제

`DeleteSubresources(GPUResource* resource)`는 텍스처의 모든 서브리소스 descriptor를 해제하는 함수다.
현재 이 함수를 호출해도 RTV(Render Target View), DSV(Depth Stencil View) descriptor가 누수된다.

**구조 이해:**

```
Resource_DX12 (base)
├─ srv                    ← SRV (Shader Resource View)
├─ uav                    ← UAV (Unordered Access View)
├─ subresources_srv[]
└─ subresources_uav[]

Texture_DX12 : Resource_DX12
├─ rtv                    ← RTV (Render Target View)
├─ dsv                    ← DSV (Depth Stencil View)
├─ subresources_rtv[]
└─ subresources_dsv[]
```

`destroy_subresources()`는 `Resource_DX12`에 정의되어 있고,
**non-virtual 함수**이므로 `Texture_DX12`에서 오버라이드할 수 없다.
(→ C++ 상속/virtual 개념: [07_structs_and_initialization.md #8](../../study/lan/c++/07_structs_and_initialization.md#8-상속-virtual-override-final))

```cpp
// GraphicsDevice_DX12.cpp:1340
void Resource_DX12::destroy_subresources()
{
    srv.destroy();
    uav.destroy();
    for (auto& x : subresources_srv) x.destroy();
    subresources_srv.clear();
    for (auto& x : subresources_uav) x.destroy();
    subresources_uav.clear();
    // ❌ rtv, dsv, subresources_rtv, subresources_dsv 없음
}
```

```cpp
// GraphicsDevice_DX12.cpp:5255
void GraphicsDevice_DX12::DeleteSubresources(GPUResource* resource)
{
    // ... to_internal() 으로 Resource_DX12* 얻기 ...
    internal_state->destroy_subresources();  // ← Texture_DX12의 rtv/dsv 정리 안 됨!
}
```

`Texture_DX12::~Texture_DX12()`(소멸자)는 rtv/dsv를 정리하지만,
`DeleteSubresources()`는 소멸자를 호출하지 않는다.
결과: **rtv, dsv, subresources_rtv, subresources_dsv descriptor가 descriptor heap에 계속 남는다.**

장기 실행 시 descriptor heap이 고갈될 수 있다.

### 해결 방법: Texture_DX12를 Resource_DX12에 병합

`Texture_DX12`의 멤버를 `Resource_DX12`로 올리고, `destroy_subresources()`에서 함께 정리한다.

`Texture_DX12 : Resource_DX12` 상속 구조를 제거하고 `Resource_DX12` 하나로 통합한다.

> **왜 virtual로 해결하지 않는가?**
> 기존 코드에서 이미 vtable을 제거하는 방향으로 설계되어 있다.
> `destroy_subresources()`를 virtual로 만들면 모든 Resource_DX12에 vptr이 생기고,
> 이는 설계 방향과 반대된다.
> 또한, WickedEngine도 같은 이유로 병합을 선택했다.

### 수정 방법

**1단계: Resource_DX12에 Texture_DX12 멤버 추가**

`GraphicsDevice_DX12.cpp:1321`의 `Resource_DX12` 구조체에 texture 전용 필드를 추가한다.

```cpp
struct Resource_DX12
{
    std::shared_ptr<GraphicsDevice_DX12::AllocationHandler> allocationhandler;
    ComPtr<D3D12MA::Allocation> allocation;
    ComPtr<ID3D12Resource> resource;
    SingleDescriptor srv;
    SingleDescriptor uav;
    std::vector<SingleDescriptor> subresources_srv;
    std::vector<SingleDescriptor> subresources_uav;
    SingleDescriptor uav_raw;

    // ↓ Texture 전용 필드 추가 (기존 Texture_DX12에서 이동)
    SingleDescriptor rtv = {};
    SingleDescriptor dsv = {};
    std::vector<SingleDescriptor> subresources_rtv;
    std::vector<SingleDescriptor> subresources_dsv;
    std::vector<SubresourceData> mapped_subresources;
    // ↑

    D3D12_GPU_VIRTUAL_ADDRESS gpu_address = 0;
    UINT64 total_size = 0;
    std::vector<D3D12_PLACED_SUBRESOURCE_FOOTPRINT> footprints;
    std::vector<UINT64> rowSizesInBytes;
    std::vector<UINT> numRows;
    SparseTextureProperties sparse_texture_properties;

    void destroy_subresources()
    {
        srv.destroy();
        uav.destroy();
        for (auto& x : subresources_srv) x.destroy();
        subresources_srv.clear();
        for (auto& x : subresources_uav) x.destroy();
        subresources_uav.clear();

        // ↓ 추가: Texture 서브리소스 정리
        rtv.destroy();
        dsv.destroy();
        for (auto& x : subresources_rtv) x.destroy();
        subresources_rtv.clear();
        for (auto& x : subresources_dsv) x.destroy();
        subresources_dsv.clear();
        // ↑
    }

    virtual ~Resource_DX12()
    {
        std::scoped_lock lck(allocationhandler->destroylocker);
        uint64_t framecount = allocationhandler->framecount;
        if (allocation) allocationhandler->destroyer_allocations.push_back(std::make_pair(allocation, framecount));
        if (resource) allocationhandler->destroyer_resources.push_back(std::make_pair(resource, framecount));
        destroy_subresources();
    }
};
```

**2단계: Texture_DX12 구조체 제거 (또는 빈 alias로 유지)**

```cpp
// 제거:
struct Texture_DX12 : public Resource_DX12 { ... };

// 대신: 타입 alias로 유지하면 코드 변경 최소화
using Texture_DX12 = Resource_DX12;
```

`using` alias를 사용하면 코드 전체에서 `Texture_DX12`를 그대로 사용할 수 있어서
나머지 코드를 수정할 필요가 없다.

> **주의**: 기존 `Texture_DX12::~Texture_DX12()`에서 rtv/dsv를 정리하던 코드는
> 이제 `Resource_DX12::destroy_subresources()`로 통합되었으므로 중복이 된다.
> `Texture_DX12` 소멸자는 제거한다.

---

## 3. Shader-readable SwapChain

> **SwapChain / DXGI_USAGE_SHADER_INPUT 개념**: [part5_texture_lifecycle.md §5.11](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part5_texture_lifecycle.md#511-swapchain--화면에-표시되는-특별한-텍스처)

### 문제

현재 back buffer는 렌더 타겟(RTV)으로만 사용할 수 있고,
쉐이더에서 입력으로 읽을 수 없다(SRV 없음).

```
현재:
BackBuffer[0] ─── RTV 만 있음 ─── 렌더 타겟으로 렌더링 가능
                                   ❌ 쉐이더에서 sampler로 읽을 수 없음

변경 후:
BackBuffer[0] ─── RTV + SRV ─── 렌더 타겟으로 렌더링 가능
                                  ✅ 쉐이더에서 읽기 가능 (후처리, 화면 캡처 등)
```

**SwapChain 생성 시 현재 코드 (GraphicsDevice_DX12.cpp:3178):**

```cpp
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;  // RTV만
```

`DXGI_USAGE_SHADER_INPUT` 플래그가 없으면 DXGI가 back buffer를 shader-readable로 만들지 않는다.

또한 현재 `SwapChain_DX12` 구조체에 SRV descriptor가 없다.

### 수정 방법

**1단계: SwapChain_DX12에 SRV 추가**

`GraphicsDevice_DX12.cpp:1494`의 `SwapChain_DX12` 구조체:

```cpp
struct SwapChain_DX12
{
    std::shared_ptr<GraphicsDevice_DX12::AllocationHandler> allocationhandler;
#ifdef PLATFORM_XBOX
    uint32_t bufferIndex = 0;
#else
    ComPtr<IDXGISwapChain3> swapChain;
#endif
    std::vector<ComPtr<ID3D12Resource>> backBuffers;
    std::vector<D3D12_CPU_DESCRIPTOR_HANDLE> backbufferRTV;
    std::vector<D3D12_CPU_DESCRIPTOR_HANDLE> backbufferSRV;  // ← 추가

    Texture dummyTexture;
    ColorSpace colorSpace = ColorSpace::SRGB;

    ~SwapChain_DX12()
    {
        std::scoped_lock lck(allocationhandler->destroylocker);
        uint64_t framecount = allocationhandler->framecount;
        for (auto& x : backBuffers)
            allocationhandler->destroyer_resources.push_back(std::make_pair(x, framecount));
        for (auto& x : backbufferRTV)
            allocationhandler->descriptors_rtv.free(x);
        for (auto& x : backbufferSRV)              // ← 추가: SRV 해제
            allocationhandler->descriptors_res.free(x);
    }
    // ...
};
```

**2단계: CreateSwapChain에서 SHADER_INPUT 플래그 추가 + SRV 생성**

```cpp
// 변경 전 (GraphicsDevice_DX12.cpp:3178)
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;

// 변경 후
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT | DXGI_USAGE_SHADER_INPUT;
```

back buffer 별 SRV 생성 (PC 경로, RTV 생성 바로 뒤에 추가):

```cpp
internal_state->backbufferSRV.resize(desc->buffer_count);

// ... (기존 RTV 생성 루프 내부)
for (uint32_t i = 0; i < desc->buffer_count; ++i)
{
    hr = internal_state->swapChain->GetBuffer(i, PPV_ARGS(internal_state->backBuffers[i]));
    assert(SUCCEEDED(hr));

    // RTV 생성 (기존)
    internal_state->backbufferRTV[i] = allocationhandler->descriptors_rtv.allocate();
    device->CreateRenderTargetView(internal_state->backBuffers[i].Get(), &rtvDesc, internal_state->backbufferRTV[i]);

    // SRV 생성 (추가)
    D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
    srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    srvDesc.Format = _ConvertFormat(desc->format);
    srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
    srvDesc.Texture2D.MipLevels = 1;
    internal_state->backbufferSRV[i] = allocationhandler->descriptors_res.allocate();
    device->CreateShaderResourceView(internal_state->backBuffers[i].Get(), &srvDesc, internal_state->backbufferSRV[i]);
}
```

resize 시 (기존 buffer 해제 코드 블록, `3101`번째 줄 근처)에도 SRV 해제를 추가한다:

```cpp
if (!internal_state->backbufferRTV.empty())
{
    WaitForGPU();
    internal_state->backBuffers.clear();
    for (auto& x : internal_state->backbufferRTV)
        allocationhandler->descriptors_rtv.free(x);
    internal_state->backbufferRTV.clear();

    // ↓ 추가
    for (auto& x : internal_state->backbufferSRV)
        allocationhandler->descriptors_res.free(x);
    internal_state->backbufferSRV.clear();
    // ↑
}
```

**3단계: GetBackBuffer()에서 SRV 연결**

`GraphicsDevice_DX12.cpp:5914`의 `GetBackBuffer()`:

```cpp
Texture GraphicsDevice_DX12::GetBackBuffer(const SwapChain* swapchain) const
{
    auto swapchain_internal = to_internal(swapchain);

    auto internal_state = std::make_shared<Texture_DX12>();
    internal_state->allocationhandler = allocationhandler;
    internal_state->resource = swapchain_internal->backBuffers[swapchain_internal->GetBufferIndex()];

    // ↓ 추가: SRV CPU descriptor 연결
    internal_state->srv.init(
        this,
        swapchain_internal->backbufferSRV[swapchain_internal->GetBufferIndex()],
        D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV
    );
    // ↑

    // ... (기존 footprint 계산 코드 유지)
```

> **XBOX 경로는 별도:** XBOX에서는 `DXGI_USAGE` 플래그가 다르고
> back buffer를 직접 생성하므로 XBOX 경로는 현재 그대로 둔다.

---

## 4. 헬퍼 함수 추가

> **Mipmap 개념**: [part5_texture_lifecycle.md §5.3](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part5_texture_lifecycle.md#53-mipmap--거리별-해상도-최적화)
> **Subresource Index 개념**: [part5_texture_lifecycle.md §5.6](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part5_texture_lifecycle.md#56-subresource--default-heap-텍스처의-특정-mipsliceplane-지정)

### 문제

**문제 1: GetMipCount(const TextureDesc&) wrapper 없음**

현재 `GetMipCount(width, height, depth)` 함수는 있지만,
`TextureDesc`를 받는 wrapper가 없어서 호출할 때마다 직접 꺼내 전달해야 한다:

```cpp
// 현재 방식: 매번 이렇게 써야 함
uint32_t mips = GetMipCount(desc.width, desc.height, desc.depth);

// 원하는 방식:
uint32_t mips = GetMipCount(desc);
```

또한 `mip_levels == 0`이면 "자동 최대값"을 의미하는 규칙이 있는데,
이것도 매번 직접 처리해야 한다.

**문제 2: GetTextureSubresourceCount 없음**

텍스처의 전체 서브리소스 개수(mip × array_size)를 구하는 함수가 없다.

서브리소스 인덱스는 `mip + (slice × mip_levels) + (plane × mip_levels × array_size)` 로 계산되는데,
총 개수 = `mip_levels × array_size` (단일 plane 기준) 를 직접 계산해야 한다.

**문제 3: CreateTexture에서 mip_levels 계산이 resource 생성 이후**

현재 CreateTexture 내부:

```cpp
// GraphicsDevice_DX12.cpp:3562
resourcedesc.MipLevels = desc->mip_levels;  // 0이면 DX12가 자동 계산
// ... resource 생성 ...
// GraphicsDevice_DX12.cpp:3639 (나중에)
if (texture->desc.mip_levels == 0)
{
    texture->desc.mip_levels = GetMipCount(...);  // 뒤늦게 계산
}
```

`mip_levels == 0`이면 DX12가 resource 생성 시 실제 mip 수를 결정하고,
이후에 `GetMipCount()`로 재계산한다.
GetTextureSubresourceCount를 추가하면 이 계산을 resource 생성 전으로 이동시키는 게 깔끔하다.

### 수정 방법

**1단계: GetMipCount(const TextureDesc&) 추가 (GBackend.h)**

기존 `GetMipCount(width, height, depth)` 함수 뒤에 추가:

```cpp
// GBackend.h (ComputeTextureMemorySizeInBytes 위쪽)
constexpr uint32_t GetMipCount(const TextureDesc& desc)
{
    if (desc.mip_levels > 0)
        return desc.mip_levels;
    return GetMipCount(desc.width, desc.height, desc.depth);
}
```

`mip_levels == 0`이면 자동 최대값을 계산하고, 아니면 그대로 반환한다.

**2단계: GetTextureSubresourceCount 추가 (GBackend.h)**

```cpp
constexpr uint32_t GetTextureSubresourceCount(const TextureDesc& desc)
{
    return GetMipCount(desc) * desc.array_size;
}
```

**3단계: CreateTexture에서 mip_levels 사전 계산**

`GraphicsDevice_DX12.cpp:3562` 부근에서 resource 생성 전에 mip_levels를 확정한다:

```cpp
// ↓ 추가: resource 생성 전에 mip_levels 확정
if (desc->mip_levels == 0)
{
    texture->desc.mip_levels = GetMipCount(*desc);
}
// ↑

resourcedesc.MipLevels = texture->desc.mip_levels;  // 이제 항상 실제 값
```

이후 `3639`번째 줄의 뒤늦은 재계산 코드는 제거한다:

```cpp
// 제거:
if (texture->desc.mip_levels == 0)
{
    texture->desc.mip_levels = GetMipCount(texture->desc.width, texture->desc.height, texture->desc.depth);
}
```

---

## 변경 전후 비교

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| BC mip 블록 계산 | base 블록 수 → shift (나머지 누락) | mip별 픽셀 수 → ceiling division |
| DeleteSubresources | RTV/DSV 누수 | rtv/dsv/subresources_rtv/dsv 모두 정리 |
| Texture_DX12 구조 | Resource_DX12 상속 | Resource_DX12로 병합 (alias 유지) |
| SwapChain 사용성 | RTV only | RTV + SRV (shader-readable) |
| GetMipCount | (width, height, depth)만 존재 | TextureDesc wrapper 추가 |
| 서브리소스 개수 | 직접 계산 필요 | GetTextureSubresourceCount() |
| CreateTexture mip 계산 | resource 생성 후 재계산 | resource 생성 전 사전 확정 |

---

## 보충 설명

> 아래 개념들의 더 상세한 설명 (업로드 전체 흐름 코드, Mipmap 메모리 계산, Barrier 등):
> [part5_texture_lifecycle.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part5_texture_lifecycle.md)

### BC(Block Compressed) 텍스처란

```
일반 텍스처 (RGBA8):
┌──┬──┬──┬──┐
│R │G │B │A │  픽셀 1개 = 4 bytes
└──┴──┴──┴──┘

BC1 텍스처:
┌──────────────────────────────────────┐
│ 4×4 픽셀 블록 = 8 bytes (BC1 기준) │  픽셀 1개당 0.5 bytes
└──────────────────────────────────────┘
```

BC 포맷은 4×4 픽셀을 하나의 블록으로 묶어 압축한다.
`pixels_per_block = 4` (가로 방향 픽셀 수).

Mip 크기 계산 시 "나머지 픽셀"이 중요하다:
- width = 6 pixels일 때: 블록 수 = ceil(6/4) = 2 blocks (4+2 픽셀, 2픽셀도 블록 1개 차지)
- 단순 나눗셈으로는 1 block만 계산됨 → 2번째 블록 데이터 누락

### Subresource index 계산

DX12에서 텍스처의 특정 mip/slice/plane을 지정할 때 Subresource index를 사용한다.

```
subresource = mip + (slice × mip_levels) + (plane × mip_levels × array_size)
```

depth/stencil 포맷(D24S8)처럼 두 개의 plane이 있는 경우 plane이 2가 됨.
일반 컬러 텍스처는 plane = 0이므로:

```
총 subresource 개수 = mip_levels × array_size
```

이것이 `GetTextureSubresourceCount`가 반환하는 값이다.

### descriptors_res란

`allocationhandler->descriptors_res`는 CBV/SRV/UAV descriptor heap을 관리하는 할당자다.
RTV는 `descriptors_rtv`, DSV는 `descriptors_dsv`, SRV/CBV/UAV는 `descriptors_res`를 사용한다.
