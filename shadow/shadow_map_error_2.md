# 문제 상황

show shadow map 버튼 클릭 시 프로그램 정지 및 에러 발생

## 예외 발생 지점

GraphicsDevice_DX12.cpp:2181
```c
HRESULT hr = device->CreatePipelineState(&streamDesc, PPV_ARGS(newpso));

0x00007FFD08ED782A(KernelBase.dll)에(Sample06.exe의) 처리되지 않은 예외가 있습니다. 0x0000087A(매개 변수: 0x0000000000000002, 0x0000001F098F3D88, 0x0000001F098F4B30).
```

## 출력

```c
D3D12 WARNING: ID3D12Device::CreateGraphicsPipelineState: The depth stencil unit or pixel shader expects a Depth Stencil View, but the generic program indicates that none will be bound. This is OK, as reads of an unbound Depth Stencil View are defined to return 0; and writes are discarded. It is also possible the developer knows the data will not be used anyway. This is only a problem if the developer actually intended to bind a Depth Stencil View here. [ STATE_CREATION WARNING #680: CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET]
D3D12: **BREAK** enabled for the previous message, which was: [ WARNING STATE_CREATION #680: CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET ]
예외 발생(0x00007FFD08ED782A(KernelBase.dll), Sample06.exe): 0x0000087A(매개 변수: 0x0000000000000002, 0x000000D4EE3D3D38, 0x000000D4EE3D4AE0).
0x00007FFD08ED782A(KernelBase.dll)에(Sample06.exe의) 처리되지 않은 예외가 있습니다. 0x0000087A(매개 변수: 0x0000000000000002, 0x000000D4EE3D3D38, 0x000000D4EE3D4AE0).
```

## 호출 스택

```c
KernelBase.dll!00007ffd08ed782a()
DXGIDebug.dll!00007ffce70f4f4f()
d3d12SDKLayers.dll!00007ffc564a52c8()
d3d12SDKLayers.dll!00007ffc563eda36()
d3d12SDKLayers.dll!00007ffc563d0787()
d3d12SDKLayers.dll!00007ffc563ee120()
D3D12Core.dll!00007ffc9e9e7843()
D3D12Core.dll!00007ffc9ea66a88()
D3D12Core.dll!00007ffc9ea3b721()
D3D12Core.dll!00007ffc9e9f4632()
D3D12Core.dll!00007ffc9e927c31()
D3D12Core.dll!00007ffc9e98dcf5()
d3d12SDKLayers.dll!00007ffc563db1b9()
GBackendDX12.dll!vz::graphics::GraphicsDevice_DX12::pso_validate(vz::graphics::CommandList cmd) 줄 2181
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\GraphicsBackends\GraphicsDevice_DX12.cpp(2181)
GBackendDX12.dll!vz::graphics::GraphicsDevice_DX12::predraw(vz::graphics::CommandList cmd) 줄 2207
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\GraphicsBackends\GraphicsDevice_DX12.cpp(2207)
GBackendDX12.dll!vz::graphics::GraphicsDevice_DX12::Draw(unsigned int vertexCount, unsigned int startVertexLocation, vz::graphics::CommandList cmd) 줄 6902
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\GraphicsBackends\GraphicsDevice_DX12.cpp(6902)
ShaderEngine.dll!vz::image::Draw(const vz::graphics::Texture * texture, const vz::image::Params & params, vz::graphics::CommandList cmd) 줄 454
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\EngineShaders\ShaderEngine\Image.cpp(454)
ShaderEngine.dll!vz::renderer::GRenderPath3DDetails::Compose(vz::graphics::CommandList cmd) 줄 1310
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\EngineShaders\ShaderEngine\Renderer.cpp(1310)
ShaderEngine.dll!vz::renderer::GRenderPath3DDetails::Render(const float dt) 줄 1433
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\EngineShaders\ShaderEngine\Renderer.cpp(1433)
VizEngined.dll!vz::RenderPath3D::Render(const float dt) 줄 249
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\EngineCore\Common\RenderPath3D.cpp(249)
VizEngined.dll!vzm::VzRenderer::Render(const unsigned __int64 vidScene, const unsigned __int64 vidCam, const float dt) 줄 267
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\EngineCore\HighAPIs\VzRenderer.cpp(267)
Sample06.exe!vzm::VzRenderer::Render(const vzm::VzScene * scene, const vzm::VzCamera * camera, const float dt) 줄 60
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\Install\vzm2\VzRenderer.h(60)
Sample06.exe!main(int __formal, char * * __formal) 줄 428
	위치: C:\graphics\vizmotive\origin\VizMotive-Engine\Examples\Sample006\sample06.cpp(428)
[외부 코드]
```

