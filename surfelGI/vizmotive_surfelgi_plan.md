# VizMotive Engine Surfel GI 통합 계획

> Surfel GI 이론 및 Wicked Engine 원본 분석: [surfelGI.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/surfelGI/surfelGI.md)
>
> 참고 — VXGI 통합 구현 기록: [vizmotive_vxgi_implementation.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/voxelGI/vizmotive_vxgi_implementation.md)

VizMotive 엔진의 `Sample014` 에 Wicked Engine 9.2 의 Surfel GI 를 그대로 이식하기 위한 작업 계획서. **DDGI 가 이미 동일한 패턴 (TLAS RT + indirect dispatch + ping-pong buffers) 으로 통합돼 있어 가장 좋은 레퍼런스다.**

---

## 0. 사전 조사 결과 — 기존 스캐폴딩 현황

VizMotive 에는 **부분적인 SurfelGI 스캐폴딩이 이미 존재**한다. 처음부터 모든 것을 작성할 필요는 없다.

| 항목 | 위치 | 상태 |
|------|------|------|
| `ShaderInterop_SurfelGI.h` | `EngineShaders/Shaders/` | **부분 존재** — 옛 Wicked 포맷 (Surfel 에 SH radiance 없음) → Wicked 9.2 포맷으로 업데이트 |
| `SH_Lite.hlsli` (L1/L2 SH 라이브러리) | `EngineShaders/Shaders/CommonHF/` | ✅ 그대로 사용 (DDGI 가 이미 사용) |
| `raytracingHF.hlsli` (RTAPI 지원) | `EngineShaders/Shaders/CommonHF/` | ✅ 사용 가능 |
| `OPTION_BIT_SURFELGI_ENABLED` 플래그 | `ShaderInterop_Renderer.h:35` | ✅ 정의됨 |
| `texture_surfelgi_index` (Camera CB) | `ShaderInterop_Renderer.h:1413` | ✅ 정의됨 |
| `isSurfelGIDebugEnabled` (renderer 전역) | `ShaderEngine.cpp:46` | ✅ 정의됨 (UI 만 비어있음) |
| Shading lookup 코드 | `shadingHF.hlsli:355-359` | ✅ 이미 작성됨 |
| `Renderer.cpp` 의 SurfelGI 호출 위치 주석 | `Renderer.cpp:1507, 1680` | 주석 처리됨 — 풀어야 함 |
| **`ShaderMeshInstance::uid` 64bit** | `ShaderInterop_Renderer.h:175` | ✅ `uint64_t` — Wicked 9.2 포맷 그대로 사용 가능 |
| `upsample_bilateral_float4CS.hlsl` | `EngineShaders/Shaders/CS/` | ✅ 셰이더 존재 (Wicked 와 동일) |
| `Postprocess_Upsample_Bilateral` C++ 래퍼 | `Renderer.cpp:796` | ❌ 주석 처리됨 — 구현 필요 |
| **CS 셰이더 8종** | — | ❌ 전무 |
| **`CSTYPE_SURFEL_*` enum** | `ShaderLoader.h` | ❌ 추가 필요 |
| **`GSceneDetails::SurfelGI` 리소스** | `Scene_Detail.h` | ❌ 추가 필요 (DDGI 옆에) |
| **`Update_SurfelGI()` / `SurfelGI_Coverage()` 함수** | `GI_Detail.cpp`, `Renderer.cpp` | ❌ 추가 필요 |
| **`isSurfelGIEnabled` 플래그** | `Renderer.h`, `ShaderEngine.cpp` | ❌ 추가 필요 |
| **Sample14 UI** | `sample14.cpp` | ❌ 추가 필요 |

**결론**: 인프라는 절반 이상 갖춰져 있고, 대부분 "DDGI 패턴을 복제"하는 작업이다.

---

## 1. 사용자 결정 사항 (확정)

