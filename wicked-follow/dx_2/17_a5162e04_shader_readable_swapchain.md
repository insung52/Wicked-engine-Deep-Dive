# 커밋 #17: a5162e04 - Added support for shader-readable swapchain

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `a5162e04` |
| 날짜 | 2025-12-01 |
| 작성자 | Turánszki János |
| 카테고리 | 기능 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | SwapChain을 쉐이더에서 읽을 수 있도록 SRV 지원 추가 |

---

## 배경 지식: SwapChain과 Back Buffer

### SwapChain 구조

```
┌─────────────────────────────────────────────────────────────────┐
│              SwapChain (Triple Buffering)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Back Buffer │  │ Back Buffer │  │ Back Buffer │              │
│  │     [0]     │  │     [1]     │  │     [2]     │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│         │                │                │                      │
│         ▼                ▼                ▼                      │
│  ┌─────────────────────────────────────────────────┐            │
│  │              Present (화면에 표시)               │            │
│  └─────────────────────────────────────────────────┘            │
│                         │                                        │
│                         ▼                                        │
│                    [Monitor]                                     │
│                                                                 │
│  Back Buffer 순환:                                              │
│  Frame 0: Render to [0] → Present [0]                           │
│  Frame 1: Render to [1] → Present [1]                           │
│  Frame 2: Render to [2] → Present [2]                           │
│  Frame 3: Render to [0] → Present [0] (다시 [0])                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기존 Back Buffer 사용 방식

```cpp
// 기존: RTV (Render Target View)만 지원
// → 쉐이더에서 "쓰기"만 가능

commandList->OMSetRenderTargets(1, &backBufferRTV, ...);
// 쉐이더가 back buffer에 픽셀 출력
```

---

## 문제: Back Buffer를 쉐이더에서 읽을 수 없음

### 왜 읽기가 필요한가?

```
┌─────────────────────────────────────────────────────────────────┐
│              Back Buffer 읽기 필요 사례                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Post-Processing (후처리)                                    │
│     ─────────────────────────                                   │
│     Back Buffer에 렌더링 → 블러/블룸 효과 적용 → 다시 출력      │
│                                                                 │
│     기존: 중간 텍스처에 복사 필요                               │
│     새로운: Back Buffer를 직접 SRV로 읽기                       │
│                                                                 │
│  2. 화면 캡처/스크린샷                                          │
│     ─────────────────────                                       │
│     현재 화면 내용을 쉐이더에서 읽어서 처리                     │
│                                                                 │
│  3. 반투명 효과                                                 │
│     ─────────────                                               │
│     이전 프레임 결과를 읽어서 블렌딩                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기존 코드의 한계

```cpp
// 기존 SwapChain_DX12 구조
struct SwapChain_DX12
{
    ComPtr<IDXGISwapChain4> swapChain;
    std::vector<ComPtr<ID3D12Resource>> backBuffers;  // 리소스만
    std::vector<D3D12_CPU_DESCRIPTOR_HANDLE> backbufferRTV;  // RTV만!
    // SRV 없음 → 쉐이더에서 읽기 불가
};

// GetBackBuffer() - 매번 새 객체 생성
Texture GetBackBuffer(const SwapChain* swapchain)
{
    Texture result;
    result.internal_state = make_shared<Texture_DX12>();
    // 매번 새로 internal_state 생성 → 비효율
    // SRV도 없음
}
```

---

## 해결: SwapChain에 SRV 지원 추가

### 1. SwapChain_DX12 구조 변경

```cpp
// 변경 전
struct SwapChain_DX12
{
    ComPtr<IDXGISwapChain4> swapChain;
    std::vector<ComPtr<ID3D12Resource>> backBuffers;
    std::vector<D3D12_CPU_DESCRIPTOR_HANDLE> backbufferRTV;

    ~SwapChain_DX12()
    {
        for (auto& x : backbufferRTV)
            allocationhandler->descriptors_rtv.free(x);
    }
};

// 변경 후
struct SwapChain_DX12
{
    ComPtr<IDXGISwapChain4> swapChain;
    std::vector<shared_ptr<Texture_DX12>> textures;  // ✅ 완전한 Texture 객체!
    // 소멸자 제거 - Texture_DX12가 자체 정리
};
```

### 2. CreateSwapChain 변경

```cpp
bool GraphicsDevice_DX12::CreateSwapChain(SwapChain* swapchain, ...)
{
    // ✅ DXGI_USAGE_SHADER_INPUT 플래그 추가
    DXGI_SWAP_CHAIN_DESC1 swapChainDesc = {};
    swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT
                              | DXGI_USAGE_SHADER_INPUT;  // 쉐이더 읽기 가능!

    // SwapChain 생성
    factory->CreateSwapChainForHwnd(..., &swapChainDesc, ...);

    // 각 back buffer를 완전한 Texture_DX12 객체로 생성
    for (uint32_t i = 0; i < bufferCount; ++i)
    {
        auto texture_internal = make_shared<Texture_DX12>();
        texture_internal->allocationhandler = allocationhandler;

        // 리소스 가져오기
        swapChain->GetBuffer(i, IID_PPV_ARGS(&texture_internal->resource));

        // ✅ RTV 생성
        texture_internal->rtv.handle = allocationhandler->descriptors_rtv.allocate();
        device->CreateRenderTargetView(
            texture_internal->resource.Get(),
            &rtv_desc,
            texture_internal->rtv.handle
        );

        // ✅ SRV 생성 (새로 추가!)
        D3D12_SHADER_RESOURCE_VIEW_DESC srv_desc = {};
        srv_desc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
        srv_desc.Format = _ConvertFormat(desc->format);
        srv_desc.Texture2D.MipLevels = 1;
        srv_desc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
        texture_internal->srv.init(this, srv_desc, texture_internal->resource.Get());

        internal_state->textures.push_back(texture_internal);
    }
}
```