---

# 문제 해결 과정

## 1. 근본 원인 분석

### D3D12 에러 메시지 분석
```
D3D12 WARNING #680: CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET
The depth stencil unit or pixel shader expects a Depth Stencil View,
but the generic program indicates that none will be bound.
```

**핵심:** PSO의 DepthStencilState 설정과 실제 depth buffer 바인딩 정보가 불일치

#### PSO : (Pipeline State Object)

GPU 렌더링 파이프라인의 “상태 묶음”을 하나의 객체로 미리 만들어 둔 것

- 미리 컴파일 되어, 한번 만들면 불변(Immutable)
- 멀티스레드 친화적(여러 cpu 스레드에서 pso 생성 가능)
- physics, postprocessing, particle 등에 사용하는 compute PSO 도 존재함 


### 문제 발생 시나리오

```
초기화 시:
  Image.cpp LoadShaders()
    → CreatePipelineState(&desc, &debugPSO)  // ❌ renderpass_info 없음
    → PSO Hash = (pso pointer, 0)

런타임 (show shadow map 버튼 클릭):
  Renderer.cpp Compose()
    → RenderPassBegin(RenderTarget만, DepthStencil 없음)
    → commandlist.renderpass_info.ds_format = Format::UNKNOWN

  Image.cpp Draw()
    → BindPipelineState(&debugPSO)

  GraphicsDevice_DX12.cpp predraw()
    → pso_validate()
    → New Hash = (pso pointer, renderpass_info.get_hash())  // ← Hash 불일치!
    → CreatePipelineState() 재시도
    → DepthStencilState는 stencil_enable=true 인데 ds_format=UNKNOWN
    → ⚠️ D3D12 WARNING #680 발생 및 프로그램 중단
```

---

## 2. 디버깅 과정

### 시도 1: Image.cpp 및 Font.cpp PSO 생성 시 RenderPassInfo 추가

#### RenderPassInfo : 이번 패스에서 어떤 buffer 들을, 어떤 규칙으로, 어떤 순서로 사용하겠다 라는 렌더링 계약서

**변경 내용:**
```cpp
// Image.cpp:517-523 (debugPSO)
RenderPassInfo renderpass_debug = {};
renderpass_debug.rt_formats[0] = Format::R10G10B10A2_UNORM;
renderpass_debug.rt_count = 1;
renderpass_debug.ds_format = Format::UNKNOWN;  // Debug rendering은 2D이므로 depth 불필요
renderpass_debug.sample_count = 1;
device->CreatePipelineState(&desc, &debugPSO, &renderpass_debug);

// Font.cpp:325-340
for (int d = 0; d < DEPTH_TEST_MODE_COUNT; ++d)
{
    RenderPassInfo renderpass_font = {};
    renderpass_font.rt_formats[0] = Format::R10G10B10A2_UNORM;
    renderpass_font.rt_count = 1;
    renderpass_font.ds_format = Format::UNKNOWN;  // 하나의 RenderPassInfo만 생성
    renderpass_font.sample_count = 1;

    device->CreatePipelineState(&desc, &PSO[d], &renderpass_font);
}
```

**결과:** ❌ Font.cpp:340에서 에러 발생 (F5 실행 즉시)

**호출 스택:**
```
GraphicsDevice_DX12.cpp:4304 (CreatePipelineState)
Font.cpp:340 (LoadShaders)
```

**에러 원인 분석:**
Font는 2개의 PSO를 생성:
- `PSO[DEPTH_TEST_OFF]`: `depth_enable = false` ✅
- `PSO[DEPTH_TEST_ON]`: `depth_enable = true` ❌

`DEPTH_TEST_ON` PSO는 depth buffer가 필요한데 `ds_format = UNKNOWN`으로 설정 → 불일치!

---

### 시도 2: Font.cpp Depth 모드별로 다른 RenderPassInfo 생성