| 항목 | 결정 |
|------|------|
| `Surfel` 구조 | **Wicked 9.2 와 동일** (SH L1 RGB radiance 포함) |
| Instance UID | `uint64_t` 사용 (VizMotive `ShaderMeshInstance::uid` 가 이미 `uint64_t`) |
| Raytrace 셰이더 | **Wicked 와 동일하게 RTAPI 와 SW BVH 양쪽 모두 컴파일** (`surfel_raytraceCS_rtapi.hlsl` + `surfel_raytraceCS.hlsl`). RTAPI 지원 GPU 면 전자, 아니면 후자를 로드 |
| Half→Full upsample | **Wicked 와 동일하게 `Postprocess_Upsample_Bilateral`** (lineardepth 기반 bilateral 업샘플) |
| 디버그 시각화 | **포함**. Wicked 에서는 별도 셰이더가 아니라 `surfel_coverageCS.hlsl` 내부의 push constant + `debugUAV` 출력으로 처리 |

---

## 2. 작업 범위 및 우선순위

### Phase 1 — 인프라 구축 (선행)
1. `ShaderInterop_SurfelGI.h` 를 Wicked 9.2 포맷으로 업데이트 (Surfel 구조에 SH L1 RGB radiance 필드 추가, `uint64_t uid`)
2. `Scene_Detail.h` 에 `GSceneDetails::SurfelGI` 리소스 구조체 추가
3. `Scene` Update 단계에서 GPU 버퍼/텍스처 생성 코드 (DDGI 옆에) — Wicked 의 `wiScene.cpp:485-583` 그대로
4. Renderer 전역 플래그 (`isSurfelGIEnabled`) + Wicked 의 `GetSurfelGIDebugEnabled()` 와 동일 시그니처의 `SURFEL_DEBUG` enum getter/setter
5. `OPTION_BIT_SURFELGI_ENABLED` 플래그 frame CB 에 실제로 set 하도록 연결
6. `Postprocess_Upsample_Bilateral` C++ 래퍼 구현 (Wicked 의 `wiRenderer.cpp` 동명 함수 그대로 포팅)

### Phase 2 — 셰이더 포팅 (메인)
1. **8개 CS 셰이더** Wicked 9.2 에서 그대로 복사:
   - `surfel_coverageCS.hlsl` (디버그 시각화 내장)
   - `surfel_indirectprepareCS.hlsl`
   - `surfel_updateCS.hlsl`
   - `surfel_gridoffsetsCS.hlsl`
   - `surfel_binningCS.hlsl`
   - `surfel_raytraceCS.hlsl` (SW BVH 버전)
   - `surfel_raytraceCS_rtapi.hlsl` (RTAPI 버전, SM_6_5)
   - `surfel_integrateCS.hlsl`
2. `ShaderLoader.h/cpp` 에 `CSTYPE_SURFEL_*` 7개 추가 (raytrace 는 RT/SW 양쪽 빌드해 같은 enum 에 매핑)
3. `vcxitems(.filters)` 에 8개 셰이더 등록
4. 각 셰이더 내부의 G-buffer 슬롯 / surface load API 를 VizMotive 에 맞게 동기화 (DDGI raytrace 셰이더가 레퍼런스)

### Phase 3 — 파이프라인 통합 (Wicked 의 호출 순서를 그대로)

Wicked 는 SurfelGI 를 **두 단계로 나눠 호출**한다:

```
[Job: scene_update / async]              [Job: maincamera_compute]
SurfelGI(scene)                          SurfelGI_Coverage(res, scene, ...)
├─ Grid Reset (ClearUAV)                 ├─ Coverage CS (G-buffer 의존)
├─ Update CS (alive→alive)               │   - 새 surfel 스폰
├─ GridOffsets CS (prefix sum)           │   - GI 결과 result_halfres 출력
├─ Binning CS                            │   - debugUAV 시각화 출력
├─ Raytrace CS (RTAPI 또는 SW)            ├─ IndirectPrepare CS (다음 프레임용)
└─ Integrate CS (SH+Moments)             └─ Postprocess_Upsample_Bilateral
                                              (result_halfres → result, lineardepth 기반)
```

호출 위치 (VizMotive 의 `Renderer.cpp`):
- `SurfelGI()` → 현재 `1507~1513` 의 DDGI 호출 옆 (UpdateRaytracingAccelerationStructures 직후)
- `SurfelGI_Coverage()` → 현재 `1680` 부근의 주석 처리된 위치 (rtLinearDepth 생성 후, RenderAO 전)

