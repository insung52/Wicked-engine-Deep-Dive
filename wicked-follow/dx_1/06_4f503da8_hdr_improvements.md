# 커밋 #6: 4f503da8 - HDR improvements

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `4f503da8` |
| 날짜 | 2025-03-21 |
| 작성자 | Turánszki János |
| 카테고리 | 기능 (Feature) |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphics.h | `ResourceState::SWAPCHAIN` 추가 |
| wiGraphicsDevice_DX12.cpp | SWAPCHAIN → PRESENT 매핑, GetBackBuffer layout 설정 |
| wiApplication.cpp | HDR10 렌더링 파이프라인 개선 |
| 기타 | 총 14개 파일 (+145줄, -106줄) |

---

## 배경 지식: HDR과 색공간

### SDR vs HDR

| 구분 | SDR (Standard Dynamic Range) | HDR (High Dynamic Range) |
|------|------------------------------|--------------------------|
| 밝기 범위 | 0 ~ 100 nits | 0 ~ 1000+ nits |
| 색심도 | 8bit (256 단계) | 10bit (1024 단계) |
| 색공간 | sRGB | Rec.2020, DCI-P3 |
| 모니터 | 일반 모니터 | HDR 지원 모니터 |

### 색공간 종류

```
┌─────────────────────────────────────────────────────────────┐
│                      색공간 비교                             │
├─────────────────────────────────────────────────────────────┤
│  sRGB (SDR)        : 일반 모니터, 웹 표준                    │
│  HDR10_ST2084      : HDR TV/모니터, PQ 곡선 (비선형)         │
│  HDR_LINEAR        : 선형 HDR, scRGB (렌더링용)              │
└─────────────────────────────────────────────────────────────┘
```

### PQ (Perceptual Quantizer) 곡선

HDR10_ST2084는 **비선형** 색공간입니다:

```
                밝기 (nits)
                    │
               1000 ┤                    ╭───
                    │                 ╭──╯
                500 ┤              ╭──╯
                    │           ╭──╯
                100 ┤        ╭──╯
                    │     ╭──╯
                 10 ┤  ╭──╯
                    │╭─╯
                  0 ┼───┴────┴────┴────┴────
                    0   0.25  0.5  0.75   1
                          코드 값

        PQ 곡선: 인간의 눈 감도에 최적화된 비선형 곡선
```

**문제**: 비선형 공간에서 선형 블렌딩 수행 시 **색상 왜곡** 발생

---

## 문제 상황

### HDR10에서 블렌딩 문제

```cpp
// 일반적인 알파 블렌딩
result = src * alpha + dst * (1 - alpha)
```

**SDR/Linear에서**:
```
src = 0.5 (회색)
dst = 1.0 (흰색)
alpha = 0.5

result = 0.5 * 0.5 + 1.0 * 0.5 = 0.75  ✅ 올바른 회색
```

**HDR10 (비선형)에서**:
```
src = PQ(500 nits) = 0.7  (비선형 인코딩)
dst = PQ(1000 nits) = 0.9
alpha = 0.5

잘못된 결과 = 0.7 * 0.5 + 0.9 * 0.5 = 0.8
실제 밝기 = PQ^-1(0.8) = 약 700 nits

올바른 결과 = (500 + 1000) / 2 = 750 nits  ❌ 색상 왜곡!
```

### 해결 전략: 선형 공간에서 작업

```
┌─────────────────────────────────────────────────────────────────┐
│                     HDR10 렌더링 파이프라인                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Linear Space]              [HDR10_ST2084]                     │
│                                                                 │
│  rendertargetPreHDR10  ──────────────▶  swapChain               │
│  (R11G11B10_FLOAT)         톤매핑        (R10G10B10A2_UNORM)     │
│       │                                                         │
│       │                                                         │
│    블렌딩/합성                        최종 출력만                 │
│    UI 렌더링                          (변환만 수행)               │
│    포스트프로세싱                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**핵심**:
- 모든 블렌딩/합성은 **선형 공간** (rendertargetPreHDR10)에서 수행
- 최종 출력 시에만 **HDR10 변환**

---

## 배경 지식: DX12 Resource State와 Barrier

### Resource State란?

DX12에서 모든 GPU 리소스(텍스처, 버퍼)는 **현재 상태(State)**를 가집니다.
GPU가 리소스를 어떤 용도로 사용하는지를 나타냅니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    주요 Resource State                           │
├─────────────────────────────────────────────────────────────────┤
│  RENDER_TARGET      : 렌더 타겟으로 사용 (쓰기)                   │
│  SHADER_RESOURCE    : 셰이더에서 읽기 (SRV)                       │
│  UNORDERED_ACCESS   : 셰이더에서 읽기/쓰기 (UAV)                  │
│  COPY_SOURCE        : 복사 원본                                   │
│  COPY_DEST          : 복사 대상                                   │
│  PRESENT            : 스왑체인 Present용 (화면 출력)              │
└─────────────────────────────────────────────────────────────────┘
```

