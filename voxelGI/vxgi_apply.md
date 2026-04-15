# VizMotive에 VoxelGI 추가하기

> **목표**: WickedEngine의 VXGI 파이프라인을 VizMotive에 이식  
> **참고**: voxelGI.md 분석 기반. 셰이더 코드는 이미 VizMotive에 존재하며 확인 필요.  
> **읽는 순서**: 아래 절차대로 읽으면 전체 파이프라인의 흐름이 보임.

---

## 현재 상태 (VizMotive에 이미 있는 것)

| 항목 | 파일 | 상태 |
|------|------|------|
| ShaderInterop_VXGI.h | EngineShaders/Shaders/ | ✅ 존재 |
| FrameCB.vxgi 필드 | ShaderInterop_Renderer.h:1208 | ✅ 존재 |
| OPTION_BIT_VXGI_ENABLED | ShaderInterop_Renderer.h | ✅ 존재 |
| meshVS/GS/PS_voxelizer.hlsl | Shaders/VS, GS, PS/ | ✅ 존재 |
| voxelConeTracingHF.hlsli | Shaders/CommonHF/ | ✅ 존재 |
| VSTYPE/GSTYPE/PSTYPE_VOXELIZER | ShaderLoader.h | ✅ 존재 |
| RENDERPASS_VOXELIZE 파이프라인 | ShaderLoader.cpp | ✅ 존재 |
| voxelGS.hlsl (debug vis) | Shaders/GS/ | ✅ 존재 |
| **VXGI compute shaders** | Shaders/CS/ | ❌ 없음 |
| **Scene VXGI GPU 리소스** | Scene_Detail.h | ❌ 없음 |
| **Update_VXGI 함수** | GI_Detail.cpp | ❌ 없음 |
| **Scene CPU update** | SceneUpdate_Detail.cpp | ❌ 없음 |
| **RenderPath3D 통합** | Renderer.cpp | ❌ 없음 |
| **Lighting pass 적용** | shadingHF.hlsli 등 | ❌ 없음 |

---

## 파이프라인 전체 흐름 (시간 순)

```
[매 프레임]
  ① SceneUpdate:  camera 위치로 clipmap center 갱신, offsetFromPrevFrame 계산
  ② FrameCB 업데이트: vxgi.texture_radiance, texture_sdf 인덱스 바인딩
  ③ Voxelization:  씬 메시 → output_atomic 3D 텍스처에 atomic 기록
  ④ Temporal CS:   atomic 언패킹 + temporal blend + cone 방향 precompute + SDF init
  ⑤ SDF Flood CS:  Jump Flood로 빈 복셀까지 거리 전파 (log2(64)=6 pass)
  ⑥ Resolve Diffuse CS:  화면 픽셀마다 ConeTraceDiffuse → 간접 diffuse 2D 텍스처
  ⑦ Resolve Specular CS: 화면 픽셀마다 ConeTraceSpecular → 간접 specular 2D 텍스처
  ⑧ Lighting Pass:  기존 lighting에 VXGI diffuse/specular 결과 합산

  [매 6프레임마다 clipmap 하나씩 로테이션 업데이트]
```

---

## Step 1 — GPU 리소스 정의 (Scene_Detail.h)

DDGI가 `GSceneDetails::DDGI ddgi;` 구조체로 리소스를 모으는 것과 동일한 패턴.

### 추가 위치
`EngineShaders/ShaderEngine/Scene_Detail.h` → `struct GSceneDetails` 안 (ddgi 아래쪽)