### Phase 4 — Sample14 UI
1. `DDGI` 체크박스 옆에 `SurfelGI` 체크박스 추가
2. 디버그 모드 ImGui Combo (`SURFEL_DEBUG_*` 7개 옵션)
3. `SURFELGI_ENABLED` config 키 추가 + ApplyConfiguration 에 로딩

---

## 3. 데이터 구조 (`ShaderInterop_SurfelGI.h` 업데이트)

### 현재 (옛 포맷, 폐기)

```cpp
struct Surfel { float3 position; uint normal; };
struct SurfelData { uint2 primitiveID; uint bary; uint uid; ... }; // 32bit uid
```

### 목표 (Wicked 9.2 포맷, 그대로)

```cpp
struct alignas(16) Surfel
{
    SH::L1_RGB::Packed radiance;
    uint2 normal;            // packed half3 (uint2 = 4×half = 64bit, 24bit 사용)
    float3 position;
    uint padding1;
    inline float GetRadius() { return SURFEL_MAX_RADIUS; }
};

struct SurfelData
{
    uint64_t uid;            // VizMotive ShaderMeshInstance.uid 와 동일 비트수
    uint2 primitiveID;
    uint bary;
    uint raydata;
    uint properties;
    float max_inconsistency;
    // padding 없음 — 16-byte align 자동
};
```

`SH::L1_RGB::Packed` 의 정확한 크기는 `SH_Lite.hlsli` 의 `Pack()` 시그니처를 따른다. DDGI 가 이미 같은 헤더를 사용 중이므로 호환성 문제 없음.

---

## 4. GPU 리소스 (`Scene_Detail.h`)

DDGI 와 같은 위치에 SurfelGI 구조체를 추가. **Wicked 의 `wiScene.cpp:485-583` 코드를 그대로 포팅**.

```cpp
// Scene_Detail.h, GSceneDetails 내부 (DDGI 옆)
struct SurfelGI {
    bool cleared = false;
    graphics::GPUBuffer surfelBuffer;       // sizeof(Surfel) × 100,000
    graphics::GPUBuffer dataBuffer;         // sizeof(SurfelData) × 100,000
    graphics::GPUBuffer varianceBuffer;     // sizeof(SurfelVarianceDataPacked) × 100,000 × 16
    graphics::GPUBuffer aliveBuffer[2];     // ping-pong, uint × 100,000
    graphics::GPUBuffer deadBuffer;         // uint × 100,000 (역순 인덱스 초기화)
    graphics::GPUBuffer statsBuffer;        // sizeof(SurfelStats), deadCount = CAPACITY 로 초기화
    graphics::GPUBuffer indirectBuffer;     // sizeof(SurfelIndirectArgs), INDIRECT_ARGS flag
    graphics::GPUBuffer gridBuffer;         // sizeof(SurfelGridCell) × TABLE_SIZE
    graphics::GPUBuffer cellBuffer;         // uint × CAPACITY × 27
    graphics::GPUBuffer rayBuffer;          // sizeof(SurfelRayDataPacked) × RAY_BUDGET
    graphics::Texture momentsTexture;       // R16G16_FLOAT, ATLAS_TEXELS²
} surfelgi;
```

`Update()` 흐름 (`SceneUpdate_Detail.cpp` 의 DDGI 생성 코드 옆):
```cpp
if (renderer::isSurfelGIEnabled) {
    if (!surfelgi.surfelBuffer.IsValid()) {
        // 11개 GPU 리소스 생성 (Wicked wiScene.cpp:485-577 그대로)
        // deadBuffer 는 [CAPACITY-1, CAPACITY-2, ..., 0] 역순 초기화
        // statsBuffer.deadCount = SURFEL_CAPACITY 로 초기 데이터 set
    }
    std::swap(surfelgi.aliveBuffer[0], surfelgi.aliveBuffer[1]);
}
else { surfelgi = {}; }
```

### RenderPath3D 측 (Frame-Local 텍스처)

`RenderPath3D_Detail.cpp` 에 결과 텍스처 추가:
```cpp
struct SurfelGIResources {
    graphics::Texture result_halfres;  // half resolution, R11G11B10_FLOAT
    graphics::Texture result;          // full resolution, R11G11B10_FLOAT
} surfelGIResources;
```
`ResizeBuffers()` 에서 internal_resolution 에 맞춰 생성, `BindCommonResources()` 에서:
```cpp
camera->texture_surfelgi_index = device->GetDescriptorIndex(&surfelGIResources.result, SubresourceType::SRV);
```