### 왜 State가 필요한가?

GPU는 **캐시와 압축**을 사용합니다. 같은 리소스라도 용도에 따라 내부 처리가 다릅니다:

```
렌더 타겟으로 쓸 때:
┌──────────────────────────────────────┐
│  GPU 내부 캐시 (빠른 쓰기용)          │
│  압축된 형태로 저장 (대역폭 절약)      │
└──────────────────────────────────────┘

셰이더에서 읽을 때:
┌──────────────────────────────────────┐
│  압축 해제 필요                       │
│  읽기용 캐시로 전환                   │
└──────────────────────────────────────┘
```

**State가 다르면 GPU 내부 동작이 다름** → State 전환 시 **Barrier** 필요

### Barrier란?

리소스의 State를 전환하는 명령입니다:

```cpp
// "이 텍스처를 RENDER_TARGET에서 SHADER_RESOURCE로 전환해줘"
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource = texture;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;  // 이전 상태
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;  // 새 상태
commandList->ResourceBarrier(1, &barrier);
```

**핵심**: `StateBefore`가 **실제 현재 상태와 일치**해야 함!

```
올바른 경우:
  실제 상태: RENDER_TARGET
  StateBefore: RENDER_TARGET  ✅
  → GPU가 올바르게 전환

잘못된 경우:
  실제 상태: PRESENT
  StateBefore: SHADER_RESOURCE  ❌
  → GPU가 혼란, undefined behavior
```

---

## desc.layout의 역할

### TextureDesc 구조

```cpp
struct TextureDesc {
    uint32_t width;
    uint32_t height;
    Format format;
    // ...
    ResourceState layout;  // ← 이 텍스처의 "현재 상태"를 추적
};
```

`desc.layout`은 **소프트웨어 측에서 리소스의 현재 상태를 추적**하는 필드입니다.

### 배리어 호출 시 layout 사용

```cpp
// GPUBarrier::Image 내부 동작
static GPUBarrier Image(const Texture* texture,
                        ResourceState before,
                        ResourceState after)
{
    // before = texture->desc.layout 을 사용하는 경우가 많음
    GPUBarrier barrier;
    barrier.image.texture = texture;
    barrier.image.layout_before = before;
    barrier.image.layout_after = after;
    return barrier;
}
```

**문제**: `desc.layout`이 실제 상태와 다르면, 배리어의 `before` 상태가 틀림!

---

## 스왑체인이 특별한 이유

### 일반 텍스처 vs 스왑체인

| 구분 | 일반 텍스처 | 스왑체인 백버퍼 |
|------|-------------|-----------------|
| 생성 | `CreateTexture()` | `CreateSwapChain()` 내부에서 자동 생성 |
| 초기 상태 | 지정 가능 (`SHADER_RESOURCE` 등) | **항상 `PRESENT`** |
| 소유권 | 애플리케이션 | DXGI (운영체제) |
| 특수 용도 | 없음 | 화면 출력 (Present) |

### 스왑체인의 생명주기

