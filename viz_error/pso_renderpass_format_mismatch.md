# PSO Render Target Format Mismatch 오류

## 오류 메시지

```
D3D12 ERROR: ID3D12CommandList::DrawInstanced:
The render target format in slot 0 does not match that specified by the current pipeline state.
(pipeline state = R10G10B10A2_UNORM, render target format = R11G11B10_FLOAT,
RTV ID3D12Resource* = ...'rtMain')
[ EXECUTION ERROR #613: RENDER_TARGET_FORMAT_MISMATCH_PIPELINE_STATE]
```

## 호출 스택 (첫 번째 오류)

```
GBackendDX12.dll!vz::graphics::GraphicsDevice_DX12::Draw
ShaderEngine.dll!vz::image::Draw  →  Image.cpp(454)
ShaderEngine.dll!vz::renderer::GRenderPath3DDetails::DrawSpritesAndFonts  →  SpriteDraw_Detail.cpp(138)
ShaderEngine.dll!vz::renderer::GRenderPath3DDetails::RenderTransparents  →  Renderer.cpp(870)
```

---

## 배경: 이전 PR과의 관계

이 오류는 이전 shadow map 오류 수정 PR의 부작용으로 발생했습니다.

> 관련 문서: [shadow_map_error_2.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/shadow/shadow_map_error_2.md)

이전 PR에서 `Image.cpp`와 `Font.cpp`의 PSO 생성 시 `RenderPassInfo`를 추가하면서,
RT 포맷을 `R10G10B10A2_UNORM`으로 하드코딩했습니다.
하지만 실제 `rtMain`의 포맷은 `R11G11B10_FLOAT`이므로 불일치가 발생했습니다.

---

## 근본 원인 분석

### PSO 생성 메커니즘

`CreatePipelineState(desc, pso, renderpass_info)` 호출 방식에 따라 동작이 달라집니다.

```
renderpass_info != nullptr
  → internal_state->resource 즉시 생성 (특정 포맷으로 컴파일된 PSO 고정)

renderpass_info == nullptr
  → internal_state->resource = null (미생성 상태 유지)
```

### BindPipelineState의 분기

```cpp
if (internal_state->resource != nullptr)
{
    // renderpass_info와 함께 생성된 경우
    SetPipelineState(internal_state->resource);  // 고정 PSO 직접 사용
    dirty_pso = false;  // pso_validate 호출 안 됨 → runtime 포맷 적응 불가
}
else
{
    // renderpass_info 없이 생성된 경우
    dirty_pso = true;  // pso_validate가 draw 시점에 현재 renderpass로 PSO 생성
}
```

### pso_validate의 runtime 적응 메커니즘

```
hash = (PSO 포인터, 현재 renderpass 해시)
→ pipelines_global 캐시에서 탐색
→ 없으면 현재 renderpass의 RT/DS 포맷으로 PSO 신규 생성
→ pipelines_global에 영구 캐싱 (이후 동일 조합 → 재사용)
```

이 메커니즘은 **하나의 PSO 객체가 여러 renderpass 포맷에 대응**할 수 있도록 설계되어 있습니다.

### imagePSO가 사용되는 RT 포맷

`imagePSO`는 단일 포맷이 아닌 여러 RT 포맷을 가진 renderpass에서 사용됩니다.

| 호출 경로 | 렌더 타겟 | 포맷 |
|-----------|-----------|------|
| RenderTransparents → DrawSpritesAndFonts | `rtMain` | `R11G11B10_FLOAT` |
| Render2D (로딩 스크린) | swapChain | `R10G10B10A2_UNORM` |
| Compose() (HDR 모드) | `rtRenderFinal_` | `R11G11B10_FLOAT` |

### 이전 PR이 문제가 된 이유

이전 PR에서 `renderpass_info`를 추가함으로써 `internal_state->resource`가 채워졌고,
`BindPipelineState`는 고정된 PSO를 직접 사용하게 됐습니다.
이로 인해 runtime 포맷 적응 메커니즘(`pso_validate`)이 완전히 비활성화됐습니다.

```
imagePSO 생성 시 rt_format = R10G10B10A2_UNORM (고정)
       ↓
RenderTransparents에서 rtMain (R11G11B10_FLOAT)에 그리기 시도
       ↓
BindPipelineState: resource != null → 고정 PSO 직접 바인딩
       ↓
pso_validate 호출 안 됨 → 포맷 불일치 그대로
       ↓
Draw: ERROR #613 RENDER_TARGET_FORMAT_MISMATCH
```

---

## 이전 PR의 실제 버그 (shadow_map_error_2)