---

## 5. 셰이더 포팅 매핑 (Wicked → VizMotive)

8개 셰이더를 그대로 복사하되, 셰이더 내부에서 사용되는 외부 API 만 VizMotive 에 맞춤.

| Wicked 9.2 원본 | VizMotive 대상 | 수정 사항 |
|---|---|---|
| `surfel_coverageCS.hlsl` | `EngineShaders/Shaders/CS/` | G-buffer 슬롯 (texture_gbuffer0/1/depth) → VizMotive 의 `texture_normalroughness`, `texture_depth` 로 매핑. **디버그 시각화 코드는 그대로 보존** (push constant 의 `debug` 값으로 분기) |
| `surfel_indirectprepareCS.hlsl` | 동일 | 그대로 |
| `surfel_updateCS.hlsl` | 동일 | `surface.load(prim, bary)` → DDGI raytrace 와 같은 패턴으로 교체 |
| `surfel_gridoffsetsCS.hlsl` | 동일 | 그대로 |
| `surfel_binningCS.hlsl` | 동일 | 그대로 |
| `surfel_raytraceCS.hlsl` (**SW BVH**) | 동일 | DDGI 의 SW BVH 셰이더와 동일하게 `RTAPI` 미정의 분기 사용 |
| `surfel_raytraceCS_rtapi.hlsl` (**RTAPI**) | 동일 | DDGI raytraceCS_rtapi 와 동일하게 `SM_6_5` 컴파일 |
| `surfel_integrateCS.hlsl` | 동일 | `bc6h.hlsli` include 제거 (사용 안 함) |

### 셰이더 등록 (`ShaderLoader.h/cpp`) — DDGI 패턴 그대로

```cpp
// ShaderLoader.h, CSTYPES enum 에 추가:
CSTYPE_SURFEL_COVERAGE,
CSTYPE_SURFEL_INDIRECTPREPARE,
CSTYPE_SURFEL_UPDATE,
CSTYPE_SURFEL_GRIDOFFSETS,
CSTYPE_SURFEL_BINNING,
CSTYPE_SURFEL_RAYTRACE,
CSTYPE_SURFEL_INTEGRATE,

// ShaderLoader.cpp — DDGI 라이팅 등록 패턴 복제
jobsystem::Execute(ctx, [](JobArgs args) { LoadShader(CS, shaders[CSTYPE_SURFEL_COVERAGE], "surfel_coverageCS.cso"); });
jobsystem::Execute(ctx, [](JobArgs args) { LoadShader(CS, shaders[CSTYPE_SURFEL_INDIRECTPREPARE], "surfel_indirectprepareCS.cso"); });
jobsystem::Execute(ctx, [](JobArgs args) { LoadShader(CS, shaders[CSTYPE_SURFEL_UPDATE], "surfel_updateCS.cso"); });
jobsystem::Execute(ctx, [](JobArgs args) { LoadShader(CS, shaders[CSTYPE_SURFEL_GRIDOFFSETS], "surfel_gridoffsetsCS.cso"); });
jobsystem::Execute(ctx, [](JobArgs args) { LoadShader(CS, shaders[CSTYPE_SURFEL_BINNING], "surfel_binningCS.cso"); });
jobsystem::Execute(ctx, [](JobArgs args) { LoadShader(CS, shaders[CSTYPE_SURFEL_INTEGRATE], "surfel_integrateCS.cso"); });

// Raytrace: RT 가능 여부에 따라 분기 (DDGI 의 ddgi_raytraceCS_rtapi 패턴 그대로)
if (device->CheckCapability(GraphicsDeviceCapability::RAYTRACING)) {
    jobsystem::Execute(CTX_raytracing, [](JobArgs args) {
        LoadShader(CS, shaders[CSTYPE_SURFEL_RAYTRACE], "surfel_raytraceCS_rtapi.cso", ShaderModel::SM_6_5);
    });
} else {
    jobsystem::Execute(CTX_raytracing, [](JobArgs args) {
        LoadShader(CS, shaders[CSTYPE_SURFEL_RAYTRACE], "surfel_raytraceCS.cso");
    });
}
```