### 추가할 코드
```cpp
// VXGI resources
struct VXGI
{
    // --- 설정값 ---
    uint32_t res = 64;               // 복셀 그리드 해상도 (한 변의 복셀 수)
    float maxDistance = 100.0f;      // cone tracing 최대 거리 (월드 단위)

    // --- Clipmap 상태 (CPU side) ---
    uint32_t clipmap_to_update = 0;  // 이번 프레임에 업데이트할 clipmap 레벨 인덱스 (0~5 로테이션)

    // --- GPU 텍스처 ---

    // output_atomic: Voxelization PS가 InterlockedAdd로 쓰는 중간 버퍼
    //   크기: (res*6) x res x (res*13) — 6 face 슬라이스 × 13 채널
    //   Format: R32_UINT (atomic 연산 요구사항)
    //   한 clipmap 레벨만 저장 (매 프레임 클리어 후 재사용)
    graphics::Texture texture_atomic;

    // texture_radiance: Temporal CS의 출력, Cone Tracing이 읽는 최종 radiance
    //   크기: (res*22) x (res*6) x res
    //     X: 22 슬라이스 (6 anisotropic face + 16 diffuse cone) × res
    //     Y: 6 clipmap 레벨 × res
    //     Z: res (깊이)
    //   Format: R16G16B16A16_FLOAT (half4, 읽기 효율)
    //   이전 프레임 데이터 보존 (temporal blend용)
    graphics::Texture texture_radiance;

    // texture_sdf: Temporal CS가 초기 기록, SDF Jump Flood가 정제
    //   크기: res x (res*6) x res
    //     Y: 6 clipmap 레벨 × res
    //   Format: R32_FLOAT (거리값)
    graphics::Texture texture_sdf;

    // texture_sdf_temp: SDF Jump Flood ping-pong용 임시 버퍼 (sdf와 동일 크기)
    graphics::Texture texture_sdf_temp;

    // Resolve 결과 (화면 해상도 2D 텍스처)
    graphics::Texture texture_diffuse;   // Resolve Diffuse CS 출력
    graphics::Texture texture_specular;  // Resolve Specular CS 출력

} vxgi;
```

**포인트**:
- `texture_atomic`은 매 voxelization 시작 전 `ClearUAV`로 초기화. 그러므로 크기를 clipmap 하나 분량으로만 만들어도 됨.
- `texture_radiance`는 이전 프레임 데이터를 보존하므로 클리어하지 않음.

---

## Step 2 — ShaderInterop_VXGI.h 확인

**이미 존재** (`EngineShaders/Shaders/ShaderInterop_VXGI.h`). 내용 확인 필요:

```cpp
struct alignas(16) VXGI {
    uint   resolution;       // 복셀 그리드 해상도 (= res)
    float  resolution_rcp;   // 1.0 / resolution
    float  stepsize;         // cone tracing step size
    float  max_distance;

    int    texture_radiance; // bindless 인덱스
    int    texture_sdf;      // bindless 인덱스
    int    padding0;
    int    padding1;

    VoxelClipMap clipmaps[VXGI_CLIPMAP_COUNT];  // VXGI_CLIPMAP_COUNT = 6

    // world_to_clipmap(), clipmap_to_world() 헬퍼 함수 포함 (#ifndef __cplusplus)
};

struct alignas(16) VoxelizerCB {  // 복셀화 패스에서 사용하는 CB
    int3  offsetfromPrevFrame;
    int   clipmap_index;
};
```

`FrameCB` 안에 `VXGI vxgi;`가 이미 있음 → 셰이더에서 `GetFrame().vxgi.xxx`로 접근 가능.

---

## Step 3 — Scene CPU 업데이트 (SceneUpdate_Detail.cpp)

매 프레임 카메라 위치 기반으로 clipmap center를 양자화하고 이전 프레임과의 차이(offset)를 계산.

### 추가 위치
`EngineShaders/ShaderEngine/SceneUpdate_Detail.cpp` → 씬 업데이트 함수 내부  
(DDGI 초기화 패턴 참고: scene_Gdetails->ddgi 업데이트 부분)