**변경 내용:**
```cpp
// Font.cpp:325-351
for (int d = 0; d < DEPTH_TEST_MODE_COUNT; ++d)
{
    RenderPassInfo renderpass_font = {};
    renderpass_font.rt_formats[0] = Format::R10G10B10A2_UNORM;
    renderpass_font.rt_count = 1;
    renderpass_font.sample_count = 1;

    if (d == DEPTH_TEST_OFF)
    {
        renderpass_font.ds_format = Format::UNKNOWN;  // 2D UI - depth 불필요
    }
    else
    {
        renderpass_font.ds_format = Format::D32_FLOAT;  // 3D debug text - depth 필요
    }

    device->CreatePipelineState(&desc, &PSO[d], &renderpass_font);
}
```

**결과:** ✅ Font 에러 해결, 하지만 다음 에러 발견

---

### 시도 3: Image.cpp imagePSO 배열 처리

**문제 발견:**
Image.cpp Line 505의 `imagePSO[i][j][k][m][d][n]` 배열도 RenderPassInfo 없이 생성됨
- 배열 차원 중 `[d]`는 depth test mode
- 배열 차원 중 `[k]`는 stencil mode

**변경 내용:**
```cpp
// Image.cpp:487-524
for (int m = 0; m < STENCILREFMODE_COUNT; ++m)
{
    for (int d = 0; d < DEPTH_TEST_MODE_COUNT; ++d)
    {
        desc.dss = &depthStencilStates[k][m][d];

        RenderPassInfo renderpass_image = {};
        renderpass_image.rt_formats[0] = Format::R10G10B10A2_UNORM;
        renderpass_image.rt_count = 1;
        renderpass_image.sample_count = 1;

        // DEPTH_TEST_OFF: no depth buffer
        // DEPTH_TEST_ON: needs depth buffer
        if (d == DEPTH_TEST_OFF)
        {
            renderpass_image.ds_format = Format::UNKNOWN;
        }
        else
        {
            renderpass_image.ds_format = Format::D32_FLOAT;
        }

        for (int n = 0; n < STRIP_MODE_COUNT; ++n)
        {
            // ...
            device->CreatePipelineState(&desc, &imagePSO[i][j][k][m][d][n], &renderpass_image);
        }
    }
}
```

**결과:** ❌ Image.cpp:521에서 에러 발생 (F5 실행 즉시)

**에러 원인 분석:**
Stencil 모드를 고려하지 않았음!

DepthStencilState 생성 코드 확인 (Image.cpp:566-636):
```cpp
for (int d = 0; d < DEPTH_TEST_MODE_COUNT; ++d)
{
    DepthStencilState dsd;

    // Depth 설정
    switch (d)
    {
        case DEPTH_TEST_OFF:
            dsd.depth_enable = false;
            break;
        case DEPTH_TEST_ON:
            dsd.depth_enable = true;
            break;
    }

    // STENCILMODE_DISABLED만 stencil_enable = false
    dsd.stencil_enable = false;
    depthStencilStates[STENCILMODE_DISABLED][i][d] = dsd;

    // 나머지 STENCILMODE는 모두 stencil_enable = true
    dsd.stencil_enable = true;
    depthStencilStates[STENCILMODE_EQUAL][i][d] = dsd;
    depthStencilStates[STENCILMODE_LESS][i][d] = dsd;
    // ... (총 7개의 stencil 모드)
}
```

**문제:**
- `DEPTH_TEST_OFF` + `STENCILMODE_EQUAL`(stencil_enable=true) PSO
- RenderPassInfo의 `ds_format = Format::UNKNOWN`
- → Stencil이 활성화되었는데 depth/stencil buffer format이 없음!

---

### 시도 4: Stencil 모드 고려한 Format 선택

**Depth/Stencil Format 확인:**
```cpp
// Renderer.h:39 - VizMotive 메인 depth buffer 포맷
constexpr Format FORMAT_depthbufferMain = Format::D32_FLOAT_S8X24_UINT;
```

**핵심 인사이트:**
- `D32_FLOAT_S8X24_UINT`: 32-bit depth + 8-bit stencil
- Depth만 필요: `D32_FLOAT`도 가능하지만, 메인 포맷과 일치시키는 것이 안전
- Depth와 Stencil 둘 다 필요: `D32_FLOAT_S8X24_UINT` 필수

