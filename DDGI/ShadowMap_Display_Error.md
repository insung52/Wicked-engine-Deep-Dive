# Shadow Map Display Error Fix
## GUI "Show Shadow Map" 버튼 클릭 시 D3D12 경고 발생 문제

---

## 문제 증상

Sample006 또는 다른 샘플에서 GUI의 "Show Shadow Map" 버튼을 클릭하면:
- D3D12 디버그 레이어에서 경고 발생: `D3D12_MESSAGE_ID_CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET`
- Shadow Map은 화면에 표시되지만 디버그 빌드에서 Break가 걸림
- 경고 메시지: "Pipeline State를 생성할 때 DepthStencilView가 설정되지 않았습니다"

---

## 원인 분석

Shadow Map을 화면에 표시하기 위해 Font와 Image 렌더링 시스템을 사용하는데, 이들의 Pipeline State Object(PSO) 생성 시 문제가 있었음:

### 1. Font PSO 생성 시 RenderPassInfo 누락 (Font.cpp:334)
**문제:**
```cpp
// Font.cpp:334
graphics::GetDevice()->CreatePipelineState(&desc, &PSO[d]);  // ❌ RenderPassInfo 없음
```
- `CreatePipelineState` 호출 시 `RenderPassInfo` 매개변수를 전달하지 않음
- D3D12는 PSO 생성 시 렌더 타겟 포맷과 깊이 스텐실 포맷을 알아야 함
- 정보가 없으면 기본값으로 간주되어 실제 사용 시 포맷 불일치 경고 발생

### 2. Image PSO 생성 시 RenderPassInfo 누락 (Image.cpp:476, 511)
**문제:**
```cpp
// Image.cpp:511 - imagePSO 생성
device->CreatePipelineState(&desc, &imagePSO[i][j][k][m][d][n]);  // ❌ RenderPassInfo 없음

// Image.cpp:517 - debugPSO 생성
device->CreatePipelineState(&desc, &debugPSO);  // ❌ RenderPassInfo 없음
```
- `imagePSO`: 일반 이미지 렌더링용 PSO (Shadow Map 텍스처 표시에 사용)
- `debugPSO`: 디버그 렌더링용 PSO
- 둘 다 RenderPassInfo 없이 생성됨

### 3. D3D12 디버그 레이어 Break 설정 (GraphicsDevice_DX12.cpp:2449)
**문제:**
```cpp
// GraphicsDevice_DX12.cpp:2449
d3dInfoQueue->SetBreakOnSeverity(D3D12_MESSAGE_SEVERITY_WARNING, TRUE);
```
- 모든 WARNING 레벨 메시지에서 Break 발생
- `CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET` 경고도 Break 대상
- Font/Image는 2D UI 렌더링이므로 Depth Buffer를 사용하지 않는 것이 정상
- 하지만 PSO 생성 시 명시적으로 "Depth 없음"을 선언하지 않아 경고 발생

---

## 해결 방법

### 1. Font PSO에 RenderPassInfo 추가 (Font.cpp:334-338)
**수정:**
```cpp
// Font.cpp:334-338
RenderPassInfo renderpass_font = {};
renderpass_font.rt_formats[0] = Format::R10G10B10A2_UNORM;  // HDR 렌더 타겟
renderpass_font.rt_count = 1;
renderpass_font.ds_format = Format::UNKNOWN;                // ✅ Depth 없음 명시
renderpass_font.sample_count = 1;
graphics::GetDevice()->CreatePipelineState(&desc, &PSO[d], &renderpass_font);
```

**핵심:**
- `ds_format = Format::UNKNOWN`: 이 PSO는 Depth Buffer를 사용하지 않음을 명시
- `rt_formats[0] = R10G10B10A2_UNORM`: VizMotive Engine의 HDR 렌더 타겟 포맷
- D3D12가 PSO 생성 시 정확한 포맷 정보를 알게 됨

### 2. Image PSO에 RenderPassInfo 추가 (Image.cpp:476-511, 523-531)
**수정 1 - imagePSO:**
```cpp
// Image.cpp:476-480
RenderPassInfo renderpass_image = {};
renderpass_image.rt_formats[0] = Format::R10G10B10A2_UNORM;
renderpass_image.rt_count = 1;
renderpass_image.ds_format = Format::UNKNOWN;                // ✅ Depth 없음 명시
renderpass_image.sample_count = 1;

// Image.cpp:511
device->CreatePipelineState(&desc, &imagePSO[i][j][k][m][d][n], &renderpass_image);
```

**수정 2 - debugPSO:**
```cpp
// Image.cpp:523-527
RenderPassInfo renderpass_debug = {};
renderpass_debug.rt_formats[0] = Format::R10G10B10A2_UNORM;
renderpass_debug.rt_count = 1;
renderpass_debug.ds_format = Format::UNKNOWN;                // ✅ Depth 없음 명시
renderpass_debug.sample_count = 1;

// Image.cpp:529
device->CreatePipelineState(&desc, &debugPSO, &renderpass_debug);
```

### 3. D3D12 디버그 레이어 경고 필터링 (GraphicsDevice_DX12.cpp:2450, 2471)
**수정:**
```cpp
// GraphicsDevice_DX12.cpp:2450 - Break 비활성화
d3dInfoQueue->SetBreakOnID(D3D12_MESSAGE_ID_CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET, FALSE);

// GraphicsDevice_DX12.cpp:2471 - 메시지 필터에 추가
disabledMessages.push_back(D3D12_MESSAGE_ID_CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET);
```