### 3. GetBackBuffer 변경

```cpp
// 변경 전 - 매번 새 객체 생성
Texture GetBackBuffer(const SwapChain* swapchain)
{
    Texture result;
    result.internal_state = make_shared<Texture_DX12>();
    // 매번 새로 설정...
    return result;
}

// 변경 후 - 미리 생성된 텍스처 반환
Texture GetBackBuffer(const SwapChain* swapchain)
{
    auto internal_state = to_internal(swapchain);
    uint32_t index = internal_state->GetBufferIndex();

    Texture result;
    result.internal_state = internal_state->textures[index];  // ✅ 기존 객체 재사용
    result.desc = swapchain->desc;
    return result;
}
```

### 4. RenderPass/Present 변경

```cpp
// 모든 backBuffers[index] 참조를 textures[index]->resource로 변경
// 모든 backbufferRTV[index] 참조를 textures[index]->rtv.handle로 변경

// RenderPassBegin
barrier.Transition.pResource = internal_state->textures[index]->resource.Get();

// OMSetRenderTargets
RTV.cpuDescriptor = internal_state->textures[index]->rtv.handle;
```

---

## 다이어그램: 구조 변경

```
┌─────────────────────────────────────────────────────────────────┐
│                      변경 전                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SwapChain_DX12:                                                │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ backBuffers[0] ──→ ID3D12Resource                      │     │
│  │ backBuffers[1] ──→ ID3D12Resource                      │     │
│  │ backBuffers[2] ──→ ID3D12Resource                      │     │
│  │                                                        │     │
│  │ backbufferRTV[0] ──→ RTV Handle                        │     │
│  │ backbufferRTV[1] ──→ RTV Handle                        │     │
│  │ backbufferRTV[2] ──→ RTV Handle                        │     │
│  │                                                        │     │
│  │ SRV 없음! ❌                                           │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  GetBackBuffer() 호출 시:                                       │
│  → 매번 새 Texture_DX12 생성 (비효율)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      변경 후                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SwapChain_DX12:                                                │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ textures[0] ──→ Texture_DX12                           │     │
│  │                 ├─ resource (ID3D12Resource)           │     │
│  │                 ├─ rtv (RTV Handle) ✅                 │     │
│  │                 └─ srv (SRV Handle) ✅ 새로 추가!      │     │
│  │                                                        │     │
│  │ textures[1] ──→ Texture_DX12                           │     │
│  │                 ├─ resource                            │     │
│  │                 ├─ rtv ✅                              │     │
│  │                 └─ srv ✅                              │     │
│  │                                                        │     │
│  │ textures[2] ──→ Texture_DX12                           │     │
│  │                 ├─ resource                            │     │
│  │                 ├─ rtv ✅                              │     │
│  │                 └─ srv ✅                              │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  GetBackBuffer() 호출 시:                                       │
│  → textures[index] 재사용 (효율적)                              │
│  → SRV로 쉐이더에서 읽기 가능 ✅                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 사용 예시

### Back Buffer를 쉐이더 입력으로 사용

```cpp
// 1. Back buffer를 SRV로 바인딩
Texture backBuffer = device->GetBackBuffer(&swapchain);
device->BindResource(backBuffer, 0, cmd);  // SRV slot 0

// 2. 다른 렌더 타겟에 후처리 효과 적용
device->RenderPassBegin(&postProcessPass, cmd);
device->Draw(...);  // 쉐이더가 back buffer를 읽어서 처리
device->RenderPassEnd(cmd);

// 3. 결과를 back buffer에 다시 출력
device->RenderPassBegin(&swapchainPass, cmd);
// ...
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | SwapChain_DX12: backBuffers + backbufferRTV → textures 통합 |
| GraphicsDevice_DX12.cpp | SwapChain_DX12 소멸자 제거 |
| GraphicsDevice_DX12.cpp | CreateSwapChain: `DXGI_USAGE_SHADER_INPUT` 추가 |
| GraphicsDevice_DX12.cpp | CreateSwapChain: SRV descriptor 생성 추가 |
| GraphicsDevice_DX12.cpp | GetBackBuffer: 미리 생성된 textures[index] 반환 |
| GraphicsDevice_DX12.cpp | RenderPassBegin: textures[index]->resource/rtv.handle 사용 |

---

## 요약

| 항목 | 내용 |
|------|------|
| 목적 | SwapChain back buffer를 쉐이더에서 읽기 가능하게 |
| 변경 1 | `DXGI_USAGE_SHADER_INPUT` 플래그 추가 |
| 변경 2 | Back buffer를 완전한 Texture_DX12 객체로 관리 |
| 변경 3 | 각 back buffer에 SRV 생성 |
| 효과 | 후처리, 화면 캡처 등에서 back buffer 직접 읽기 가능 |
| VizMotive | ✅ 적용 완료 |

### 핵심 교훈

> **SwapChain Back Buffer도 완전한 텍스처로 관리**
>
> Back buffer를 단순 리소스+RTV가 아닌
> 완전한 Texture 객체(RTV+SRV)로 관리하면:
> - 쉐이더 읽기 가능
> - 일관된 리소스 처리
> - GetBackBuffer() 효율 향상

> **DXGI_USAGE 플래그 확인**
>
> SwapChain 생성 시 BufferUsage 플래그로
> 사용 방식을 지정해야 함:
> - `DXGI_USAGE_RENDER_TARGET_OUTPUT`: 렌더 타겟 (기본)
> - `DXGI_USAGE_SHADER_INPUT`: 쉐이더 입력 (SRV)