**변경 내용:**
```cpp
// Image.cpp:498-507
// Only if both depth and stencil are disabled, use UNKNOWN
// Otherwise, use depth+stencil format
if (d == DEPTH_TEST_OFF && k == STENCILMODE_DISABLED)
{
    renderpass_image.ds_format = Format::UNKNOWN;  // No depth, no stencil
}
else
{
    renderpass_image.ds_format = Format::D32_FLOAT_S8X24_UINT;  // Depth and/or stencil
}
```

**Font.cpp도 동일하게 수정:**
```cpp
// Font.cpp:340
renderpass_font.ds_format = Format::D32_FLOAT_S8X24_UINT;  // D32_FLOAT → D32_FLOAT_S8X24_UINT
```

**결과:** ✅ 초기화 에러 해결!

---

### 시도 5: Show Shadow Map 버튼 클릭 테스트

**결과:** ❌ Show Shadow Map 버튼 클릭 시 다시 에러 발생!

**에러 메시지:**
```
D3D12 ERROR #615: DEPTH_STENCIL_FORMAT_MISMATCH_PIPELINE_STATE
pipeline state = D32_FLOAT_S8X24_UINT, DSV = nullptr
```

**호출 스택:**
```
GraphicsDevice_DX12.cpp:6904 (Draw)
Image.cpp:454 (Draw)
Renderer.cpp:1310 (Compose)
```

**에러 원인 분석:**

debugPSO 설정 확인:
```cpp
// Image.cpp:531 (수정 전)
desc.dss = &depthStencilStates[STENCILMODE_ALWAYS][STENCILREFMODE_ALL][DEPTH_TEST_OFF];

// Image.cpp:537 (수정 전)
renderpass_debug.ds_format = Format::D32_FLOAT_S8X24_UINT;
```

Compose() RenderPass 확인:
```cpp
// Renderer.cpp:1421-1433
graphics::RenderPassImage rp[] = {
    graphics::RenderPassImage::RenderTarget(&rtRenderFinal_),  // RenderTarget만
    // ← DepthStencil 없음!
};
device->RenderPassBegin(rp, arraysize(rp), cmd);
Compose(cmd);  // debugPSO 사용
```

**문제:**
- debugPSO는 `D32_FLOAT_S8X24_UINT` 포맷으로 생성 (depth/stencil 필요)
- 실제 Compose()는 depth buffer 없이 실행 (RenderTarget만)
- Runtime 불일치 → 에러!

**해결 방법:**
debugPSO는 **실제로 depth/stencil buffer 없이 사용**되므로:
1. DepthStencilState를 `STENCILMODE_DISABLED`로 변경
2. RenderPassInfo의 `ds_format`을 `UNKNOWN`으로 변경

---

### 시도 6: debugPSO 수정 (최종 해결)

**변경 내용:**
```cpp
// Image.cpp:531, 537
desc.dss = &depthStencilStates[STENCILMODE_DISABLED][STENCILREFMODE_ALL][DEPTH_TEST_OFF];  // STENCILMODE_ALWAYS → STENCILMODE_DISABLED
renderpass_debug.ds_format = Format::UNKNOWN;  // D32_FLOAT_S8X24_UINT → UNKNOWN
```

**결과:** ✅ 완벽하게 작동!

---

## 3. 최종 해결 방법 정리

### 문제 요약

**근본 원인:** PSO 생성 시 RenderPassInfo를 제공하지 않아, PSO의 DepthStencilState 설정과 실제 런타임 depth buffer 바인딩 정보가 불일치

**핵심 문제 3가지:**
1. Image/Font PSO 배열: 초기화 시 RenderPassInfo 없이 생성
2. debugPSO의 잘못된 DepthStencilState: `STENCILMODE_ALWAYS` 사용 (실제로는 depth buffer 없이 사용됨)
3. Stencil 모드 미고려: Depth 모드만 체크하고 Stencil 활성화 여부를 확인하지 않음

---

### 변경된 파일 및 내용

#### 1) Image.cpp - imagePSO 배열 (Line 493-521)

**해결한 문제:**
- 다양한 렌더링 상황에서 사용되는 Image PSO 배열이 RenderPassInfo 없이 생성됨
- Stencil 모드를 고려하지 않아, stencil이 활성화된 PSO도 `ds_format = UNKNOWN`으로 생성되어 에러 발생