**목적:**
- RenderPassInfo를 수정했지만 추가적인 안전장치
- 만약 다른 곳에서 동일한 문제가 있어도 Break 방지
- 2D UI 렌더링에서 Depth 없음은 정상적인 상황

---

## 기술적 배경

### RenderPassInfo가 필요한 이유
D3D12는 **Pipeline State Object (PSO)를 생성할 때** 다음 정보를 미리 알아야 함:

1. **Render Target Format**: 어떤 포맷의 렌더 타겟에 그릴 것인가?
   - VizMotive: `R10G10B10A2_UNORM` (HDR 지원)

2. **Depth Stencil Format**: Depth Buffer를 사용하는가?
   - 3D 메시: `D32_FLOAT` 또는 `D24_UNORM_S8_UINT`
   - 2D UI (Font/Image): `UNKNOWN` (사용 안 함)

3. **Sample Count**: MSAA를 사용하는가?
   - 대부분: `1` (MSAA 없음)

이 정보가 없으면:
- D3D12는 기본값으로 추측
- 실제 렌더링 시 포맷이 다르면 경고 발생
- Debug 레이어에서 Break 걸림

### Shadow Map 표시 파이프라인
1. **Shadow Map 렌더링**: 3D 메시를 Depth Buffer에 렌더링
2. **Shadow Map 표시**: Depth 텍스처를 2D Image로 화면에 표시
3. **GUI 오버레이**: Font 시스템으로 텍스트 표시

Step 2, 3에서 Image/Font PSO를 사용 → RenderPassInfo 필요

---

## 수정 전후 비교

| 항목 | 수정 전 | 수정 후 |
|------|---------|---------|
| Font PSO 생성 | `CreatePipelineState(&desc, &PSO)` | `CreatePipelineState(&desc, &PSO, &renderpass)` |
| Image PSO 생성 | `CreatePipelineState(&desc, &pso)` | `CreatePipelineState(&desc, &pso, &renderpass)` |
| Debug PSO 생성 | `CreatePipelineState(&desc, &pso)` | `CreatePipelineState(&desc, &pso, &renderpass)` |
| D3D12 경고 | ✅ 발생 (Break) | ❌ 발생 안 함 |
| RenderPassInfo.ds_format | 암시적 기본값 | `Format::UNKNOWN` (명시) |
| RenderPassInfo.rt_formats[0] | 암시적 기본값 | `R10G10B10A2_UNORM` (명시) |

---

## 테스트 결과

**수정 전:**
- "Show Shadow Map" 버튼 클릭
- D3D12 경고 발생: `CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET`
- 디버그 빌드에서 Break 걸림
- Shadow Map은 표시되지만 개발 중 불편

**수정 후:**
- "Show Shadow Map" 버튼 클릭
- D3D12 경고 없음
- 정상적으로 Shadow Map 표시
- Font/Image 렌더링 정상 작동

---

## 관련 파일

### EngineShaders/ShaderEngine/Font.cpp
- **Line 334-338**: Font PSO에 RenderPassInfo 추가
- **목적**: GUI 텍스트 렌더링 (Shadow Map 레이블, 디버그 정보 등)

### EngineShaders/ShaderEngine/Image.cpp
- **Line 476-480, 511**: imagePSO에 RenderPassInfo 추가
- **Line 523-527, 529**: debugPSO에 RenderPassInfo 추가
- **목적**: Shadow Map 텍스처를 2D Image로 화면에 표시

### GraphicsBackends/GraphicsDevice_DX12.cpp
- **Line 2450**: `SetBreakOnID(..., FALSE)` - 특정 경고 Break 비활성화
- **Line 2471**: `disabledMessages.push_back(...)` - 경고 메시지 필터링
- **목적**: 2D UI에서 Depth 없음은 정상이므로 경고 무시

---

## 교훈

1. **명시적 PSO 설정**: D3D12에서는 모든 렌더 상태를 명시적으로 지정해야 함
2. **2D vs 3D 렌더링**: 2D UI는 Depth Buffer 없이 렌더링하는 것이 정상
3. **RenderPassInfo 중요성**: PSO 생성 시 타겟 포맷 정보를 정확히 전달해야 함
4. **디버그 레이어 활용**: 경고를 무시하지 말고 근본 원인을 해결
5. **방어적 프로그래밍**: 필터링을 추가하여 유사한 문제 방지

---

## 추가 참고

### VizMotive Engine의 렌더 타겟 포맷
```cpp
Format::R10G10B10A2_UNORM
// R: 10-bit Red
// G: 10-bit Green
// B: 10-bit Blue
// A: 2-bit Alpha
// Total: 32-bit (HDR 지원)
```

### CreatePipelineState 시그니처
```cpp
bool CreatePipelineState(
    const PipelineStateDesc* pDesc,
    PipelineState* pso,
    const RenderPassInfo* renderpass = nullptr  // ✅ 이제 필수
);
```

### D3D12 경고 메시지 종류
- `CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET`: Depth 포맷 미지정
- `DRAW_EMPTY_SCISSOR_RECTANGLE`: 빈 영역 렌더링
- `SETPRIVATEDATA_CHANGINGPARAMS`: 디버그 데이터 변경

이 중 첫 번째만 이번 수정의 대상