```
┌─────────────────────────────────────────────────────────────────┐
│                    스왑체인 백버퍼 상태 흐름                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [PRESENT] ──Barrier──▶ [RENDER_TARGET] ──Barrier──▶ [PRESENT] │
│      │                        │                          │      │
│      │                        │                          │      │
│   Present 후              렌더링 중                   Present 전  │
│   (DXGI 소유)            (GPU가 그림)               (DXGI에 반환) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**핵심**: 스왑체인 백버퍼는 **PRESENT 상태로 시작하고 끝남**

### GetBackBuffer의 문제

```cpp
// 기존 코드
Texture GetBackBuffer(const SwapChain* swapchain) const
{
    Texture result;
    result.desc = _ConvertTextureDesc_Inv(resourcedesc);
    // desc.layout 설정 안함 → 기본값 SHADER_RESOURCE
    return result;
}
```

`_ConvertTextureDesc_Inv()`는 DX12의 `D3D12_RESOURCE_DESC`를 엔진의 `TextureDesc`로 변환하는데,
**리소스의 현재 상태(State)는 DESC에 없음** → `layout`은 기본값으로 남음

```
GetBackBuffer() 호출 후:
  실제 상태: PRESENT (스왑체인이므로)
  desc.layout: SHADER_RESOURCE (기본값) ❌ 불일치!
```

---

## DX12 변경 1: SWAPCHAIN ResourceState 추가

### 기존 문제

엔진의 `ResourceState` enum에 `PRESENT`를 표현할 방법이 없었습니다:

```cpp
enum class ResourceState {
    UNDEFINED,
    SHADER_RESOURCE,
    RENDER_TARGET,
    COPY_SRC,
    COPY_DST,
    // ... PRESENT가 없음!
};
```

### 해결: SWAPCHAIN 상태 추가

**wiGraphics.h**:
```cpp
enum class ResourceState {
    // ... 기존 상태들 ...
    VIDEO_DECODE_SRC = 1 << 15,
    VIDEO_DECODE_DST = 1 << 16,
    SWAPCHAIN = 1 << 17,  // ← 새로 추가: PRESENT 상태를 표현
};
```

**wiGraphicsDevice_DX12.cpp**:
```cpp
constexpr D3D12_RESOURCE_STATES _ConvertResourceStates(ResourceState value)
{
    D3D12_RESOURCE_STATES ret = {};

    // ... 기존 매핑 ...
    if (has_flag(value, ResourceState::RENDER_TARGET))
        ret |= D3D12_RESOURCE_STATE_RENDER_TARGET;

    if (has_flag(value, ResourceState::SHADER_RESOURCE))
        ret |= D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE
             | D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE;

    // 새로 추가
    if (has_flag(value, ResourceState::SWAPCHAIN))
        ret |= D3D12_RESOURCE_STATE_PRESENT;

    return ret;
}
```

### 상태 매핑 표

| ResourceState (엔진) | D3D12_RESOURCE_STATES |
|----------------------|----------------------|
| `SHADER_RESOURCE` | `PIXEL_SHADER_RESOURCE \| NON_PIXEL_SHADER_RESOURCE` |
| `RENDER_TARGET` | `RENDER_TARGET` |
| `COPY_SRC` | `COPY_SOURCE` |
| `COPY_DST` | `COPY_DEST` |
| **`SWAPCHAIN`** | **`PRESENT`** |

---

## DX12 변경 2: GetBackBuffer에서 layout 설정

### 변경 전 - 문제 상황

```cpp
Texture GraphicsDevice_DX12::GetBackBuffer(const SwapChain* swapchain) const
{
    // ...
    Texture result;
    result.type = GPUResource::Type::TEXTURE;
    result.internal_state = internal_state;
    result.desc = _ConvertTextureDesc_Inv(resourcedesc);
    // desc.layout은 기본값 (SHADER_RESOURCE) ← 문제!
    return result;
}
```

**실제 상황**:
```
스왑체인 백버퍼의 실제 DX12 상태: D3D12_RESOURCE_STATE_PRESENT
엔진이 추적하는 상태 (desc.layout): SHADER_RESOURCE

→ 불일치!
```

### 변경 후 - 올바른 상태 추적

```cpp
Texture GraphicsDevice_DX12::GetBackBuffer(const SwapChain* swapchain) const
{
    // ...
    Texture result;
    result.type = GPUResource::Type::TEXTURE;
    result.internal_state = internal_state;
    result.desc = _ConvertTextureDesc_Inv(resourcedesc);
    result.desc.layout = ResourceState::SWAPCHAIN;  // ← 실제 상태와 일치!
    return result;
}
```

**수정 후 상황**:
```
스왑체인 백버퍼의 실제 DX12 상태: D3D12_RESOURCE_STATE_PRESENT
엔진이 추적하는 상태 (desc.layout): SWAPCHAIN (= PRESENT)