`compile.bat` 와 `vcxitems`/`vcxitems.filters` 에도 8개 hlsl 모두 등록.

---

## 6. C++ 함수 포팅 (Wicked → VizMotive)

### 6.1 `Postprocess_Upsample_Bilateral` 래퍼 신규 작성

VizMotive 에 셰이더는 있지만 C++ 래퍼는 없음. Wicked `wiRenderer.cpp` 의 동명 함수를 그대로 포팅:

```cpp
// Renderer.cpp
void Postprocess_Upsample_Bilateral(
    const Texture& input,           // half resolution
    const Texture& lineardepth,
    const Texture& output,          // full resolution
    CommandList cmd,
    bool is_pixelshader = false,
    float threshold = 1.0f
);
```

내부 동작:
- 입력 포맷 (`R11G11B10` / `R32F`) 에 따라 `CSTYPE_UPSAMPLE_BILATERAL_FLOAT4` 또는 `_FLOAT1` 셰이더 선택
- threshold 를 push constant 로 전달
- input/lineardepth 를 SR, output 을 UAV 로 바인딩, dispatch

### 6.2 `Update_SurfelGI()` — `GI_Detail.cpp`

Wicked 의 `SurfelGI()` 함수 (`wiRenderer.cpp:12037~`) 를 그대로 포팅:

```cpp
void GRenderPath3DDetails::Update_SurfelGI(CommandList cmd)
{
    if (!scene_Gdetails->surfelgi.surfelBuffer.IsValid()) return;
    if (!scene_Gdetails->TLAS.IsValid() && !scene_Gdetails->BVH.IsValid()) return;

    auto prof_range = profiler::BeginRangeGPU("SurfelGI", &cmd);
    device->EventBegin("SurfelGI", cmd);

    // 1) Initial clear (한 번만)
    if (!scene_Gdetails->surfelgi.cleared) {
        scene_Gdetails->surfelgi.cleared = true;
        device->ClearUAV(&scene_Gdetails->surfelgi.momentsTexture, 0, cmd);
        device->ClearUAV(&scene_Gdetails->surfelgi.varianceBuffer, 0, cmd);
    }

    // 2) Grid reset
    device->ClearUAV(&scene_Gdetails->surfelgi.gridBuffer, 0, cmd);

    // 3) Update — alive→alive ping-pong
    device->BindComputeShader(&shaders[CSTYPE_SURFEL_UPDATE], cmd);
    device->DispatchIndirect(&surfelgi.indirectBuffer, offsetof(SurfelIndirectArgs, iterate), cmd);

    // 4) Grid offsets — prefix sum
    device->BindComputeShader(&shaders[CSTYPE_SURFEL_GRIDOFFSETS], cmd);
    device->Dispatch((SURFEL_TABLE_SIZE + 63) / 64, 1, 1, cmd);

    // 5) Binning
    device->BindComputeShader(&shaders[CSTYPE_SURFEL_BINNING], cmd);
    device->DispatchIndirect(&surfelgi.indirectBuffer, offsetof(SurfelIndirectArgs, iterate), cmd);

    // 6) Raytrace (RTAPI 또는 SW BVH)
    device->BindComputeShader(&shaders[CSTYPE_SURFEL_RAYTRACE], cmd);
    device->DispatchIndirect(&surfelgi.indirectBuffer, offsetof(SurfelIndirectArgs, raytrace), cmd);

    // 7) Integrate
    device->BindComputeShader(&shaders[CSTYPE_SURFEL_INTEGRATE], cmd);
    device->DispatchIndirect(&surfelgi.indirectBuffer, offsetof(SurfelIndirectArgs, integrate), cmd);

    device->EventEnd(cmd);
}
```

호출 위치 (`Renderer.cpp:1507` 의 주석 해제):
```cpp
if (scene_Gdetails->isAccelerationStructureUpdateRequested)
    scene_Gdetails->UpdateRaytracingAccelerationStructures(cmd);
if (renderer::isSurfelGIEnabled)
    Update_SurfelGI(cmd);
if (renderer::isDDGIEnabled)
    Update_DDGI(cmd);
```

### 6.3 `SurfelGI_Coverage()` — `Renderer.cpp`