### 추가할 코드
```cpp
// VXGI clipmap center 업데이트
if (renderer::isVXGIEnabled)  // 나중에 Renderer.h에 추가할 플래그
{
    GSceneDetails::VXGI& vxgi = scene_Gdetails->vxgi;
    const uint32_t clip_idx = vxgi.clipmap_to_update;

    // clipmap voxelSize: 레벨마다 2배씩 커짐
    // 레벨 0 = 가장 세밀, 레벨 5 = 가장 넓음
    // voxelSize0 = scene에서 설정한 기본 복셀 크기 (예: 0.125f)
    const float baseVoxelSize = 0.125f; // TODO: 설정 가능하게
    const float voxelSize = baseVoxelSize * std::pow(2.0f, (float)clip_idx);
    const float texelSize = voxelSize * 2.0f;  // 복셀 하나의 월드 크기

    // GPU에 올릴 clipmap 데이터 (ShaderInterop_VXGI.h의 VoxelClipMap)
    XMFLOAT3 camEye = camera.Eye;  // 현재 카메라 위치

    // 양자화: 복셀 그리드 경계에 snap
    XMFLOAT3 newCenter = {
        std::floor(camEye.x / texelSize) * texelSize,
        std::floor(camEye.y / texelSize) * texelSize,
        std::floor(camEye.z / texelSize) * texelSize
    };

    // offsetFromPrevFrame: 이전 프레임과 현재 프레임의 그리드 시프트량 (복셀 단위 정수)
    // Temporal CS에서 이 값으로 이전 데이터를 현재 좌표로 보정함
    VoxelClipMap& clipmap = scene_Gdetails->shaderscene.vxgi.clipmaps[clip_idx];  // TODO: 실제 경로 확인
    XMFLOAT3 prevCenter = clipmap.center;
    Int3 offset = {
        (int)((prevCenter.x - newCenter.x) / texelSize),
        -(int)((prevCenter.y - newCenter.y) / texelSize),  // Y축 뒤집기
        (int)((prevCenter.z - newCenter.z) / texelSize)
    };
    // → VoxelizerCB.offsetfromPrevFrame으로 GPU에 전달

    clipmap.center = newCenter;
    clipmap.voxelSize = voxelSize;
}
```

**포인트**:
- `offsetfromPrevFrame`이 0이 아닐 때 temporal CS에서 이전 프레임 radiance를 시프트해서 재사용.
- `voxelSize`는 복셀 반지름 개념. WickedEngine에서는 `voxelSize * 2 = texelSize` (한 복셀의 전체 크기).

---

## Step 4 — FrameCB 바인딩 업데이트

`FrameCB.vxgi.texture_radiance`, `.texture_sdf`에 bindless descriptor index를 넣어야 셰이더에서 접근 가능.

### 추가 위치
FrameCB를 업데이트하는 함수 (`UpdatePerFrameData` 또는 동등한 위치)

```cpp
// VXGI texture bindings in FrameCB
if (renderer::isVXGIEnabled && scene_Gdetails->vxgi.texture_radiance.IsValid())
{
    frameCB.vxgi.texture_radiance = device->GetDescriptorIndex(
        &scene_Gdetails->vxgi.texture_radiance, SubresourceType::SRV);
    frameCB.vxgi.texture_sdf = device->GetDescriptorIndex(
        &scene_Gdetails->vxgi.texture_sdf, SubresourceType::SRV);
    frameCB.vxgi.resolution = scene_Gdetails->vxgi.res;
    frameCB.vxgi.resolution_rcp = 1.0f / (float)scene_Gdetails->vxgi.res;
    frameCB.vxgi.stepsize = 1.0f;
    frameCB.vxgi.max_distance = scene_Gdetails->vxgi.maxDistance;
    frameCB.options |= OPTION_BIT_VXGI_ENABLED;
}
```

---

## Step 5 — GPU 텍스처 생성 함수

텍스처는 처음 enable되거나 해상도가 변경될 때 한 번만 생성.

### 추가 위치
`GI_Detail.cpp` 상단에 새 함수로 추가 (또는 `Update_VXGI` 내부에서 `if (!texture.IsValid())` 조건부 생성)