→ 일치! ✅
```

---

## 배리어 전환: 수정 전 vs 수정 후

### 수정 전 - 잘못된 배리어

```cpp
Texture backbuffer = device->GetBackBuffer(&swapchain);
// backbuffer.desc.layout = SHADER_RESOURCE (기본값, 틀림)

// 렌더링 시작 전 배리어
device->Barrier(
    GPUBarrier::Image(&backbuffer,
        backbuffer.desc.layout,        // before: SHADER_RESOURCE ❌ 틀림!
        ResourceState::RENDER_TARGET),
    cmd
);
```

**DX12로 변환 시**:
```cpp
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;  // 틀림!
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_RENDER_TARGET;
// 실제 상태는 PRESENT인데 SHADER_RESOURCE에서 전환하라고 함
```

**결과**:
- Debug Layer 경고/에러
- 일부 드라이버에서 undefined behavior
- 운이 좋으면 드라이버가 암묵적으로 처리 (하지만 보장 안 됨)

### 수정 후 - 올바른 배리어

```cpp
Texture backbuffer = device->GetBackBuffer(&swapchain);
// backbuffer.desc.layout = SWAPCHAIN (올바름)

// 렌더링 시작 전 배리어
device->Barrier(
    GPUBarrier::Image(&backbuffer,
        backbuffer.desc.layout,        // before: SWAPCHAIN ✅ 맞음!
        ResourceState::RENDER_TARGET),
    cmd
);

// 렌더링 수행...

// Present 전 배리어
device->Barrier(
    GPUBarrier::Image(&backbuffer,
        ResourceState::RENDER_TARGET,
        ResourceState::SWAPCHAIN),     // after: SWAPCHAIN ✅
    cmd
);
```

**DX12로 변환 시**:
```cpp
// 렌더링 전
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_PRESENT;  // 맞음!
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_RENDER_TARGET;

// Present 전
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PRESENT;  // 맞음!
```

---

## 타임라인으로 보는 스왑체인 렌더링

```
시간 ──────────────────────────────────────────────────────────────────▶

┌─────────────────────────────────────────────────────────────────────┐
│ 1. GetBackBuffer()                                                  │
│    실제 상태: PRESENT                                                │
│    desc.layout: SWAPCHAIN (수정 후) / SHADER_RESOURCE (수정 전)     │
└─────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. Barrier(SWAPCHAIN → RENDER_TARGET)                               │
│    GPU: 캐시 플러시, 압축 모드 변경                                   │
│    desc.layout 갱신: RENDER_TARGET                                  │
└─────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. RenderPassBegin / Draw / RenderPassEnd                           │
│    GPU가 백버퍼에 렌더링                                             │
└─────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. Barrier(RENDER_TARGET → SWAPCHAIN)                               │
│    GPU: 렌더링 완료 대기, 캐시 플러시                                 │
│    desc.layout 갱신: SWAPCHAIN                                      │
└─────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. Present()                                                        │
│    DXGI가 백버퍼를 화면에 출력                                        │
│    (PRESENT 상태여야 함)                                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Application 레벨 변경 (참고)

VizMotive와 아키텍처가 다르므로 참고용으로만 기록합니다.

### HDR10 렌더링 흐름

```cpp
void Application::Run()
{
    ColorSpace colorspace = graphicsDevice->GetSwapChainColorSpace(&swapChain);

    if (colorspace == ColorSpace::HDR10_ST2084)
    {
        // HDR10: 중간 렌더타겟 필요
        if (!rendertargetPreHDR10.IsValid())
        {
            TextureDesc desc;
            desc.format = Format::R11G11B10_FLOAT;  // 선형 HDR 포맷
            desc.width = swapChain.desc.width;
            desc.height = swapChain.desc.height;
            graphicsDevice->CreateTexture(&desc, nullptr, &rendertargetPreHDR10);
        }

        // 1단계: 선형 공간에서 렌더링
        graphicsDevice->RenderPassBegin(&rendertargetPreHDR10, cmd, true);
        Compose(cmd);  // 모든 블렌딩/합성
        graphicsDevice->RenderPassEnd(cmd);

        // 2단계: HDR10 변환
        graphicsDevice->RenderPassBegin(&swapChain, cmd);
        wi::image::Params fx;
        fx.enableFullScreen();
        fx.enableHDR10OutputMapping();  // PQ 톤매핑
        wi::image::Draw(&rendertargetPreHDR10, fx, cmd);
        graphicsDevice->RenderPassEnd(cmd);
    }
    else
    {
        // SDR/Linear HDR: 직접 스왑체인에 렌더링
        graphicsDevice->RenderPassBegin(&swapChain, cmd);
        Compose(cmd);
        graphicsDevice->RenderPassEnd(cmd);
    }
}
```