Wicked `wiRenderer.cpp:11944~` 의 동명 함수를 그대로 포팅. **Coverage CS + IndirectPrepare CS + Bilateral Upsample 의 3단계**:

```cpp
void GRenderPath3DDetails::SurfelGI_Coverage(
    const SurfelGIResources& res, const Scene& scene,
    const Texture& linearDepth, const Texture& debugUAV,
    CommandList cmd)
{
    auto prof_range = profiler::BeginRangeGPU("SurfelGI - Coverage", &cmd);
    device->EventBegin("SurfelGI - Coverage", cmd);

    device->ClearUAV(&res.result_halfres, 0, cmd);

    // 1) Coverage CS — G-buffer 읽고, GI 출력, 새 surfel 스폰, debugUAV 출력
    device->BindComputeShader(&shaders[CSTYPE_SURFEL_COVERAGE], cmd);
    SurfelDebugPushConstants push;
    push.debug = renderer::GetSurfelGIDebugMode();
    device->PushConstants(&push, sizeof(push), cmd);
    /* SR: surfelBuffer, gridBuffer, cellBuffer, momentsTexture, depth, normal_roughness
       UAV: dataBuffer, deadBuffer, aliveBuffer (write side), statsBuffer, result_halfres, debugUAV */
    device->Dispatch(
        (res.result_halfres.desc.width + 15) / 16,
        (res.result_halfres.desc.height + 15) / 16, 1, cmd);

    // 2) IndirectPrepare — 다음 프레임용 indirect args 계산
    device->BindComputeShader(&shaders[CSTYPE_SURFEL_INDIRECTPREPARE], cmd);
    device->Dispatch(1, 1, 1, cmd);

    // 3) Half→Full bilateral upsample
    Postprocess_Upsample_Bilateral(
        res.result_halfres, linearDepth, res.result, cmd, false, 2);

    device->EventEnd(cmd);
}
```

호출 위치 (`Renderer.cpp:1680` 의 주석 해제 + 활성화):
```cpp
if (renderer::isSurfelGIEnabled) {
    SurfelGI_Coverage(surfelGIResources, *scene, rtLinearDepth, debugUAV, cmd);
}
```

### 6.4 Renderer 전역 + 디버그 enum

`Renderer.h`:
```cpp
extern bool isSurfelGIEnabled;
SURFEL_DEBUG GetSurfelGIDebugMode();    // SurfelDebugPushConstants 에 들어갈 값
void SetSurfelGIDebugMode(SURFEL_DEBUG value);
```

`ShaderEngine.cpp`:
```cpp
bool isSurfelGIEnabled = false;
static SURFEL_DEBUG g_surfelDebug = SURFEL_DEBUG_NONE;
SURFEL_DEBUG GetSurfelGIDebugMode() { return g_surfelDebug; }
void SetSurfelGIDebugMode(SURFEL_DEBUG v) { g_surfelDebug = v; }

// ApplyConfiguration:
isSurfelGIEnabled = config::GetBoolConfig("SHADER_ENGINE_SETTINGS", "SURFELGI_ENABLED");
```

기존 `isSurfelGIDebugEnabled` (bool) 는 `g_surfelDebug != SURFEL_DEBUG_NONE` 체크로 대체.

### 6.5 Frame CB 옵션 비트 set

`RenderPath3D_Detail.cpp` 의 frame CB 작성 부분에:
```cpp
if (renderer::isSurfelGIEnabled) frameCB.options |= OPTION_BIT_SURFELGI_ENABLED;
```

---

## 7. Sample14 UI

`sample14.cpp` 의 DDGI 체크박스 옆에 추가 (현재 ~748번 라인 부근):

```cpp
static bool surfelgi_enabled = vz::config::GetBoolConfig("SHADER_ENGINE_SETTINGS", "SURFELGI_ENABLED");
if (ImGui::Checkbox("SurfelGI", &surfelgi_enabled)) {
    vzm::ParamMap<std::string> config_options;
    config_options.SetParam("SURFELGI_ENABLED", surfelgi_enabled);
    vzm::SetConfigure(config_options);
}

// Debug mode combo (SURFEL_DEBUG_* 7개)
static int surfelgi_debug = (int)SURFEL_DEBUG_NONE;
const char* debugItems[] = {
    "None", "Normal", "Color", "Point", "Random", "Heatmap", "Inconsistency"
};
if (ImGui::Combo("SurfelGI Debug", &surfelgi_debug, debugItems, IM_ARRAYSIZE(debugItems))) {
    renderer->SetSurfelGIDebugMode((SURFEL_DEBUG)surfelgi_debug);
}
```