이전 PR이 해결해야 했던 실제 문제는 `debugPSO` 하나였습니다.

```
debugPSO DSS 설정: STENCILMODE_ALWAYS (stencil_enable = true)
       ↓
Compose()에서 사용: DS buffer 없는 renderpass
       ↓
pso_validate 런타임 재생성 시도:
  stencil_enable=true + ds_format=UNKNOWN → D3D12 WARNING #680 → BREAK → 크래시
```

**필요한 수정은 `debugPSO`의 DSS를 `STENCILMODE_DISABLED`로 변경하는 것뿐**이었습니다.
`imagePSO`와 `fontPSO`에 `renderpass_info`를 추가한 것은 불필요한 수정이었으며,
오히려 runtime 포맷 적응을 깨뜨리는 부작용을 낳았습니다.

---

## 해결 방법

### Image.cpp - imagePSO 생성

`renderpass_info` 블록 제거, `CreatePipelineState` 세 번째 인자 제거.
runtime 적응 메커니즘이 각 renderpass에 맞는 PSO를 자동으로 생성·캐싱합니다.

```cpp
// 변경 전
RenderPassInfo renderpass_image = {};
renderpass_image.rt_formats[0] = Format::R10G10B10A2_UNORM;  // 하드코딩 → 오류 원인
renderpass_image.rt_count = 1;
renderpass_image.sample_count = 1;
if (d == DEPTH_TEST_OFF && k == STENCILMODE_DISABLED)
    renderpass_image.ds_format = Format::UNKNOWN;
else
    renderpass_image.ds_format = Format::D32_FLOAT_S8X24_UINT;
device->CreatePipelineState(&desc, &imagePSO[i][j][k][m][d][n], &renderpass_image);

// 변경 후
device->CreatePipelineState(&desc, &imagePSO[i][j][k][m][d][n]);
```

### Image.cpp - debugPSO 생성

`STENCILMODE_DISABLED`는 유지 (실제 버그 수정), `renderpass_info` 제거.

```cpp
// 변경 전
desc.dss = &depthStencilStates[STENCILMODE_DISABLED][STENCILREFMODE_ALL][DEPTH_TEST_OFF];
RenderPassInfo renderpass_debug = {};
renderpass_debug.rt_formats[0] = Format::R10G10B10A2_UNORM;
renderpass_debug.rt_count = 1;
renderpass_debug.ds_format = Format::UNKNOWN;
renderpass_debug.sample_count = 1;
device->CreatePipelineState(&desc, &debugPSO, &renderpass_debug);

// 변경 후
desc.dss = &depthStencilStates[STENCILMODE_DISABLED][STENCILREFMODE_ALL][DEPTH_TEST_OFF];
device->CreatePipelineState(&desc, &debugPSO);
```

### Font.cpp - Font PSO 생성

`renderpass_info` 블록 제거, `CreatePipelineState` 세 번째 인자 제거.

```cpp
// 변경 전
RenderPassInfo renderpass_font = {};
renderpass_font.rt_formats[0] = Format::R10G10B10A2_UNORM;
renderpass_font.rt_count = 1;
renderpass_font.sample_count = 1;
if (d == DEPTH_TEST_OFF)
    renderpass_font.ds_format = Format::UNKNOWN;
else
    renderpass_font.ds_format = Format::D32_FLOAT_S8X24_UINT;
graphics::GetDevice()->CreatePipelineState(&desc, &PSO[d], &renderpass_font);

// 변경 후
graphics::GetDevice()->CreatePipelineState(&desc, &PSO[d]);
```

---

## 변경 파일 요약

| 파일 | 변경 내용 |
|------|-----------|
| `Image.cpp` | `imagePSO` 생성 시 `renderpass_info` 제거 |
| `Image.cpp` | `debugPSO` 생성 시 `renderpass_info` 제거 (`STENCILMODE_DISABLED`는 유지) |
| `Font.cpp` | `fontPSO` 생성 시 `renderpass_info` 제거 |

---

## 핵심 교훈

**`renderpass_info`를 제공하면 runtime PSO 적응이 비활성화된다.**

- PSO가 **단일 RT 포맷**에서만 사용되는 경우 → 생성 시 `renderpass_info` 제공 가능
- PSO가 **여러 RT 포맷**에서 사용되는 경우 → `renderpass_info` 없이 생성, runtime 적응에 맡겨야 함

`imagePSO`와 `fontPSO`는 다양한 renderpass에서 사용되는 범용 PSO이므로,
`renderpass_info` 없이 생성하고 engine의 runtime PSO 적응 메커니즘을 활용하는 것이 올바른 설계입니다.