```cpp
void GRenderPath3DDetails::CreateVXGIResources()
{
    auto* device = graphics::GetDevice();
    const uint32_t res = scene_Gdetails->vxgi.res;

    // --- output_atomic ---
    // 한 clipmap 레벨용. X = 6 face × res, Y = res, Z = 13 channel × res
    {
        TextureDesc desc;
        desc.type = TextureDesc::Type::TEXTURE_3D;
        desc.width  = res * 6;                     // 6 anisotropic faces (cone은 temporal에서 처리)
        desc.height = res;
        desc.depth  = res * VOXELIZATION_CHANNEL_COUNT;  // 13
        desc.format = Format::R32_UINT;
        desc.bind_flags = BindFlag::UNORDERED_ACCESS;
        desc.usage = Usage::DEFAULT;
        device->CreateTexture(&desc, nullptr, &scene_Gdetails->vxgi.texture_atomic);
        device->SetName(&scene_Gdetails->vxgi.texture_atomic, "vxgi_atomic");
    }

    // --- texture_radiance ---
    // 전체 clipmap 레벨 + 22 방향 슬라이스
    {
        TextureDesc desc;
        desc.type = TextureDesc::Type::TEXTURE_3D;
        desc.width  = res * (6 + DIFFUSE_CONE_COUNT);  // 22 slices
        desc.height = res * VXGI_CLIPMAP_COUNT;          // 6 clipmap levels
        desc.depth  = res;
        desc.format = Format::R16G16B16A16_FLOAT;
        desc.bind_flags = BindFlag::SHADER_RESOURCE | BindFlag::UNORDERED_ACCESS;
        desc.usage = Usage::DEFAULT;
        // Mip 없음 (Temporal에서 직접 trilinear 샘플링)
        device->CreateTexture(&desc, nullptr, &scene_Gdetails->vxgi.texture_radiance);
        device->SetName(&scene_Gdetails->vxgi.texture_radiance, "vxgi_radiance");
    }

    // --- texture_sdf / sdf_temp ---
    {
        TextureDesc desc;
        desc.type = TextureDesc::Type::TEXTURE_3D;
        desc.width  = res;
        desc.height = res * VXGI_CLIPMAP_COUNT;  // 6 clipmap levels
        desc.depth  = res;
        desc.format = Format::R32_FLOAT;
        desc.bind_flags = BindFlag::SHADER_RESOURCE | BindFlag::UNORDERED_ACCESS;
        desc.usage = Usage::DEFAULT;
        device->CreateTexture(&desc, nullptr, &scene_Gdetails->vxgi.texture_sdf);
        device->CreateTexture(&desc, nullptr, &scene_Gdetails->vxgi.texture_sdf_temp);
        device->SetName(&scene_Gdetails->vxgi.texture_sdf, "vxgi_sdf");
        device->SetName(&scene_Gdetails->vxgi.texture_sdf_temp, "vxgi_sdf_temp");
    }

    // --- Resolve 결과 (화면 해상도 2D) ---
    {
        TextureDesc desc;
        desc.type = TextureDesc::Type::TEXTURE_2D;
        desc.width  = internalResolution.x;  // 렌더 해상도
        desc.height = internalResolution.y;
        desc.format = Format::R16G16B16A16_FLOAT;
        desc.bind_flags = BindFlag::SHADER_RESOURCE | BindFlag::UNORDERED_ACCESS;
        desc.usage = Usage::DEFAULT;
        device->CreateTexture(&desc, nullptr, &scene_Gdetails->vxgi.texture_diffuse);
        device->CreateTexture(&desc, nullptr, &scene_Gdetails->vxgi.texture_specular);
        device->SetName(&scene_Gdetails->vxgi.texture_diffuse, "vxgi_diffuse");
        device->SetName(&scene_Gdetails->vxgi.texture_specular, "vxgi_specular");
    }
}
```

---

## Step 6 — VXGI Compute Shader 파일 추가

WickedEngine 소스에서 아래 4개 CS 파일을 복사하여 `EngineShaders/Shaders/CS/`에 추가:

| 파일명 | 역할 |
|--------|------|
| `vxgi_temporalCS.hlsl` | atomic 언패킹 + temporal blend + SDF init + cone precompute |
| `vxgi_sdf_jumpfloodCS.hlsl` | Jump Flood로 SDF 정제 |
| `vxgi_resolve_diffuseCS.hlsl` | 화면 픽셀별 ConeTraceDiffuse → 2D 출력 |
| `vxgi_resolve_specularCS.hlsl` | 화면 픽셀별 ConeTraceSpecular → 2D 출력 |

복사 후 VizMotive 셰이더 include 구조에 맞게 확인:
- `GetFrame().vxgi` 접근 → VizMotive의 `Globals.hlsli`가 include되면 자동 동작
- `ConeTraceDiffuse`, `ConeTraceSpecular` → `voxelConeTracingHF.hlsli` include (이미 VizMotive에 존재)
- `g_xVoxelizer` constant buffer → `ShaderInterop_VXGI.h`의 `VoxelizerCB` 사용

---

## Step 7 — ShaderLoader에 CSTYPE 추가

`EngineShaders/ShaderEngine/ShaderLoader.h` → CSTYPE 열거형에 추가:

```cpp
// VXGI
CSTYPE_VXGI_TEMPORAL,
CSTYPE_VXGI_SDF_JUMPFLOOD,
CSTYPE_VXGI_RESOLVE_DIFFUSE,
CSTYPE_VXGI_RESOLVE_SPECULAR,
```

