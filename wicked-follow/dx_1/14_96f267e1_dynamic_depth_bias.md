# 커밋 #14: 96f267e1 - Improved Shadow Bias (Dynamic Depth Bias)

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `96f267e1` |
| 날짜 | 2025-05-24 |
| 작성자 | Turánszki János |
| 카테고리 | 렌더링 / DX12 기능 |
| 우선순위 | 낮음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | PSO_STREAM1에 Flags 필드 추가 |
| 셰이더 파일들 | Shadow bias 튜닝값 조정 |

**참고**: 커밋의 주요 목적은 셰이더 shadow bias 튜닝이며, DX12 변경은 **Dynamic Depth Bias 지원을 위한 준비**입니다.

---

## 배경 지식: Depth Bias란?

### Shadow Acne 문제

섀도우 맵 렌더링에서 가장 흔한 아티팩트 중 하나입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                       Shadow Acne 발생 원인                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Light ───→  ════════════════════  ← 표면 (Surface)            │
│              ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓                               │
│              섀도우맵 텍셀들                                     │
│                                                                 │
│  문제:                                                          │
│  - 섀도우맵 해상도 한계로 인해 깊이값이 양자화됨                  │
│  - 표면의 픽셀이 자신의 깊이와 섀도우맵 깊이를 비교할 때          │
│  - 부동소수점 오차로 "나는 그림자 안이다"라고 잘못 판단           │
│  - 결과: 줄무늬 패턴의 아티팩트                                  │
│                                                                 │
│  ┌──────────────────────────────┐                               │
│  │ ░░██░░██░░██░░██░░██░░██░░██ │  ← Shadow Acne               │
│  │ ██░░██░░██░░██░░██░░██░░██░░ │    (얼룩덜룩한 그림자)        │
│  │ ░░██░░██░░██░░██░░██░░██░░██ │                               │
│  └──────────────────────────────┘                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Depth Bias 해결책

섀도우맵에 깊이를 기록할 때 약간의 오프셋을 추가합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Depth Bias 적용                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  원래 깊이값:    0.50000                                        │
│  Depth Bias:    +0.00001                                        │
│  ─────────────────────────                                      │
│  보정된 깊이값:  0.50001   ← 표면을 빛에서 살짝 멀어지게         │
│                                                                 │
│  효과: 표면이 자기 자신의 그림자에 가려지지 않음                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Depth Bias 파라미터 3가지

```cpp
D3D12_RASTERIZER_DESC rs = {};
rs.DepthBias = 100;              // 1. 고정 오프셋
rs.DepthBiasClamp = 0.0f;        // 2. 최대 bias 제한
rs.SlopeScaledDepthBias = 1.0f;  // 3. 기울기 비례 오프셋
```

| 파라미터 | 설명 |
|----------|------|
| `DepthBias` | 모든 픽셀에 적용되는 고정 오프셋 |
| `DepthBiasClamp` | bias가 이 값을 초과하지 않도록 제한 |
| `SlopeScaledDepthBias` | 표면 기울기에 비례하는 추가 오프셋 |

**Slope Scaled Depth Bias가 필요한 이유**:

```
┌─────────────────────────────────────────────────────────────────┐
│              표면 기울기에 따른 Shadow Acne 차이                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  정면 표면 (Light와 수직)     기울어진 표면 (Light와 비스듬)     │
│                                                                 │
│  Light                        Light                             │
│    ↓                            ↘                               │
│  ═════  ← 오차 적음            ╲═══  ← 오차 큼!                 │
│                                  ╲                              │
│                                                                 │
│  고정 bias로 충분              더 큰 bias 필요                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Depth Bias 계산 공식

```
최종 Bias = DepthBias × r + SlopeScaledDepthBias × MaxDepthSlope

여기서:
- r = 깊이 버퍼 포맷의 최소 표현 가능 값 (24bit의 경우 1/2^24)
- MaxDepthSlope = max(|∂z/∂x|, |∂z/∂y|) - 표면의 최대 기울기
```

---

## 배경 지식: Static vs Dynamic Depth Bias

### Static Depth Bias (기존 방식)

```cpp
// Rasterizer State에 bias 값이 고정됨
D3D12_RASTERIZER_DESC rs = {};
rs.DepthBias = 100;                  // INT 타입!
rs.SlopeScaledDepthBias = 1.0f;