---

## 8. 작업 단위 체크리스트

### Phase 1: Interop & Resources
- [ ] `ShaderInterop_SurfelGI.h` Wicked 9.2 포맷으로 업데이트 (Surfel 에 SH L1 RGB radiance, `uint64_t uid`)
- [ ] `Scene_Detail.h` 에 `GSceneDetails::SurfelGI` 구조체 추가
- [ ] `SceneUpdate_Detail.cpp` 에 GPU 버퍼/텍스처 생성 코드 추가 (Wicked `wiScene.cpp:485-583` 그대로)
- [ ] `RenderPath3D_Detail.cpp` 에 `SurfelGIResources` 추가 + `texture_surfelgi_index` 노출
- [ ] `Renderer.h` / `ShaderEngine.cpp` 에 `isSurfelGIEnabled` + `Get/SetSurfelGIDebugMode` 추가
- [ ] `ShaderEngine.cpp::ApplyConfiguration` 에 `SURFELGI_ENABLED` 로딩 추가
- [ ] `RenderPath3D_Detail.cpp` 의 frame CB 작성 부분에 `OPTION_BIT_SURFELGI_ENABLED` set
- [ ] `Postprocess_Upsample_Bilateral` C++ 래퍼 구현 (Wicked 동명 함수 포팅)

### Phase 2: Shader Port
- [ ] Wicked surfel CS 8개 파일을 `EngineShaders/Shaders/CS/` 로 복사 (디버그 시각화 코드 포함)
- [ ] G-buffer / surface load / instance UID 부분을 VizMotive API 에 맞게 수정 (DDGI raytrace 셰이더 참고)
- [ ] `surfel_integrateCS.hlsl` 의 `bc6h.hlsli` include 제거
- [ ] `ShaderLoader.h` 에 `CSTYPE_SURFEL_*` 7개 enum 추가
- [ ] `ShaderLoader.cpp` 에 LoadShader 등록 (raytrace 만 RT 지원 분기)
- [ ] `ModernShaders.vcxitems(.filters)` + `compile.bat` 에 8개 hlsl 등록

### Phase 3: Pipeline
- [ ] `GI_Detail.cpp` 에 `Update_SurfelGI()` 함수 작성 (Update→GridOffsets→Binning→Raytrace→Integrate)
- [ ] `Renderer.cpp` 에 `SurfelGI_Coverage()` 함수 작성 (Coverage→IndirectPrepare→Upsample)
- [ ] `Renderer.cpp:1507` 의 SurfelGI 호출 주석 해제 + `isSurfelGIEnabled` 분기
- [ ] `Renderer.cpp:1680` 의 SurfelGI_Coverage 호출 주석 해제 + 정확한 인자
- [ ] (검증) `Renderer.cpp:796` 의 `Postprocess_Upsample_Bilateral` 주석 해제도 같이 활성화 가능한지

### Phase 4: Sample14 UI
- [ ] DDGI 옆에 SurfelGI 체크박스 + config 저장
- [ ] SurfelGI Debug Combo (7개 모드) — `SetSurfelGIDebugMode` 호출
- [ ] (선택) Surfel 통계 표시 (count/dead/rayCount — readback 필요)

### Phase 5: 검증
- [ ] 단순 씬 (cornell box 스타일) 에서 DDGI 끄고 SurfelGI 만으로 GI 적용 확인
- [ ] Sample14 의 동적 큐브 추가 시 surfel 이 표면에 따라붙는지 (debug heatmap)
- [ ] 카메라 이동 시 멀어지는 surfel 이 recycle 되는지 (debug random color 로 ID 가 바뀌는지)
- [ ] DDGI + SurfelGI 동시 활성화 시 `shadingHF.hlsli:355` 우선순위 확인
- [ ] 디버그 모드 7개가 모두 정상 출력되는지 (None/Normal/Color/Point/Random/Heatmap/Inconsistency)