`EngineShaders/ShaderEngine/ShaderLoader.cpp` → shader 로드 코드에 추가 (DDGI 로드 패턴 참고):

```cpp
jobsystem::Execute(ctx, [](jobsystem::JobArgs args) {
    LoadShader(ShaderStage::CS, shaders[CSTYPE_VXGI_TEMPORAL],          "vxgi_temporalCS.cso"); });
jobsystem::Execute(ctx, [](jobsystem::JobArgs args) {
    LoadShader(ShaderStage::CS, shaders[CSTYPE_VXGI_SDF_JUMPFLOOD],     "vxgi_sdf_jumpfloodCS.cso"); });
jobsystem::Execute(ctx, [](jobsystem::JobArgs args) {
    LoadShader(ShaderStage::CS, shaders[CSTYPE_VXGI_RESOLVE_DIFFUSE],   "vxgi_resolve_diffuseCS.cso"); });
jobsystem::Execute(ctx, [](jobsystem::JobArgs args) {
    LoadShader(ShaderStage::CS, shaders[CSTYPE_VXGI_RESOLVE_SPECULAR],  "vxgi_resolve_specularCS.cso"); });
```

---

## Step 8 — Update_VXGI 함수 (GI_Detail.cpp)

DDGI의 `Update_DDGI` 패턴을 따라 작성.

**함수 시그니처** → `RenderPath3D_Detail.h`에 선언:
```cpp
void Update_VXGI(CommandList cmd);
```

**구현** → `GI_Detail.cpp`에 추가:

```cpp
void GRenderPath3DDetails::Update_VXGI(CommandList cmd)
{
    if (!scene_Gdetails->vxgi.texture_radiance.IsValid())
        CreateVXGIResources();

    auto prof_range = profiler::BeginRangeGPU("VXGI", &cmd);
    device->EventBegin("VXGI", cmd);

    GSceneDetails::VXGI& vxgi = scene_Gdetails->vxgi;
    const uint32_t clip_idx = vxgi.clipmap_to_update;
    const uint32_t res = vxgi.res;

    BindCommonResources(cmd);  // FrameCB 등 공통 리소스 바인딩

    // ── [A] output_atomic 클리어 ──────────────────────────────────────
    // 이번 프레임 voxelization 결과를 축적하기 위해 초기화
    {
        device->Barrier(GPUBarrier::Image(&vxgi.texture_atomic,
            ResourceState::SHADER_RESOURCE_COMPUTE, ResourceState::UNORDERED_ACCESS), cmd);
        device->ClearUAV(&vxgi.texture_atomic, 0, cmd);
        device->Barrier(GPUBarrier::Memory(&vxgi.texture_atomic), cmd);
    }

    // ── [B] Voxelization (메시 래스터화) ────────────────────────────────
    // 씬 메시를 '가상 64×64 스크린'에 래스터화.
    // PS(meshPS_voxelizer)가 각 복셀 위치에서 InterlockedAdd로 texture_atomic에 기록.
    {
        device->EventBegin("Voxelize", cmd);

        // VoxelizerCB 설정
        VoxelizerCB voxelizerCB;
        voxelizerCB.clipmap_index = (int)clip_idx;
        voxelizerCB.offsetfromPrevFrame = /* Step 3에서 계산한 값 */;

        // ForwardEntityMask (광원 컬링 비트마스크) 설정
        // → meshPS_voxelizer의 ForwardLighting이 광원별 직접광 계산에 사용
        ForwardEntityMaskCB lightmask = ForwardEntityCullingCPU(visMain, clipmapAABB, RENDERPASS_VOXELIZE);
        device->BindConstantBuffer(&lightmaskBuffer, CBSLOT_RENDERER_FORWARD_LIGHTMASK, cmd);

        // render target 없이 UAV write만 허용
        device->RenderPassBegin(nullptr, 0, cmd, RenderPassFlags::ALLOW_UAV_WRITES);

        // output_atomic을 u0에 바인딩
        device->BindUAV(&vxgi.texture_atomic, 0, cmd);

        // 씬 메시 렌더링 (RENDERPASS_VOXELIZE → VS/GS/PS 자동 선택)
        DrawSceneMeshes(visMain, RENDERPASS_VOXELIZE, DRAWSCENE_OPAQUE, cmd);

        device->RenderPassEnd(cmd);
        device->EventEnd(cmd);
    }

    // ── [C] atomic → radiance 변환 (Temporal CS) ──────────────────────
    // texture_atomic을 읽어 radiance(half4)로 변환하고,
    // 이전 프레임 radiance와 temporal blend.
    // 16개 cone 방향 precompute와 SDF 초기값도 여기서 설정.
    {
        device->EventBegin("Temporal", cmd);

        // 바리어: atomic은 SRV로, radiance는 UAV로
        device->Barrier(GPUBarrier::Image(&vxgi.texture_atomic,
            ResourceState::UNORDERED_ACCESS, ResourceState::SHADER_RESOURCE_COMPUTE), cmd);
        device->Barrier(GPUBarrier::Image(&vxgi.texture_radiance,
            ResourceState::SHADER_RESOURCE_COMPUTE, ResourceState::UNORDERED_ACCESS), cmd);
        device->Barrier(GPUBarrier::Image(&vxgi.texture_sdf,
            ResourceState::SHADER_RESOURCE_COMPUTE, ResourceState::UNORDERED_ACCESS), cmd);

        device->BindComputeShader(&shaders[CSTYPE_VXGI_TEMPORAL], cmd);

        // 입력: t0 = texture_radiance (이전 프레임), t1 = texture_atomic
        const GPUResource* srvs[] = { &vxgi.texture_radiance, &vxgi.texture_atomic };
        device->BindResources(srvs, 0, 2, cmd);

        // 출력: u0 = output_radiance, u1 = output_sdf
        const GPUResource* uavs[] = { &vxgi.texture_radiance, &vxgi.texture_sdf };
        device->BindUAVs(uavs, 0, 2, cmd);

        // VoxelizerCB 바인딩 (clipmap_index, offsetfromPrevFrame 포함)
        device->BindConstantBuffer(&voxelizerCB_buffer, CBSLOT_RENDERER_VOXELIZER, cmd);

        // Dispatch: 복셀 그리드 전체 (res/8 × res/8 × res/8 워크그룹)
        device->Dispatch(res / 8, res / 8, res / 8, cmd);

        device->Barrier(GPUBarrier::Image(&vxgi.texture_radiance,
            ResourceState::UNORDERED_ACCESS, ResourceState::SHADER_RESOURCE_COMPUTE), cmd);
        device->Barrier(GPUBarrier::Image(&vxgi.texture_sdf,
            ResourceState::UNORDERED_ACCESS, ResourceState::SHADER_RESOURCE_COMPUTE), cmd);

        device->EventEnd(cmd);
    }

    // ── [D] SDF Jump Flood (거리 필드 정제) ────────────────────────────
    // 표면 복셀(sdf=0) 주변 빈 복셀들의 정확한 거리값 계산.
    // log2(res) = 6번 pass, ping-pong 텍스처 스왑.
    {
        device->EventBegin("SDF Jump Flood", cmd);
        device->BindComputeShader(&shaders[CSTYPE_VXGI_SDF_JUMPFLOOD], cmd);

        const Texture* _write = &vxgi.texture_sdf_temp;
        const Texture* _read  = &vxgi.texture_sdf;

        int passcount = (int)std::ceil(std::log2((float)res));  // = 6 (for res=64)
        for (int i = 0; i < passcount; ++i)
        {
            float jump_size = std::pow(2.0f, float(passcount - i - 1));
            // push constant: jump_size (몇 복셀씩 점프할지)
            device->PushConstants(&jump_size, sizeof(float), cmd);

            device->Barrier(GPUBarrier::Image(_read,
                ResourceState::UNORDERED_ACCESS, ResourceState::SHADER_RESOURCE_COMPUTE), cmd);
            device->Barrier(GPUBarrier::Image(_write,
                ResourceState::SHADER_RESOURCE_COMPUTE, ResourceState::UNORDERED_ACCESS), cmd);

            device->BindResource(_read, 0, cmd);
            device->BindUAV(_write, 0, cmd);

            device->Dispatch(res / 8, res / 8, res / 8, cmd);

            if (i < passcount - 1)
                std::swap(_read, _write);
        }

        device->EventEnd(cmd);
    }

    // ── [E] Resolve Diffuse (화면 픽셀 → 간접 diffuse 2D 출력) ───────────
    // G-Buffer(depth, normal)에서 표면 위치와 법선을 읽고,
    // texture_radiance로 ConeTraceDiffuse → texture_diffuse에 쓰기.
    {
        device->EventBegin("Resolve Diffuse", cmd);
        device->BindComputeShader(&shaders[CSTYPE_VXGI_RESOLVE_DIFFUSE], cmd);

        device->Barrier(GPUBarrier::Image(&vxgi.texture_diffuse,
            ResourceState::SHADER_RESOURCE_COMPUTE, ResourceState::UNORDERED_ACCESS), cmd);

        PostProcess postprocess;
        postprocess.resolution = { (uint32_t)internalResolution.x, (uint32_t)internalResolution.y };
        postprocess.resolution_rcp = { 1.0f / postprocess.resolution.x, 1.0f / postprocess.resolution.y };
        device->PushConstants(&postprocess, sizeof(PostProcess), cmd);

        // t0 = depth, t1 = normal (G-Buffer에서)
        const GPUResource* srvs[] = { &depthBufferMain, &rtGbuffer[GBUFFER_NORMAL_ROUGHNESS] };
        device->BindResources(srvs, 0, 2, cmd);

        device->BindUAV(&vxgi.texture_diffuse, 0, cmd);

        device->Dispatch(
            (internalResolution.x + 7) / 8,
            (internalResolution.y + 7) / 8,
            1, cmd);

        device->Barrier(GPUBarrier::Image(&vxgi.texture_diffuse,
            ResourceState::UNORDERED_ACCESS, ResourceState::SHADER_RESOURCE_COMPUTE), cmd);

        device->EventEnd(cmd);
    }

    // ── [F] Resolve Specular (화면 픽셀 → 간접 specular 2D 출력) ─────────
    if (renderer::isVXGIReflectionsEnabled)
    {
        device->EventBegin("Resolve Specular", cmd);
        device->BindComputeShader(&shaders[CSTYPE_VXGI_RESOLVE_SPECULAR], cmd);

        device->Barrier(GPUBarrier::Image(&vxgi.texture_specular,
            ResourceState::SHADER_RESOURCE_COMPUTE, ResourceState::UNORDERED_ACCESS), cmd);

        // t0 = depth, t1 = normal+roughness G-Buffer
        device->BindUAV(&vxgi.texture_specular, 0, cmd);

        device->Dispatch(
            (internalResolution.x + 7) / 8,
            (internalResolution.y + 7) / 8,
            1, cmd);

        device->Barrier(GPUBarrier::Image(&vxgi.texture_specular,
            ResourceState::UNORDERED_ACCESS, ResourceState::SHADER_RESOURCE_COMPUTE), cmd);

        device->EventEnd(cmd);
    }

    // ── [G] 다음 프레임을 위해 clipmap 로테이션 ─────────────────────────
    vxgi.clipmap_to_update = (vxgi.clipmap_to_update + 1) % VXGI_CLIPMAP_COUNT;

    device->EventEnd(cmd);
    profiler::EndRange(prof_range);
}
```