// Rasterizer State는 PSO의 일부
// → bias 값을 바꾸려면 새 PSO 필요!
```

**문제: PSO 폭발**

상황마다 다른 bias가 필요합니다:

| 상황 | 필요한 bias |
|------|-------------|
| Directional Light Cascade 0 | 낮음 |
| Directional Light Cascade 3 | 높음 |
| Point Light | 중간 |
| Spot Light | 중간 |
| 얇은 오브젝트 | 매우 낮음 |
| 투명 오브젝트 | 조정 필요 |

→ 각 bias 조합마다 별도의 PSO가 필요 = **수십 개의 PSO**

### Dynamic Depth Bias (DX12 새 기능)

```cpp
// 1. PSO 생성 시 Dynamic 플래그 설정
D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc = {};
psoDesc.Flags = D3D12_PIPELINE_STATE_FLAG_DYNAMIC_DEPTH_BIAS;

// 2. 런타임에 bias 동적 변경 (PSO 재생성 없이!)
ID3D12GraphicsCommandList9* cmdList9 = ...;

// Cascade 0 렌더링
cmdList9->RSSetDepthBias(0.0001f, 0.0f, 1.0f);
DrawCascade0();

// Cascade 3 렌더링 (같은 PSO, 다른 bias)
cmdList9->RSSetDepthBias(0.001f, 0.0f, 4.0f);
DrawCascade3();
```

### 비교 요약

```
┌─────────────────────────────────────────────────────────────────┐
│              Static vs Dynamic Depth Bias                        │
├────────────────────────────┬────────────────────────────────────┤
│  Static (기존)              │  Dynamic (새로움)                  │
├────────────────────────────┼────────────────────────────────────┤
│  PSO에 bias 고정           │  런타임에 bias 변경 가능           │
│  bias마다 PSO 필요         │  PSO 1개로 모든 bias 처리          │
│  INT 타입 DepthBias        │  FLOAT 타입 DepthBias              │
│  PSO 전환 오버헤드         │  단순 상태 변경 (저렴)             │
│  모든 DX12 버전            │  Agility SDK 1.608.0+              │
└────────────────────────────┴────────────────────────────────────┘
```

---

## 배경 지식: DepthBias 타입 변경 (INT → FLOAT)

### 기존 D3D12_RASTERIZER_DESC

```cpp
typedef struct D3D12_RASTERIZER_DESC {
    D3D12_FILL_MODE FillMode;
    D3D12_CULL_MODE CullMode;
    BOOL FrontCounterClockwise;
    INT DepthBias;                    // 정수!
    FLOAT DepthBiasClamp;
    FLOAT SlopeScaledDepthBias;
    // ...
} D3D12_RASTERIZER_DESC;
```

**INT 타입의 불편함**:
```cpp
// 실제 bias = DepthBias × (1 / 2^24)   (24bit 깊이 버퍼 기준)
//
// 원하는 bias: 0.00001
// 필요한 DepthBias = 0.00001 × 2^24 = 167.77...
// → INT로 반올림: 168
// → 실제 bias: 168 / 2^24 = 0.0000100135...
//
// 정밀도 손실 발생!
```

### 새로운 D3D12_RASTERIZER_DESC1

```cpp
typedef struct D3D12_RASTERIZER_DESC1 {
    D3D12_FILL_MODE FillMode;
    D3D12_CULL_MODE CullMode;
    BOOL FrontCounterClockwise;
    FLOAT DepthBias;                  // 실수! 직접 지정 가능
    FLOAT DepthBiasClamp;
    FLOAT SlopeScaledDepthBias;
    // ...
} D3D12_RASTERIZER_DESC1;
```

**FLOAT 타입의 장점**:
```cpp
// 원하는 bias를 직접 지정
rs.DepthBias = 0.00001f;  // 그대로 사용됨!
```

---

## DX12 변경 내용

### Wicked Engine의 변경

**PSO_STREAM1 구조체에 Flags 필드 추가**:

```cpp
// 변경 전
struct PSO_STREAM1
{
    CD3DX12_PIPELINE_STATE_STREAM_VS VS;
    CD3DX12_PIPELINE_STATE_STREAM_HS HS;
    CD3DX12_PIPELINE_STATE_STREAM_DS DS;
    // ... 나머지 필드들
};