**imagePSO 배열 구조:**
```cpp
imagePSO[sprite][blend][stencil_mode][stencil_ref][depth_mode][strip_mode]
         [i]    [j]    [k]          [m]          [d]        [n]

- [k] stencil_mode: DISABLED, EQUAL, LESS, LESSEQUAL, GREATER, GREATEREQUAL, NOT, ALWAYS (8개)
- [d] depth_mode: DEPTH_TEST_OFF, DEPTH_TEST_ON (2개)
```

**변경 전:**
```cpp
// Line 505 (원본)
device->CreatePipelineState(&desc, &imagePSO[i][j][k][m][d][n]);  // ❌ RenderPassInfo 없음
```

**변경 후:**
```cpp
// Line 489-521
for (int m = 0; m < STENCILREFMODE_COUNT; ++m)
{
    for (int d = 0; d < DEPTH_TEST_MODE_COUNT; ++d)
    {
        desc.dss = &depthStencilStates[k][m][d];

        RenderPassInfo renderpass_image = {};
        renderpass_image.rt_formats[0] = Format::R10G10B10A2_UNORM;  // HDR render target
        renderpass_image.rt_count = 1;
        renderpass_image.sample_count = 1;

        // 핵심 로직: Depth AND Stencil 둘 다 확인!
        if (d == DEPTH_TEST_OFF && k == STENCILMODE_DISABLED)
        {
            // 2D UI 렌더링 등 - depth/stencil 모두 불필요
            renderpass_image.ds_format = Format::UNKNOWN;
        }
        else
        {
            // 3D 렌더링 OR stencil 사용 - depth+stencil buffer 필요
            renderpass_image.ds_format = Format::D32_FLOAT_S8X24_UINT;
        }

        for (int n = 0; n < STRIP_MODE_COUNT; ++n)
        {
            // ... primitive topology 설정
            device->CreatePipelineState(&desc, &imagePSO[i][j][k][m][d][n], &renderpass_image);
        }
    }
}
```

**왜 이렇게 해결했는가:**
- `depthStencilStates[k][m][d]` 배열에서:
  - `k == STENCILMODE_DISABLED`: `stencil_enable = false`
  - 나머지 7개 모드: `stencil_enable = true`
- Stencil이 활성화되면 반드시 depth/stencil buffer format 명시 필요
- 따라서 `d`(depth mode)와 `k`(stencil mode) **둘 다** 체크해야 함

**실제 사용 사례:**
- `imagePSO[0][OPAQUE][STENCILMODE_DISABLED][ALL][DEPTH_TEST_OFF][STRIP_ON]`: 2D 이미지 (depth/stencil 없음)
- `imagePSO[0][ALPHA][STENCILMODE_EQUAL][ENGINE][DEPTH_TEST_ON][STRIP_OFF]`: Stencil 마스킹된 3D 오브젝트

---

#### 2) Image.cpp - debugPSO (Line 531, 534-539)

**해결한 문제:**
- **가장 치명적인 버그!** "show shadow map" 버튼 클릭 시 에러의 직접적 원인
- debugPSO가 `STENCILMODE_ALWAYS` (stencil 활성화) + `ds_format = D32_FLOAT_S8X24_UINT`로 생성
- 실제로는 Compose()에서 depth buffer **없이** 사용됨 → Runtime format 불일치

**debugPSO 사용 위치:**
```cpp
// Renderer.cpp:1305-1310
fx.enableDebugTest();  // ← isDebugTestEnabled() = true
image::Draw(&shadowMapAtlas, fx_sm, cmd);  // ← debugPSO 바인딩

// Image.cpp:428-431
if (params.isDebugTestEnabled())
{
    device->BindPipelineState(&debugPSO, cmd);  // ← Show shadow map 시 호출
}
```

**Compose() RenderPass 구조:**
```cpp
// Renderer.cpp:1421-1433
RenderPassImage rp[] = {
    RenderPassImage::RenderTarget(&rtRenderFinal_),  // ← RenderTarget만 있음
    // DepthStencil 없음!
};
device->RenderPassBegin(rp, arraysize(rp), cmd);
Compose(cmd);  // ← debugPSO가 여기서 사용됨
```

**변경 전:**
```cpp
// Line 531 (원본)
desc.dss = &depthStencilStates[STENCILMODE_ALWAYS][STENCILREFMODE_ALL][DEPTH_TEST_OFF];
// ↑ stencil_enable = true

// Line 517 (원본)
device->CreatePipelineState(&desc, &debugPSO);  // ❌ RenderPassInfo 없음
```