---

## 9. 작업 순서 제안 (현실적인 일정)

1. **Day 1 — Phase 1**: 인프라 스캐폴딩 (셰이더 없이 빌드 통과). GPU 리소스 생성 + binding 에러 없음.
2. **Day 2 — Phase 2 (셰이더)**: 8개 CS 파일 포팅 + 컴파일 성공.
3. **Day 3 — Phase 3 (RT 빠진 상태)**: Update/GridOffsets/Binning 만 동작. Coverage CS 로 surfel 이 스폰되는지 stats readback 확인.
4. **Day 4 — Phase 3 (RT 활성화)**: Raytrace + Integrate 활성화, SH radiance 가 채워지는지 확인. `Postprocess_Upsample_Bilateral` 도 같이.
5. **Day 5 — Phase 4 + Phase 5**: UI 토글, 디버그 모드 검증, DDGI 와 비교.

---

## 10. 예상 이슈 및 검토 포인트

| 이슈 | 원인 | 대응 |
|------|------|------|
| **`Surface.load(prim, bary)` API 차이** | Wicked 와 VizMotive 의 메시 표면 로드 헬퍼 이름이 다를 수 있음 | DDGI raytrace 셰이더가 hit point 에서 surface 를 로드하는 패턴 그대로 사용 |
| **G-buffer 슬롯 매핑** | `texture_gbuffer0/1` ↔ VizMotive 슬롯 (`texture_normalroughness` 등) 차이 | DDGI 셰이더와 `objectHF.hlsli` 의 G-buffer 출력 패턴 동기화 |
| **`bc6h.hlsli` include** | `surfel_integrateCS.hlsl` 이 include 만 하고 사용 안 함 | 제거 (VizMotive 에 없으면 컴파일 실패) |
| **`CreateBufferZeroed` / `CreateBuffer2`** | `deadBuffer` 역순 초기화에 `CreateBuffer2(buf, fill_callback, ...)` 패턴 사용 | VizMotive `GraphicsDevice` 에 동일 API 가 있는지 확인. 없으면 staging buffer 로 대체 |
| **debug push constant 슬롯** | 다른 push constant 와 슬롯 충돌 가능성 | DDGI 의 push constant 패턴 동일하게 적용 |
| **`Postprocess_Upsample_Bilateral` push constant** | Wicked 셰이더가 기대하는 push constant 구조체와 VizMotive 셰이더 호환성 | VizMotive 의 `upsample_bilateral_float4CS.hlsl` 헤더 직접 확인 |
| **Indirect args alignment** | DX12 의 IndirectDispatchArgs 자연 정렬 | 헤더의 `vz::graphics::AlignTo` 매크로가 처리 중 — 그대로 사용 |
| **Surfel 디버그 시각화 표시 위치** | `debugUAV` 가 메인 RT 와 합성되는 지점 확인 필요 | Wicked 는 `RenderPath3D::Compose` 에서 합성 — VizMotive 도 같은 위치 찾기 |

---

## 11. 참고 사항

- **Wicked 9.2 레퍼런스 위치:**
  - `wiScene.cpp:485-583` — GPU 리소스 생성 (그대로 포팅)
  - `wiRenderer.cpp:11929-12035` — `CreateSurfelGIResources` + `SurfelGI_Coverage`
  - `wiRenderer.cpp:12037~12200` — `SurfelGI` 함수 (Update~Integrate 전체)
  - `wiRenderer.cpp:1121-1133` — 셰이더 로드 등록 패턴
  - `wiRenderer.cpp:19285-19297` — Get/Set 함수
  - `wiRenderPath3D.cpp:528, 743, 899, 1086` — 호출 위치
- **VizMotive DDGI 레퍼런스 위치:**
  - `GI_Detail.cpp:5-200` — `Update_DDGI()` (가장 가까운 패턴)
  - DDGI raytrace CS 등록 (RT/SW 분기) 패턴 — `ShaderLoader.cpp:765-774`
- **현재 master 브랜치 기준** 작성. VXGI 작업이 진행 중인 `Add-VoxelGI` 브랜치에서 머지된 후 작업 시작 권장 (`Renderer.cpp` 충돌 가능성).