---

## VizMotive 적용

### 적용 일자
2025-01-26

### 변경 파일 및 내용

| 파일 | 라인 | 변경 |
|------|------|------|
| `GBackend.h` | 485 | `SWAPCHAIN = 1 << 17` 추가 |
| `GraphicsDevice_DX12.cpp` | 121-122 | `SWAPCHAIN → D3D12_RESOURCE_STATE_PRESENT` 매핑 |
| `GraphicsDevice_DX12.cpp` | 5960 | `GetBackBuffer`에서 `desc.layout = ResourceState::SWAPCHAIN` 설정 |

### 적용 코드

**1. GBackend.h - ResourceState 추가**
```cpp
enum class ResourceState {
    // ... 기존 상태들 ...
    VIDEO_DECODE_SRC = 1 << 15,
    VIDEO_DECODE_DST = 1 << 16,
    SWAPCHAIN = 1 << 17,  // 추가
};
```

**2. GraphicsDevice_DX12.cpp - 상태 변환**
```cpp
constexpr D3D12_RESOURCE_STATES _ConvertResourceStates(ResourceState value)
{
    // ... 기존 코드 ...

    if (has_flag(value, ResourceState::VIDEO_DECODE_DST))
        ret |= D3D12_RESOURCE_STATE_VIDEO_DECODE_WRITE;

    // 추가
    if (has_flag(value, ResourceState::SWAPCHAIN))
        ret |= D3D12_RESOURCE_STATE_PRESENT;

    return ret;
}
```

**3. GraphicsDevice_DX12.cpp - GetBackBuffer**
```cpp
Texture GraphicsDevice_DX12::GetBackBuffer(const SwapChain* swapchain) const
{
    // ... 기존 코드 ...

    Texture result;
    result.type = GPUResource::Type::TEXTURE;
    result.internal_state = internal_state;
    result.desc = _ConvertTextureDesc_Inv(resourcedesc);
    result.desc.layout = ResourceState::SWAPCHAIN;  // 추가
    return result;
}
```

---

## 적용하지 않은 부분

| 항목 | 이유 |
|------|------|
| `rendertargetPreHDR10` | VizMotive는 자체 렌더링 파이프라인 사용 |
| HDR10 톤매핑 로직 | Application 아키텍처 다름 |
| `EnsureRenderTargetValid()` | Wicked Application 전용 함수 |

---

## 효과

### 적용 전
- 스왑체인 백버퍼의 `desc.layout`이 기본값 (`SHADER_RESOURCE`)
- 실제 상태 (`PRESENT`)와 불일치
- 배리어 전환 시 잠재적 문제 (드라이버가 암묵적 처리)

### 적용 후
- 스왑체인 상태 명시적 관리 (`ResourceState::SWAPCHAIN`)
- 배리어 전환 시 정확한 시작/끝 상태
- 디버그 레이어 경고 감소
- 코드 의도 명확화

---

## 요약

| 변경 전 | 변경 후 |
|---------|---------|
| SWAPCHAIN 상태 없음 | `ResourceState::SWAPCHAIN = 1 << 17` |
| GetBackBuffer layout 기본값 | `desc.layout = SWAPCHAIN` |
| 암묵적 상태 처리 | 명시적 상태 관리 |

### 핵심 포인트

> **스왑체인 = 특별한 리소스**
>
> - 초기 상태가 `PRESENT` (다른 리소스와 다름)
> - `ResourceState::SWAPCHAIN`으로 명시적 표현
> - 배리어 전환 정확성 향상