**변경 후:**
```cpp
// Line 531 (수정)
desc.dss = &depthStencilStates[STENCILMODE_DISABLED][STENCILREFMODE_ALL][DEPTH_TEST_OFF];
// ↑ depth_enable = false, stencil_enable = false

// Line 534-539 (추가)
RenderPassInfo renderpass_debug = {};
renderpass_debug.rt_formats[0] = Format::R10G10B10A2_UNORM;  // HDR render target
renderpass_debug.rt_count = 1;
renderpass_debug.ds_format = Format::UNKNOWN;  // ✅ Compose()는 depth buffer 없음!
renderpass_debug.sample_count = 1;

device->CreatePipelineState(&desc, &debugPSO, &renderpass_debug);
```

**핵심: 2가지 변경 모두 필수!**
1. `STENCILMODE_ALWAYS` → `STENCILMODE_DISABLED`: DepthStencilState를 실제 사용 환경에 맞게 변경
2. `ds_format = UNKNOWN`: Compose() RenderPass에 depth buffer가 없음을 명시

**왜 원래 코드가 STENCILMODE_ALWAYS를 사용했는가?**
- 추측: 일반적인 렌더링에서 stencil을 사용하려고 했을 수 있음
- 실제: debugPSO는 **오직 Compose()에서만** 사용되고, Compose()는 depth buffer 없는 2D post-processing 단계

---

#### 3) Font.cpp - PSO 배열 (Line 327-350)

**해결한 문제:**
- Font PSO가 depth 모드별로 다른 RenderPassInfo 필요한데 하나만 제공
- `DEPTH_TEST_ON` PSO는 3D 공간 디버그 텍스트 렌더링에 사용되어 depth test 필요

**Font PSO 사용 사례:**
```cpp
// Renderer.cpp:1313-1319 (2D UI 텍스트 - DEPTH_TEST_OFF)
font::Params fx_font;
fx_font.disableDepthTest();  // ← DEPTH_TEST_OFF
font::Draw("Shadow Map", fx_font, cmd);

// DebugRenderer.cpp:202-205 (3D 공간 텍스트 - DEPTH_TEST_ON)
if (x.params.flags & DebugTextParams::DEPTH_TEST)
{
    params.enableDepthTest();  // ← DEPTH_TEST_ON
}
font::Draw(x.params.text, params, cmd);
```

**변경 전:**
```cpp
// Line 334 (원본)
device->CreatePipelineState(&desc, &PSO[d]);  // ❌ RenderPassInfo 없음
```

**변경 후:**
```cpp
// Line 325-350
for (int d = 0; d < DEPTH_TEST_MODE_COUNT; ++d)
{
    RenderPassInfo renderpass_font = {};
    renderpass_font.rt_formats[0] = Format::R10G10B10A2_UNORM;
    renderpass_font.rt_count = 1;
    renderpass_font.sample_count = 1;

    // Depth 모드별 format 선택
    if (d == DEPTH_TEST_OFF)
    {
        // 2D UI 텍스트 (화면에 직접 그리기)
        renderpass_font.ds_format = Format::UNKNOWN;
    }
    else  // d == DEPTH_TEST_ON
    {
        // 3D 공간 텍스트 (depth test로 가려짐 확인)
        renderpass_font.ds_format = Format::D32_FLOAT_S8X24_UINT;
    }

    PipelineStateDesc desc;
    desc.vs = &vertexShader;
    desc.ps = &pixelShader;
    desc.bs = &blendState;
    desc.dss = &depthStencilStates[d];  // ← depth 모드에 따라 다른 state
    desc.rs = &rasterizerState;
    desc.pt = PrimitiveTopology::TRIANGLESTRIP;

    graphics::GetDevice()->CreatePipelineState(&desc, &PSO[d], &renderpass_font);
}
```

**Font DepthStencilState 구조:**
```cpp
// Font.cpp:393-401
depthStencilStates[DEPTH_TEST_OFF]:
  - depth_enable = false
  - stencil_enable = false  // ← Font는 stencil 미사용

depthStencilStates[DEPTH_TEST_ON]:
  - depth_enable = true
  - stencil_enable = false
  - depth_func = GREATER_EQUAL
```