// 변경 후
struct PSO_STREAM1
{
    CD3DX12_PIPELINE_STATE_STREAM_FLAGS Flags;  // ← 추가!
    CD3DX12_PIPELINE_STATE_STREAM_VS VS;
    CD3DX12_PIPELINE_STATE_STREAM_HS HS;
    CD3DX12_PIPELINE_STATE_STREAM_DS DS;
    // ... 나머지 필드들
};
```

### 실제 사용 여부

**Wicked Engine 주석**:
```cpp
// D3D12_PIPELINE_STATE_FLAG_DYNAMIC_DEPTH_BIAS는 Windows 10에서 미작동
```

→ 구조체만 준비해두고, 실제 dynamic depth bias는 **사용하지 않음**

---

## 요구 사항

| 항목 | 요구 사항 |
|------|-----------|
| Agility SDK | 1.608.0 이상 (2022년 12월 릴리스) |
| Windows | 10 버전 1909 (19H2) 이상 |
| Command List | ID3D12GraphicsCommandList9 |
| 하드웨어 | 드라이버 지원 필요 |

### ID3D12GraphicsCommandList9::RSSetDepthBias

```cpp
void RSSetDepthBias(
    FLOAT DepthBias,
    FLOAT DepthBiasClamp,
    FLOAT SlopeScaledDepthBias
);
```

**사용 조건**:
- PSO가 `D3D12_PIPELINE_STATE_FLAG_DYNAMIC_DEPTH_BIAS` 플래그로 생성되어야 함
- 플래그 없이 호출하면 Debug Layer 에러

---

## VizMotive 현재 상태

### PSO_STREAM1 구조체

```cpp
// GraphicsDevice_DX12.cpp:1430
struct PSO_STREAM1
{
    CD3DX12_PIPELINE_STATE_STREAM_VS VS;
    CD3DX12_PIPELINE_STATE_STREAM_HS HS;
    CD3DX12_PIPELINE_STATE_STREAM_DS DS;
    CD3DX12_PIPELINE_STATE_STREAM_GS GS;
    CD3DX12_PIPELINE_STATE_STREAM_PS PS;
    CD3DX12_PIPELINE_STATE_STREAM_RASTERIZER RS;
    // ... 나머지
    // Flags 필드 없음!
};
```

### Static Depth Bias 사용

```cpp
// GraphicsDevice_DX12.cpp:4203-4205
rs.DepthBias = pRasterizerStateDesc.depth_bias;
rs.DepthBiasClamp = pRasterizerStateDesc.depth_bias_clamp;
rs.SlopeScaledDepthBias = pRasterizerStateDesc.slope_scaled_depth_bias;
```

---

## VizMotive 적용 권장

### 우선순위: 낮음

**이유**:
1. Wicked Engine도 실제로 사용하지 않음 (구조체만 준비)
2. Static depth bias로 충분히 동작
3. ID3D12GraphicsCommandList9 인터페이스 추가 필요
4. Agility SDK 버전 의존성 추가

### 향후 적용 시 필요 작업

```cpp
// 1. PSO_STREAM1에 Flags 필드 추가
struct PSO_STREAM1
{
    CD3DX12_PIPELINE_STATE_STREAM_FLAGS Flags;  // 추가
    // ... 기존 필드들
};

// 2. Command List 인터페이스 업그레이드
ComPtr<ID3D12GraphicsCommandList9> commandList9;
commandList->QueryInterface(IID_PPV_ARGS(&commandList9));

// 3. Dynamic Depth Bias 호출
if (commandList9)
{
    commandList9->RSSetDepthBias(bias, clamp, slope);
}
```

### 적용하면 좋은 경우

- 섀도우 캐스케이드마다 다른 bias가 필요한 경우
- 런타임에 bias 튜닝이 필요한 경우 (에디터/디버그)
- PSO 개수를 줄이고 싶은 경우

---

## 참고 자료

- [D3D12_PIPELINE_STATE_FLAGS - Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_pipeline_state_flags)
- [Agility SDK 1.608.0 - DirectX Developer Blog](https://devblogs.microsoft.com/directx/agility-sdk-1-608-0/)
- [VulkanOn12 Specs - DirectX-Specs](https://microsoft.github.io/DirectX-Specs/d3d/VulkanOn12.html)
- [Depth Bias - Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/direct3d11/d3d10-graphics-programming-guide-output-merger-stage-depth-bias)

---

## 요약

| 항목 | 내용 |
|------|------|
| 기능 | Dynamic Depth Bias - 런타임에 depth bias 변경 가능 |
| 장점 | PSO 폭발 방지, FLOAT 타입 정밀도, 유연한 섀도우 품질 조정 |
| 요구사항 | Agility SDK 1.608.0+, Windows 10 1909+, ID3D12GraphicsCommandList9 |
| Wicked Engine | Flags 필드만 추가, 실제 사용 안 함 |
| VizMotive | 미적용 (Flags 필드 없음) |
| 적용 권장 | 낮음 (향후 섀도우 품질 개선 시 검토) |