---

## Step 9 — Renderer.cpp에서 Update_VXGI 호출

**위치**: `Renderer.cpp`에서 DDGI가 호출되는 곳 바로 아래

```cpp
// DDGI
if (renderer::isDDGIEnabled)
{
    Update_DDGI(cmd);
}

// VXGI  ← 여기에 추가
if (renderer::isVXGIEnabled)
{
    Update_VXGI(cmd);
}
```

**플래그 추가**: `Renderer.h`에:
```cpp
extern bool isVXGIEnabled;
extern bool isVXGIReflectionsEnabled;
```

`Renderer.cpp` 상단에:
```cpp
bool isVXGIEnabled = false;
bool isVXGIReflectionsEnabled = false;
```

---

## Step 10 — Lighting Pass에서 VXGI 결과 적용

voxelGI의 최종 목표: 간접광을 기존 lighting에 합산.

### shadingHF.hlsli (또는 동등 위치)
```hlsl
// VXGI 간접 조명 적용
#ifdef OPTION_BIT_VXGI_ENABLED
if (GetFrame().options & OPTION_BIT_VXGI_ENABLED)
{
    // texture_diffuse / texture_specular는 화면 해상도 2D 텍스처
    // 현재 픽셀의 UV로 샘플링
    float4 vxgi_diffuse  = vxgi_diffuse_texture.SampleLevel(sampler_point_clamp, screenUV, 0);
    float4 vxgi_specular = vxgi_specular_texture.SampleLevel(sampler_point_clamp, screenUV, 0);

    lighting.indirect.diffuse  += vxgi_diffuse.rgb;
    lighting.indirect.specular += vxgi_specular.rgb;
}
#endif
```