**왜 D32_FLOAT_S8X24_UINT을 사용하는가?**
- Font는 stencil을 사용하지 않지만 (`stencil_enable = false`)
- VizMotive의 메인 depth buffer 포맷이 `D32_FLOAT_S8X24_UINT`
- PSO format은 실제 바인딩될 depth buffer의 **물리적 포맷**과 일치해야 함
- Stencil 비활성화는 소프트웨어 설정, 포맷은 하드웨어 리소스

---

### 변경 사항 요약표

| 파일 | PSO | 원인 | 해결 | 핵심 변화 |
|------|-----|------|------|-----------|
| Image.cpp | imagePSO | Stencil 모드 미고려 | `d && k` 조건 체크 | Stencil 활성화 여부 확인 |
| Image.cpp | debugPSO | 잘못된 DepthStencilState | STENCILMODE_DISABLED + UNKNOWN | 실제 사용 환경 반영 |
| Font.cpp | PSO[d] | Depth 모드별 format 미분리 | Loop 내부에서 조건별 생성 | DEPTH_TEST_ON → D32_FLOAT_S8X24_UINT |

---

## 4. 핵심 개념 정리

### PSO (Pipeline State Object) 생성 원칙

D3D12에서 PSO를 생성할 때는 다음 정보가 **일치**해야 함:

1. **DepthStencilState 설정**
   - `depth_enable`: depth test 활성화 여부
   - `stencil_enable`: stencil test 활성화 여부

2. **RenderPassInfo의 depth format**
   - `Format::UNKNOWN`: Depth/Stencil buffer 없음
   - `Format::D32_FLOAT`: 32-bit depth만
   - `Format::D32_FLOAT_S8X24_UINT`: 32-bit depth + 8-bit stencil

### Format 선택 가이드라인

| Depth Enable | Stencil Enable | Format 선택 |
|--------------|----------------|-------------|
| false | false | `Format::UNKNOWN` |
| true | false | `Format::D32_FLOAT_S8X24_UINT` |
| false | true | `Format::D32_FLOAT_S8X24_UINT` |
| true | true | `Format::D32_FLOAT_S8X24_UINT` |

**이유:** VizMotive의 메인 depth buffer가 `D32_FLOAT_S8X24_UINT` 포맷이므로, depth나 stencil 중 하나라도 사용하면 해당 포맷을 명시해야 함.

### Runtime PSO Validation 메커니즘

VizMotive Engine은 PSO를 바인딩할 때마다 다음을 검증:

```
PSO Hash = (PSO Pointer, Current RenderPass Hash)
```

- 초기화 시 RenderPassInfo 없이 생성 → Hash = (ptr, 0)
- Runtime에 RenderPass 설정 → Hash = (ptr, renderpass_hash)
- Hash 불일치 → PSO 재생성 시도
- Format 불일치 → D3D12 경고/에러 발생

**해결:** 초기화 시부터 올바른 RenderPassInfo 제공 → Runtime 재생성 불필요

---

## 5. 교훈

### 1. D3D12 PSO는 초기화 시 완전한 정보 필요
RenderPassInfo 없이 PSO를 생성하면 runtime 오버헤드 발생 및 에러 가능성 증가

### 2. DepthStencilState와 RenderPass의 일치성 중요
PSO의 depth/stencil 설정이 실제 사용 환경과 정확히 일치해야 함

### 3. 다차원 PSO 배열 처리 시 모든 차원 고려 필요
Image.cpp의 `imagePSO[i][j][k][m][d][n]` 배열에서:
- `[d]` (depth mode) 뿐만 아니라
- `[k]` (stencil mode)도 RenderPassInfo 결정에 영향

### 4. 실제 사용 환경 분석이 핵심
debugPSO의 경우, 코드만 보면 stencil을 사용할 것 같지만, 실제 Compose()에서는 depth buffer 없이 사용됨을 파악해야 함

---

## 6. 참고: D3D12 Warning/Error 코드

- **#680: CREATEGRAPHICSPIPELINESTATE_DEPTHSTENCILVIEW_NOT_SET**
  - PSO가 depth/stencil을 예상하지만 RenderPassInfo에 명시되지 않음

- **#615: DEPTH_STENCIL_FORMAT_MISMATCH_PIPELINE_STATE**
  - PSO의 depth format과 실제 바인딩된 DSV의 format 불일치

---

**최종 해결 날짜:** 2026-01-07

**브랜치:** Fix-shadowmap-error

**커밋 예정:** Image.cpp, Font.cpp 수정 사항