`texture_diffuse`와 `texture_specular`는 bindless 인덱스로 FrameCB에 추가하거나,
별도 texture slot으로 바인딩.

---

## 요약: 추가해야 할 파일/항목 목록

| # | 파일 | 작업 내용 |
|---|------|-----------|
| 1 | `Scene_Detail.h` | `struct VXGI vxgi;` GPU 리소스 정의 |
| 2 | `ShaderInterop_VXGI.h` | 내용 확인 (이미 존재) |
| 3 | `SceneUpdate_Detail.cpp` | clipmap center/offset CPU 업데이트 |
| 4 | `SceneUpdate_Detail.cpp` | FrameCB.vxgi 필드 채우기 |
| 5 | `Shaders/CS/` | vxgi_temporalCS.hlsl 추가 |
| 6 | `Shaders/CS/` | vxgi_sdf_jumpfloodCS.hlsl 추가 |
| 7 | `Shaders/CS/` | vxgi_resolve_diffuseCS.hlsl 추가 |
| 8 | `Shaders/CS/` | vxgi_resolve_specularCS.hlsl 추가 |
| 9 | `ShaderLoader.h` | CSTYPE_VXGI_* 4개 추가 |
| 10 | `ShaderLoader.cpp` | 4개 CS 로드 코드 추가 |
| 11 | `Renderer.h` | isVXGIEnabled, isVXGIReflectionsEnabled |
| 12 | `Renderer.cpp` | isVXGIEnabled 변수 정의, Update_VXGI 호출 |
| 13 | `RenderPath3D_Detail.h` | Update_VXGI 선언, CreateVXGIResources 선언 |
| 14 | `GI_Detail.cpp` | CreateVXGIResources, Update_VXGI 구현 |
| 15 | `shadingHF.hlsli` | VXGI 간접광 합산 |

---

## 주의사항

### 1. output_atomic 크기 차이
WickedEngine의 `output_atomic`은 X= * res (6 face만). Temporal CS에서 i < 6이면 atomic에서 읽고, i >= 6 (cone)이면 aniso_colors[]에서 계산.
→ **texture_atomic 크기는 (6 * res) × res × (13 * res)**, 6 face 분량만.

### 2. texture_radiance 읽기/쓰기 동시성
Temporal CS에서 `input_previous_radiance`(SRV)와 `output_radiance`(UAV)가 **같은 텍스처**. 이는 각 스레드가 서로 다른 픽셀을 쓰므로 실제로 안전함 (HLSL spec에서 허용).
VizMotive에서 같은 텍스처를 SRV+UAV로 동시 바인딩 가능한지 확인 필요.

### 3. VoxelizerCB 바인딩 슬롯
`ShaderInterop_VXGI.h`의 `CBSLOT_RENDERER_VOXELIZER` 상수 확인. Voxelization VS/GS/PS, Temporal CS, SDF CS 모두 이 CB를 읽음.

### 4. Resolve CS의 G-Buffer 접근
`vxgi_resolve_diffuseCS`는 depth + normal 텍스처를 읽어 월드 위치를 재구성.  
VizMotive G-Buffer 텍스처 이름 확인:
- depth → `depthBufferMain` 또는 `depthBufferMain_Copy`  
- normal → `rtGbuffer[GBUFFER_NORMAL_ROUGHNESS]` (normal + roughness 패킹)

### 5. DrawSceneMeshes 시그니처
VizMotive에서 RENDERPASS_VOXELIZE 씬 드로우 호출 함수가 `DrawSceneMeshes`인지 `DrawScene`인지 확인. ShaderLoader.cpp:186에서 RENDERPASS_VOXELIZE 처리가 있으므로 기존 함수에 패스만 변경하면 됨.
