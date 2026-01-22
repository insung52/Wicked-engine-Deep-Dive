# Wicked Engine 변경사항 - 2025년 3월

## 목차
- [Vehicle physics](#1-vehicle-physics-1053)
- [Capsule shadows](#2-capsule-shadows-1055)
- [Fix Lua stack overflow vulnerability](#3-fix-lua-stack-overflow-vulnerability-1054)
- [capsuleshadow and character controller improvements](#4-capsuleshadow-and-character-controller-improvements)
- [spherical harmonics](#5-spherical-harmonics-1056)
- [ddgi backface pulling leak improvement](#6-ddgi-backface-pulling-leak-improvement)
- [physics: added ragdoll ghost mode](#7-physics-added-ragdoll-ghost-mode)
- [added vehicle metadata type](#8-added-vehicle-metadata-type)
- [linux: add missing break](#9-linux-add-missing-break-to-get-rid-of-error-message)
- [Github virus detected fix](#10-github-virus-detected-fix-pr-1058)
- [require cmake 3.19](#11-require-cmake-319)
- [wiScene: support address sanitizer](#12-wiscene-support-address-sanitizer)
- [vkWaitForFences timeout](#13-vkwaitforfences-timeout)
- [raytracing fixes](#14-raytracing-fixes)
- [Linux hang fix](#15-linux-hang-fix)
- [linux mouse movement fix](#16-linux-mouse-movement-fix)
- [linux vulkan improvement](#17-linux-vulkan-improvement)
- [linux texture streaming fix](#18-linux-texture-streaming-fix)
- [linux: disabled vulkan resource aliasing](#19-linux-disabled-vulkan-resource-aliasing)
- [added some safety GPU resource clears](#20-added-some-safety-gpu-resource-clears)
- [vulkan: extra attempt to find sparse queue flag](#21-vulkan-extra-attempt-to-find-sparse-queue-flag)
- [dx12 and vulkan additional safety updates](#22-dx12-and-vulkan-additional-safety-updates)
- [logfile writing thread safety](#23-logfile-writing-thread-safety)
- [vulkan: use dedicated queue for resource inits](#24-vulkan-use-dedicated-queue-for-resource-inits-if-available)
- [vulkan timeout reporting and additional safety](#25-vulkan-timeout-reporting-and-additional-safety)
- [debug shader tessellation compatibility fix](#26-debug-shader-tessellation-compatibility-fix-1067)
- [vulkan: gpu queue sync delayed](#27-vulkan-gpu-queue-sync-is-delayed-into-next-frame-submit)
- [fixed small dx12 gpuvalidation issue](#28-fixed-small-dx12-gpuvalidation-issue)
- [dx12 separated cpu and gpu fences](#29-dx12-separated-cpu-and-gpu-fences-for-improved-safety)
- [xbox gpu hang fix](#30-xbox-gpu-hang-fix)
- [added playstation touchpad button](#31-added-playstation-touchpad-button)
- [model importer fix](#32-model-importer-fix)
- [added culling for 3D fonts](#33-added-culling-for-3d-fonts)
- [Jolt: update to v5.3.0](#34-jolt-update-to-v530-1070)
- [added crossfade fade type](#35-added-crossfade-fade-type)
- [improved stencil composition for renderpath2D](#36-improved-stencil-composition-for-renderpath2d)
- [fix: don't use image at startup](#37-fix-dont-use-image-at-startup-while-not-initialized)
- [HDR improvements](#38-hdr-improvements)
- [renderpath2d improvement: resizebuffers](#39-renderpath2d-improvement-resizebuffers-will-be-called-in-prerender)
- [fix: sometimes renderpath2d resizebuffers must happen in update](#40-fix-sometimes-renderpath2d-resizebuffers-must-happen-in-update)
- [reduced physics locking](#41-reduced-physics-locking)
- [job system updates](#42-job-system-updates)
- [fix void character collision](#43-fix-void-character-collision-when-layer-is-not-specified)
- [editor: negative axes darkening control](#44-editor-negative-axes-darkening-control-1071)
- [don't crash when calling OverrideWehicleWheelTransforms](#45-dont-crash-when-calling-overridewehiclewheeltransforms-with-invalid-physics-scene)
- [dx12 and vulkan improvements](#46-dx12-and-vulkan-improvements)
- [camera render texture can be set to material](#47-camera-render-texture-can-be-set-to-material)
- [added multithreading to camera component render](#48-added-multithreading-to-camera-component-render)
- [camera feed mipmapping; stencil scaling check; nan fixup](#49-camera-feed-mipmapping-stencil-scaling-check-nan-fixup-other-improvements)
- [linux string format compile fix](#50-linux-string-format-compile-fix)
- [added light color mask texture support](#51-added-light-color-mask-texture-support)
- [LOD and subset management fixes](#52-lod-and-subset-management-fixes)
- [hdr window resize fix and cursor fix](#53-hdr-window-resize-fix-and-cursor-fix)
- [winapi cursor state improvements](#54-winapi-cursor-state-improvements)
- [fix in imgui docking sample](#55-fix-in-imgui-docking-sample)
- [linux: added editor drag and drop file support](#56-linux-added-editor-drag-and-drop-file-support)
- [added height field physics shape type](#57-added-height-field-physics-shape-type)

---

## 1. Vehicle physics (#1053)
**커밋:** `4357dc5b`
**날짜:** 2025년 3월 초

### 설명
Jolt 물리 엔진을 사용한 차량(Vehicle) 물리 시뮬레이션 기능 추가.

### 주요 변경사항
- `wiPhysics_Jolt.cpp`: 579줄 이상의 차량 물리 코드 추가
- `wiPhysics.h`: 차량 관련 인터페이스 23줄 추가
- `wiPhysics_BindLua.cpp/h`: Lua 바인딩 추가 (111줄)
- `RigidBodyWindow.cpp`: 에디터 UI 754줄 대폭 확장
- `wiScene_Components.h`: `VehicleComponent` 구조체 추가 (72줄)
- `Content/models/vehicle_test.wiscene`: 테스트용 차량 씬 추가

### 변경 파일 (29개)
- Editor: `RigidBodyWindow`, `Editor.cpp`, `HierarchyWindow`, `ComponentsWindow` 등
- WickedEngine: `wiPhysics*`, `wiScene*`, `wiRenderPath3D` 등

---

## 2. Capsule shadows (#1055)
**커밋:** `545e859c`

**VizMotive 미구현**

### 설명
캡슐 형태의 그림자를 통해 캐릭터(Humanoid) 등의 부드러운 접지 그림자를 구현하는 기능입니다.

기존 섀도우 맵 방식과 달리, 캡슐 기하학적 형태를 이용한 Ambient Occlusion 스타일의 soft shadow입니다.

- **[capsule shadow 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/shadow/capsule_shadow.md)**

### 주요 변경사항
- `capsuleShadowHF.hlsli`: 새로운 캡슐 그림자 셰이더 헤더 (69줄)
- `shadingHF.hlsli`: 캡슐 그림자 통합 (52줄 추가)
- `lightCullingCS.hlsl`: 라이트 컬링에 캡슐 그림자 처리 추가 (33줄)
- `wiRenderer.cpp`: 렌더러 통합 (404줄 수정)
- `HumanoidWindow`: 휴머노이드 캡슐 그림자 설정 UI
- `MaterialWindow`: 머티리얼별 캡슐 그림자 설정

### 변경 파일 (23개)
- 셰이더: `capsuleShadowHF.hlsli`, `shadingHF.hlsli`, `lightCullingCS.hlsl` 등
- WickedEngine: `wiRenderer`, `wiScene_Components`, `wiPrimitive` 등

### VizMotive Engine 비교 분석

**결론: VizMotive Engine에 완전히 구현되어 있지 않음!**

| 구성요소 | Wicked | VizMotive |
|---------|--------|-----------|
| `capsuleShadowHF.hlsli` 파일 | ✅ | ✅ |
| `directionalOcclusionCapsule()` 함수 | ✅ | ✅ |
| TiledLighting 적용 코드 | ✅ | ❌ |
| `forces()` capsule collider 처리 | ✅ | ❌ |

VizMotive에는 셰이더 파일(`capsuleShadowHF.hlsli`)만 있고, 실제로 `TiledLighting`에서 적용하는 통합 코드가 없음:
```hlsl
// Wicked Engine만 있는 코드 (shadingHF.hlsli:562-621)
if ((GetFrame().options & OPTION_BIT_CAPSULE_SHADOW_ENABLED) && !forces().empty())
{
    half4 occlusion_cone = half4(surface.dominant_lightdir, GetCapsuleShadowAngle());
    // ... capsule shadow 계산 ...
    surface.occlusion *= capsuleshadow;  // VizMotive에 없음!
}
```

**VizMotive에 이 기능을 추가하려면 이 커밋을 참고해야 함.**

---

## 3. Fix Lua stack overflow vulnerability (#1054)
**커밋:** `fdaf5ed7`

### 설명
Lua 스택 오버플로우 취약점 수정. 보안상 중요한 패치.

### 주요 변경사항
- `LUA/ldebug.c`: 스택 오버플로우 방지 로직 추가 (5줄)

### 변경 파일 (1개)
```
WickedEngine/LUA/ldebug.c | 6 +++++-
```

---

## 4. capsuleshadow and character controller improvements
**커밋:** `f3cb98b6`

### 설명
캡슐 그림자 및 캐릭터 컨트롤러 개선 사항.

### 주요 변경사항
- `wiRenderer_BindLua.cpp`: Lua 바인딩 확장 (42줄)
- `character_controller.lua`: 캐릭터 컨트롤러 스크립트 개선
- `ShaderInterop_Renderer.h`: 셰이더 인터롭 수정

### 변경 파일 (6개)

---

## 5. spherical harmonics (#1056)
**커밋:** `808421db`

**[x] VizMotive에 이미 구현됨**

### 설명
구면 조화 함수(Spherical Harmonics) 기반 GI 시스템 대폭 개선. DDGI와 Surfel GI 모두에 적용.

기존 6*6 Octahedral 텍스처를 저장하는 대신 Spherical Harmonics L1 으로 저장.

추가로 GI에서도 Specular 반사 구현.

### 주요 변경사항
- `SH_Lite.hlsli`: 새로운 SH 라이브러리 (1029줄!)
- `ddgi_raytraceCS.hlsl`: DDGI 레이트레이싱 개선 (161줄 수정)
- `surfel_raytraceCS.hlsl`: Surfel GI 레이트레이싱 개선 (167줄 수정)
- `ShaderInterop_DDGI.h`: DDGI 인터롭 구조체 대폭 변경 (94줄)
- `wiVoxelGrid.cpp/h`: 복셀 그리드 SH 통합
- Archive 버전 업데이트

### 변경 파일 (43개)
- 셰이더 대량 수정: DDGI, Surfel, RT 관련 셰이더들
- WickedEngine: `wiScene`, `wiRenderer`, `wiHelper` 등

### VizMotive Engine 비교 분석

**결론: VizMotive Engine에 이미 완전히 구현되어 있음**

#### 1. SH 기반 DDGI 저장 방식 (구현됨)

| 기능 | Wicked Engine | VizMotive Engine |
|------|---------------|------------------|
| SH 기반 irradiance 저장 | `SH::L1_RGB` | `SH::L1_RGB` |
| 프로브 버퍼 구조 | `DDGIProbe.radiance` | `DDGIProbe.radiance` |
| Irradiance 계산 | `SH::CalculateIrradiance()` | `SH::CalculateIrradiance()` |
| Dominant Light Direction | `SH::ApproximateDirectionalLight()` | `SH::ApproximateDirectionalLight()` |
| SH 라이브러리 | `SH_Lite.hlsli` (MJP) | `SH_Lite.hlsli` (동일) |

#### 2. GI Specular 반사 (구현됨)

두 엔진 모두 `shadingHF.hlsli`에서 동일한 방식으로 구현:

```hlsl
// Wicked & VizMotive 공통 구현 (shadingHF.hlsli)
if (GetScene().ddgi.probe_buffer >= 0)
{
    half3 irradiance = ddgi_sample_irradiance(surface.P, surface.N,
                         surface.dominant_lightdir, surface.dominant_lightcolor);

    // Indirect Specular: DLD 방향으로 BRDF 계산
    SurfaceToLight surface_to_light;
    surface_to_light.create(surface, surface.dominant_lightdir);
    lighting.indirect.specular += BRDF_GetSpecular(surface, surface_to_light)
                                  * surface.dominant_lightcolor;
}
```

**동작 원리:**
1. `ddgi_sample_irradiance()` 호출 시 SH에서 Dominant Light Direction(DLD) + Color 추출
2. DLD 방향으로 `SurfaceToLight` 생성
3. `BRDF_GetSpecular()`로 스펙큘러 계산
4. `lighting.indirect.specular`에 추가

VizMotive 는 이후 커밋 ## 14. raytracing fixes 를 적용한 상태로, 일부 최적화가 추가되었습니다.

---

## 6. ddgi backface pulling leak improvement
**커밋:** `3dfd5787`

**[x] VizMotive에 이미 구현됨 (DDGI만)**

### 설명
DDGI 프로브가 벽 내부에 있을 때 빛이 벽을 통과해 새어나가는 현상(light leak) 개선.

### 핵심 변경
1. **Face Culling 비활성화**: `RAY_FLAG_CULL_BACK/FRONT_FACING_TRIANGLES` 주석 처리
2. **Backface Hit 시 Depth 축소**: 프로브를 안쪽으로 밀어넣음
```hlsl
if (surface.IsBackface()) {
    hit_depth *= 0.9; // DDGI (Surfel은 0.5)
}
```

### 주요 변경사항
- `ddgi_raytraceCS.hlsl`: 백페이스 처리 개선
- `surfel_raytraceCS.hlsl`: Surfel GI도 동일 개선
- `wiScene.cpp`: 씬 처리 수정

### VizMotive 비교
- DDGI: ✅ 동일하게 구현됨
- Surfel GI: ❌ VizMotive에 Surfel GI 미구현

### 변경 파일 (5개)

---

## 7. physics: added ragdoll ghost mode
**커밋:** `b87a31dd`

### 설명
래그돌에 고스트 모드 추가. 래그돌이 다른 물체와 충돌하지 않고 통과할 수 있음.

### 주요 변경사항
- `wiPhysics_Jolt.cpp`: 고스트 모드 구현 (15줄)
- `wiPhysics_BindLua.cpp`: Lua 바인딩 (21줄)
- Scripting API 문서 업데이트

### 변경 파일 (7개)

---

## 8. added vehicle metadata type
**커밋:** `b937bab7`

### 설명
에디터에서 차량 메타데이터 타입 및 시각화 추가.

### 주요 변경사항
- `dummy_vehicle.h`: 차량 더미 비주얼라이저 데이터 (5214줄!)
- `DummyVisualizer.cpp/h`: 차량 더미 시각화
- `MetadataWindow.cpp`: 메타데이터 창에 차량 타입 추가

### 변경 파일 (11개)

---

## 9. linux: add missing break to get rid of error message
**커밋:** `0b464d3f`

### 설명
Linux에서 스트리밍 스레드 우선순위 설정 시 누락된 break문 추가. 에러 메시지 제거.

### 주요 변경사항
- `wiJobSystem.cpp`: switch문에 break 추가

### 변경 파일 (1개)
```
WickedEngine/wiJobSystem.cpp | 1 +
```

---

## 10. Github virus detected fix (PR #1058)
**커밋:** `a9fc1007`, `2d5e4648`, `79be0153`, `10ae5a57`

### 설명
GitHub Actions 빌드에서 바이러스 오탐지 문제 해결을 위해 UPX 압축 제거.

- UPX (Ultimate Packer for eXecutables) = 실행 파일 압축 도구

### 주요 변경사항
- `.github/workflows/*.yml`: UPX 관련 코드 모두 제거

### 변경 파일
- `build-pr.yml`, `build-nightly.yml`, `build.yml`

---

## 11. require cmake 3.19
**커밋:** `b0390c1f`

### 설명
CMake 최소 버전을 3.19로 상향. 해당 버전 이전에서 지원하지 않는 기능 사용.

### 주요 변경사항
- 모든 CMakeLists.txt 파일에서 버전 요구사항 업데이트

### 변경 파일 (12개)

---

## 12. wiScene: support address sanitizer
**커밋:** `aca529bc`

### 설명
Address Sanitizer 지원을 위한 wiScene 수정.

Address Sanitizer (ASan)란?

메모리 버그 탐지 도구:
- 버퍼 오버플로우
- Use-after-free
- 초기화되지 않은 메모리 접근
- 메모리 릭

#### 컴파일 시 활성화
g++ -fsanitize=address -g main.cpp

### 주요 변경사항
- `wiScene.cpp`: ASan 호환성 코드 추가 (11줄)

### 변경 파일 (1개)

---

## 13. vkWaitForFences timeout
**커밋:** `2ef4f769`

### 설명
Vulkan vkWaitForFences 타임아웃 값 수정.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: 타임아웃 값 조정

### 변경 파일 (1개)

---

## 14. raytracing fixes
**커밋:** `0070de66`

**[x] VizMotive에 이미 구현됨**

### 설명
레이트레이싱 관련 버그 수정. 두 가지 핵심 문제 해결:
1. Ray Direction 정규화 누락
2. half 정밀도로 인한 GI 품질 저하

### 핵심 변경 1: Ray Direction 정규화

DXR/Vulkan Ray Tracing에서 `RayDesc.Direction`은 **반드시 단위 벡터(길이=1)**여야 함.

**변경 전 (버그):**
```hlsl
newRay.Direction = L;
newRay.Direction = L + max3(surface.sss);
ray.Direction = sample_hemisphere_cos(surface.N, rng);
```

**변경 후 (수정):**
```hlsl
newRay.Direction = normalize(L);
newRay.Direction = normalize(L + max3(surface.sss));
ray.Direction = normalize(sample_hemisphere_cos(surface.N, rng));
```

정규화 안 된 방향 벡터 사용 시:
- `TMax` 거리 계산 오류 → 잘못된 히트 판정
- 특히 `L + max3(surface.sss)` (SSS 오프셋) 결과는 절대 정규화되지 않음

### 핵심 변경 2: half → float 정밀도

**변경 전:**
```hlsl
half energy_conservation = 0.95;
half3 ddgi = ddgi_sample_irradiance(...);
```

**변경 후:**
```hlsl
float energy_conservation = 0.95;
float3 ddgi = ddgi_sample_irradiance(...);
```

| 타입 | 비트 | 정밀도 | 문제점 |
|-----|-----|-------|-------|
| `half` | 16-bit | ~3자리 | 색상 밴딩, 에너지 누적 오차 |
| `float` | 32-bit | ~7자리 | 없음 |

### 주요 변경사항
- `ddgi_raytraceCS.hlsl`: normalize 3곳 + float 2곳
- `surfel_raytraceCS.hlsl`: normalize 2곳 + float 1곳
- `renderlightmapPS.hlsl`: normalize 1곳

### VizMotive 비교

| 수정 사항 | Wicked | VizMotive |
|----------|--------|-----------|
| `normalize(L)` | ✅ | ✅ |
| `normalize(raydir)` | ✅ | ✅ |
| `normalize(L + max3(surface.sss))` | ✅ | ✅ |
| `float energy_conservation` | ✅ | ✅ |
| `float3 ddgi` | ✅ | ✅ |

**결론: 모든 수정사항이 VizMotive에 이미 반영되어 있음.**

### 변경 파일 (4개)

---

## 15. Linux hang fix
**커밋:** `bf27a9fb`

### 설명
Linux에서 발생하는 hang(멈춤) 문제 수정. 그래픽 디바이스와 렌더패스 관련 수정.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: Vulkan 동기화 수정 (50줄)
- `wiGraphicsDevice_DX12.cpp`: DX12도 관련 수정 (23줄 감소)
- `wiRenderPath3D.cpp`: 렌더패스 수정 (72줄)
- `wiRenderPath2D.cpp`: 수정
- `Sponza.wiscene` 파일명 대소문자 수정

### 변경 파일 (10개)

---

## 16. linux mouse movement fix
**커밋:** `42364108`

### 설명
Linux에서 마우스 움직임 관련 버그 수정.

### 주요 변경사항
- `wiInput.cpp`: 불필요한 코드 제거 (15줄 감소)

### 변경 파일 (2개)

---

## 17. linux vulkan improvement
**커밋:** `452649d5`

### 설명
Linux Vulkan 관련 개선.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: 불필요한 코드 제거

### 변경 파일 (1개)

---

## 18. linux texture streaming fix
**커밋:** `13c9cc9b`

### 설명
Linux 텍스처 스트리밍 버그 수정.

### 주요 변경사항
- `CommonInclude.h`: 플랫폼별 매크로 확장 (24줄 추가)
- `wiResourceManager.cpp`: 리소스 매니저 수정

### 변경 파일 (3개)

---

## 19. linux: disabled vulkan resource aliasing
**커밋:** `d763593d`

### 설명
Linux에서 Vulkan 리소스 앨리어싱 비활성화. 간헐적 unknown error 발생 방지.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: 리소스 앨리어싱 비활성화 (12줄 추가)

### 변경 파일 (2개)

---

## 20. added some safety GPU resource clears
**커밋:** `872cac1c`

**[x] VizMotive가 더 발전된 상태**

### 설명
GPU 리소스 안전 초기화 추가. 초기화되지 않은 GPU 메모리 사용으로 인한 아티팩트 및 GPU hang 방지.

### 핵심 문제
- GPU 메모리는 기본적으로 초기화되지 않음
- 이전 프레임 데이터나 랜덤 값이 남아있을 수 있음
- 특히 Xbox에서 GPU hang 발생 가능

### 주요 변경사항

**1. DDGI 버퍼 제로 초기화:**
```cpp
wi::vector<uint8_t> zerodata;
zerodata.resize(buf.size);
device->CreateBuffer(&buf, zerodata.data(), &ddgi.ray_buffer);  // 0으로 초기화
```

**2. 렌더링 전 ClearUAV 추가:**
```cpp
device->ClearUAV(res.depthbuffer, 0, cmd);
device->ClearUAV(res.lineardepth, 0, cmd);
device->ClearUAV(&res.texture_normals, 0, cmd);
device->ClearUAV(&res.texture_roughness, 0, cmd);
// ...
```

**3. Barrier 코드 단순화**

### VizMotive 비교

| 기능 | Wicked (커밋 #20) | VizMotive |
|------|------------------|-----------|
| DDGI 버퍼 제로 초기화 | ✅ 수동 zerodata | ✅ `CreateBufferZeroed()` 헬퍼 |
| ClearUAV 사용 | ✅ 다수 추가 | ✅ 일부 사용 |
| 제로 버퍼 헬퍼 함수 | ❌ (커밋 #30에서 추가) | ✅ 이미 있음 |

**VizMotive는 `CreateBufferZeroed()` 헬퍼 함수가 이미 있어 더 깔끔한 코드:**
```cpp
// VizMotive (SceneUpdate_Detail.cpp)
device->CreateBufferZeroed(&buf, &ddgi.rayBuffer);
device->CreateBufferZeroed(&buf, &ddgi.varianceBuffer);
device->CreateBufferZeroed(&buf, &ddgi.probeBuffer);
```

### 변경 파일 (3개)
- `wiRenderer.cpp`: ClearUAV 추가 + Barrier 단순화
- `wiScene.cpp`: DDGI 버퍼 제로 초기화

---

## 21. vulkan: extra attempt to find sparse queue flag
**커밋:** `eb511c64`

### 설명
Vulkan에서 sparse queue 플래그 탐색 로직 개선.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: 큐 탐색 로직 개선 (46줄)
- `wiGraphicsDevice_Vulkan.h`: 관련 변수 추가 (4줄)

### 변경 파일 (3개)

---

## 22. dx12 and vulkan additional safety updates
**커밋:** `3f5a5cc6`

### 설명
DX12와 Vulkan 안전성 관련 추가 업데이트.

### 주요 변경사항
- `wiGraphicsDevice_DX12.cpp`: DX12 안전성 코드 (64줄)
- `wiGraphicsDevice_Vulkan.cpp`: Vulkan 안전성 코드 (207줄 수정)
- `wiScene.cpp`: 씬 관련 수정

### 변경 파일 (6개)

---

## 23. logfile writing thread safety
**커밋:** `7c1b43fe`

### 설명
로그 파일 쓰기 스레드 안전성 보장.

### 주요 변경사항
- `wiBacklog.cpp`: 스레드 안전 로직 추가 (2줄)

### 변경 파일 (1개)

---

## 24. vulkan: use dedicated queue for resource inits if available
**커밋:** `8beca2b8`

### 설명
Vulkan에서 가능한 경우 리소스 초기화용 전용 큐 사용.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: 전용 큐 사용 로직 (45줄)
- `wiGraphicsDevice_Vulkan.h`: 관련 멤버 추가 (3줄)

### 변경 파일 (3개)

---

## 25. vulkan timeout reporting and additional safety
**커밋:** `85f6abd7`

### 설명
Vulkan 타임아웃 리포팅 및 추가 안전성 조치.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: 타임아웃 리포팅 (163줄 수정)
- `wiGraphicsDevice_Vulkan.h`: 관련 수정

### 변경 파일 (3개)

---

## 26. debug shader tessellation compatibility fix #1067
**커밋:** `8b9e2ce6`

### 설명
디버그용 심플 셰이더에서 테셀레이션 사용 시 호환성 문제 수정.

### 변경 내용
```hlsl
// objectPS_simple.hlsl, objectVS_simple.hlsl
#define OBJECTSHADER_USE_NORMAL // tessellation compat!
```

테셀레이션 셰이더는 노멀 데이터가 필요한데, 심플 셰이더에 이 매크로가 없어서 호환 안 됨.

### 변경 파일 (2개)

---

## 27. vulkan: gpu queue sync is delayed into next frame submit
**커밋:** `8e3e68c2`

### 설명
Vulkan GPU 큐 동기화를 다음 프레임 제출로 지연.

### 주요 변경사항
- `wiGraphicsDevice_Vulkan.cpp`: 동기화 타이밍 변경 (12줄)

### 변경 파일 (2개)

---

## 28. fixed small dx12 gpuvalidation issue
**커밋:** `39847415`

### 설명
DX12 GPU Validation 오류 수정. 리소스 상태 전환(Barrier) 누락 및 불필요한 플래그 제거.

- `BUFFERTYPE_INDIRECT_DEBUG_0`: `UpdateBuffer()` 전에 `COPY_SRC` 상태로 전환하는 Barrier 추가
- `BUFFERTYPE_INDIRECT_DEBUG_1`: 불필요한 `COPY_SRC` 플래그 제거

실제 동작에는 영향 없고, GPU Validation 디버깅 시 오류 메시지 제거용.

### 변경 파일 (1개)

---

## 29. dx12 separated cpu and gpu fences for improved safety
**커밋:** `93ebdaa6`

**[ ] VizMotive 미적용 - 적용 고려 필요**

### 설명
DX12에서 CPU/GPU 펜스를 분리하여 경쟁 조건(Race Condition) 방지.

### 배경: Fence란?
GPU-CPU 동기화 메커니즘. GPU 작업 완료 시 펜스 값 증가(Signal), CPU나 다른 큐가 대기(Wait).

### 문제점: 단일 펜스 사용
```cpp
// 기존: 하나의 펜스를 두 용도로 사용
frame_fence[BUFFERCOUNT][QUEUE_COUNT];

// 용도 1: CPU 대기 (값: 1)
// 용도 2: GPU 큐 간 대기 (값: FRAMECOUNT)
// → 경쟁 조건 발생 가능
```

### 해결: 펜스 분리
```cpp
// 변경 후
frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];  // CPU 대기 전용
frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];  // GPU 큐 간 대기 전용

// Signal
queue.queue->Signal(frame_fence_cpu[...].Get(), 1);
queue.queue->Signal(frame_fence_gpu[...].Get(), FRAMECOUNT);
```

### VizMotive 비교

| 항목 | Wicked (커밋 후) | VizMotive |
|------|-----------------|-----------|
| 펜스 구조 | `frame_fence_cpu` + `frame_fence_gpu` | `frame_fence` (단일) |
| 경쟁 조건 | 해결 | 잠재적 문제 |

**VizMotive는 단일 펜스 구조 유지 중. DX12에서 간헐적 hang 발생 시 이 변경 적용 고려.**

### 주요 변경사항
- `wiGraphicsDevice_DX12.cpp`: 펜스 분리 구현 (41줄)
- `wiGraphicsDevice_DX12.h`: 펜스 멤버 2개로 분리

### 변경 파일 (3개)

---

## 30. xbox gpu hang fix
**커밋:** `dc532889`
**[ ] CHECKOUT 필요 - Xbox GPU hang 수정**

### 설명
Xbox에서 GPU hang 문제 수정. indirect 버퍼 생성 시 초기화 추가. 제로 초기화 버퍼 생성 헬퍼 추가.

### 주요 변경사항
- `wiGraphicsDevice.h`: `CreateZeroedBuffer` 헬퍼 함수 추가 (10줄)
- `wiScene.cpp`: 버퍼 초기화 로직 개선 (40줄)
- `wiRenderer.cpp`: indirect 버퍼 초기화
- `wiEmittedParticle.cpp`, `wiHairParticle.cpp`, `wiOcean.cpp`: 관련 수정

### 변경 파일 (8개)

---

## 31. added playstation touchpad button
**커밋:** `58eea921`

### 설명
PlayStation 터치패드 버튼 지원 추가.

### 주요 변경사항
- `wiInput.cpp`: 터치패드 버튼 처리 (3줄)
- `wiInput.h`: 버튼 enum 추가

### 변경 파일 (2개)

---

## 32. model importer fix
**커밋:** `b588a8b7`

### 설명
FBX/GLTF 임포터에서 무효한 휴머노이드 제거 시 루프 버그 수정.

```cpp
// 변경 전: 인덱스로 제거 (버그)
scene.humanoids.Remove(i);

// 변경 후: 엔티티로 제거 + 루프 인덱스 처리 수정
scene.humanoids.Remove(scene.humanoids.GetEntity(i));
```

컬렉션 순회 중 삭제 시 인덱스 처리 오류로 인한 잘못된 동작 수정.

### 변경 파일 (2개)

---

## 33. added culling for 3D fonts
**커밋:** `18d0e2ab`

### 설명
3D 폰트에 대한 컬링 기능 추가. 성능 향상.

### 주요 변경사항
- `wiSpriteFont.cpp`: 3D 폰트 컬링 구현 (31줄)
- `wiSpriteFont.h`: 관련 인터페이스 (3줄)
- `wiRenderer.cpp`: 렌더러 통합 (37줄)
- `wiScene.cpp/h`: 씬 통합

### 변경 파일 (7개)

---

## 34. Jolt: update to v5.3.0 (#1070)
**커밋:** `506749de`

### 설명
Jolt Physics 엔진을 v5.3.0으로 업데이트. 많은 변경 사항 포함.

### 주요 변경사항
- Jolt 라이브러리 전체 업데이트 (182개 파일, +5702/-2412줄)
- 새로운 기능:
  - `HashTable.h`: 새로운 해시 테이블 구현 (872줄)
  - `BinaryHeap.h`: 바이너리 힙 구현 (96줄)
  - `CharacterID.h`: 캐릭터 ID 시스템 (98줄)
  - `BVec16.h/inl`: 16비트 벡터 타입 (276줄)
  - `SimShapeFilter.h/SimShapeFilterWrapper.h`: 시뮬레이션 셰이프 필터
- 삭제된 기능:
  - `TriangleGrouper*`: 삭제
  - `TriangleSplitterFixedLeafSize`, `TriangleSplitterLongestAxis`, `TriangleSplitterMorton`: 삭제
- CharacterVirtual 대폭 개선 (335줄 수정)
- PhysicsSystem 개선 (189줄 수정)
- HeightFieldShape 개선 (232줄 수정)

### 변경 파일 (182개)

---

## 35. added crossfade fade type
**커밋:** `652fe0da`

### 설명
크로스페이드 전환 타입 추가. 씬 전환 시 부드러운 크로스페이드 효과.

### 주요 변경사항
- `wiApplication.cpp`: 크로스페이드 구현 (181줄 수정)
- `wiApplication.h`: 관련 인터페이스
- `wiFadeManager.h`: 페이드 매니저 확장 (19줄)
- `wiApplication_BindLua.cpp`: Lua 바인딩 (79줄)
- `wiGraphicsDevice.h`: 그래픽 디바이스 헬퍼 추가 (8줄)

### 변경 파일 (14개)

---

## 36. improved stencil composition for renderpath2D
**커밋:** `0da0f3b5`

**[ ] VizMotive 미구현**

### 설명
RenderPath2D에서 스텐실 합성 기능 개선. 스케일링 및 MSAA 지원 추가.

### 새로 추가된 함수
```cpp
// 입력 마스크를 현재 렌더패스 스텐실로 스케일링
void ScaleStencilMask(const Viewport& vp, const Texture& input, CommandList cmd);

// 뎁스 스텐실에서 스텐실을 R8_UINT 텍스처로 추출
void ExtractStencil(const Texture& input_depthstencil, const Texture& output, CommandList cmd);
```

### 셰이더 변경
- `extractStencilBitPS.hlsl`: 새 셰이더 - 스텐실 비트 → 컬러 이미지 추출
- `copyStencilBitPS.hlsl`: MSAA 지원 + UV 기반 스케일링 추가

### 배경: Vulkan 호환성
Vulkan에서 `CopyTexture(COLOR → STENCIL)` 미지원으로 셰이더 기반 워크어라운드 필요.

### VizMotive 비교

| 기능 | Wicked | VizMotive |
|------|--------|-----------|
| `ScaleStencilMask()` | ✅ | ❌ |
| `ExtractStencil()` | ✅ | ❌ |
| 스텐실 스케일링 | ✅ | ❌ |
| MSAA 스텐실 지원 | ✅ | ❌ |

**VizMotive의 RenderPath2D에는 스텐실 합성 관련 코드가 없음.**

### 변경 파일 (15개)

---

## 37. fix: don't use image at startup while not initialized
**커밋:** `4f42c93e`

### 설명
시작 시 초기화되지 않은 이미지 사용 방지.

### 주요 변경사항
- `wiApplication.cpp`: 초기화 체크 추가 (10줄)

### 변경 파일 (1개)

---

## 38. HDR improvements
**커밋:** `4f503da8`

**[ ] VizMotive 미적용**

### 설명
HDR10 디스플레이에서 UI 블렌딩이 올바르게 작동하도록 color space 변환 파이프라인 개선.

### 배경 지식: Color Space와 블렌딩

#### 1. 왜 Linear Space에서 블렌딩해야 하는가?

```
sRGB (감마 보정된 색) → 모니터가 기대하는 형식
Linear (물리적으로 정확한 빛) → 수학적 연산이 올바른 공간
```

**문제 예시: 50% 투명도로 흰색과 검정색 블렌딩**
```
sRGB 공간에서:  (1.0 + 0.0) / 2 = 0.5 (sRGB) → 실제로는 21.8% 밝기로 보임 (너무 어두움)
Linear 공간에서: (1.0 + 0.0) / 2 = 0.5 (Linear) → 50% 밝기로 올바르게 보임
```

그래서 블렌딩은 **반드시 Linear space에서** 해야 자연스러움.

#### 2. HDR10 디스플레이의 색 공간

```
일반 모니터:  sRGB (Rec.709 색역, 감마 2.2 커브)
HDR10 모니터: Rec.2020 색역 + ST.2084 PQ 커브 (더 넓은 색역, 더 밝은 밝기)
```

UI 텍스처/폰트는 보통 **sRGB로 저작**됨. HDR10 출력 시 변환 필요:
1. sRGB → Linear (감마 제거)
2. Rec.709 → Rec.2020 (색역 변환)
3. Linear → PQ 커브 적용 (ST.2084)

#### 3. 기존 문제: if-else if 구조

```hlsl
// 기존 코드 (VizMotive 현재 상태)
if (HDR10_플래그) {
    // 바로 HDR10으로 변환 후 출력
    color = REC709toREC2020(color);
    color = ApplyPQCurve(color);
}
else if (LINEAR_플래그) {
    // Linear로만 변환
    color = RemoveSRGBCurve(color);
}
```

**문제점:**
- HDR10 모드에서 UI 요소 A, B를 블렌딩할 때:
  - A: sRGB → HDR10 (PQ 커브 적용됨)
  - B: sRGB → HDR10 (PQ 커브 적용됨)
  - 블렌딩: **비선형 PQ 공간에서 발생** → 색이 부자연스러움

```
[UI A (sRGB)] → [HDR10 변환] → 출력
                      ↓
              [블렌딩] ← 비선형 공간! 잘못됨
                      ↑
[UI B (sRGB)] → [HDR10 변환] → 출력
```

### 핵심 변경 1: Color Space 변환 분기 분리

**변경 후 (Wicked Engine):**
```hlsl
// 별도 if 브랜치 → 순차 적용 가능
if (LINEAR_플래그) {
    color = RemoveSRGBCurve(color);  // 1단계: Linear로 변환
    color *= hdr_scaling;
}

if (HDR10_플래그) {
    color = REC709toREC2020(color);  // 2단계: HDR10으로 변환
    color = ApplyPQCurve(color);
}
```

**이제 가능한 시나리오:**

| 플래그 조합 | 동작 | 용도 |
|------------|------|------|
| LINEAR만 | sRGB → Linear | 중간 버퍼에 Linear로 출력 (블렌딩용) |
| HDR10만 | 직접 HDR10 변환 | 이미 Linear인 입력을 HDR10으로 |
| 둘 다 | sRGB → Linear → HDR10 | 완전한 파이프라인 |

### 핵심 변경 2: 중간 렌더 타겟 (rendertargetPreHDR10)

```
[올바른 파이프라인]

[UI A (sRGB)] → [Linear 변환] ─┐
                               ├→ [Linear 버퍼에서 블렌딩] → [HDR10 변환] → 최종 출력
[UI B (sRGB)] → [Linear 변환] ─┘
```

- `rendertargetPreHDR10`: Linear space 중간 버퍼
- 모든 UI를 Linear space에서 먼저 합성
- 최종적으로 한 번에 HDR10 변환

### 핵심 변경 3: Premultiplied Alpha

```hlsl
// fontPS.hlsl에 추가됨
color.rgb *= color.a; // premultiplied blending
```

**일반 Alpha 블렌딩:**
```
result = src.rgb * src.a + dst.rgb * (1 - src.a)
```

**Premultiplied Alpha 블렌딩:**
```
// src.rgb가 이미 alpha와 곱해져 있음
result = src.rgb + dst.rgb * (1 - src.a)
```

**왜 HDR에서 중요한가?**

일반 알파 블렌딩은 Linear space에서 반투명 가장자리에 **검은 테두리(dark halo)** 발생:
- 반투명 픽셀: `rgb=(1,1,1), a=0.1` (거의 투명한 흰색)
- sRGB에서 블렌딩하면 괜찮아 보이지만
- Linear에서 블렌딩하면 가장자리가 어두워짐

Premultiplied alpha는 이 문제를 해결함.

### VizMotive 비교

| 항목 | Wicked Engine | VizMotive |
|------|--------------|-----------|
| Color space 분기 구조 | 별도 `if` (순차 적용) | `if-else if` (택일) |
| fontPS premultiplied | ✅ `color.rgb *= color.a` | ❌ 없음 |
| 중간 렌더 타겟 (`rendertargetPreHDR10`) | ✅ | ❌ 없음 |

**VizMotive에서 HDR10 블렌딩 품질 개선 시 이 변경 적용 필요.**

### 주요 변경사항
- `wiApplication.cpp`: HDR 처리 개선 (151줄)
- `fontPS.hlsl`: 폰트 셰이더 HDR 지원 (12줄)
- `imagePS.hlsl`: 이미지 셰이더 개선 (38줄)
- `downsample4xCS.hlsl`: 다운샘플링 개선 (10줄)
- `wiGraphics.h`: HDR 관련 타입 추가 (1줄)
- DX12/Vulkan: HDR 지원 관련 수정

### 변경 파일 (14개)

---

## 39. renderpath2d improvement: resizebuffers will be called in prerender
**커밋:** `8182c33a`

### 설명
`ResizeBuffers()` 호출 시점을 `Update()` → `PreRender()`로 이동.

### 변경 이유
| 단계 | 역할 |
|------|------|
| `Update()` | 로직 업데이트 (게임 로직, UI 상태) |
| `PreRender()` | **렌더링 준비** (버퍼 생성, 리사이즈) |

렌더 버퍼 생성은 렌더링 준비 작업이므로 `PreRender()`가 더 적절.

### 안전장치
```cpp
void RenderPath2D::Render() const {
    if (!rtFinal.IsValid()) {
        assert(0);  // PreRender 호출 안 했으면 경고
        const_cast<RenderPath2D*>(this)->PreRender();  // fallback
    }
}
```

### 변경 파일 (5개)

---

## 40. fix: sometimes renderpath2d resizebuffers must happen in update
**커밋:** `249caeca`

### 설명
일부 경우 RenderPath2D ResizeBuffers가 Update에서 발생해야 하는 문제 수정.

### 주요 변경사항
- `wiRenderPath2D.cpp`: 조건부 처리 추가 (6줄)
- `wiRenderPath3D.cpp`: 관련 수정 (14줄)

### 변경 파일 (2개)

---

## 41. reduced physics locking
**커밋:** `9e5ae4a6`

### 설명
물리 시스템 락킹 감소로 성능 향상.

### 주요 변경사항
- `wiPhysics_Jolt.cpp`: 락킹 최적화 (406줄 수정)
- `wiScene_Components.h`: 컴포넌트 수정 (48줄)

### 변경 파일 (4개)

---

## 42. job system updates
**커밋:** `658697e2`

### 설명
잡 시스템 업데이트 및 개선.

### 주요 변경사항
- `wiJobSystem.cpp`: 잡 시스템 개선 (62줄)
- `wiJobSystem.h`: 인터페이스 수정 (5줄)
- `wiLoadingScreen.cpp`: 로딩 스크린 수정

### 변경 파일 (4개)

---

## 43. fix void character collision when layer is not specified
**커밋:** `ece6e894`

### 설명
레이어가 지정되지 않은 경우 캐릭터 충돌이 무효화되는 버그 수정.

### 주요 변경사항
- `wiScene.cpp`: 레이어 기본값 처리 (7줄)

### 변경 파일 (2개)

---

## 44. editor: negative axes darkening control #1071
**커밋:** `f1ce71c5`

### 설명
에디터에서 음수 축 어둡게 표시 제어 기능 추가.

### 주요 변경사항
- `GeneralWindow.cpp`: UI 추가 (15줄)
- `Translator.cpp`: 트랜슬레이터 수정 (21줄)

### 변경 파일 (4개)

---

## 45. don't crash when calling OverrideWehicleWheelTransforms with invalid physics scene
**커밋:** `26c9fc3e`

### 설명
잘못된 물리 씬에서 OverrideWehicleWheelTransforms 호출 시 크래시 방지.

### 주요 변경사항
- `wiPhysics_Jolt.cpp`: null 체크 추가 (2줄)

### 변경 파일 (1개)

---

## 46. dx12 and vulkan improvements
**커밋:** `99c82676`

### 설명
DX12 및 Vulkan 개선 사항.

### 주요 변경사항
- `wiGraphicsDevice_DX12.cpp`: DX12 개선 (86줄 수정)
- `wiGraphicsDevice_Vulkan.cpp`: Vulkan 개선 (28줄)

### 변경 파일 (5개)

---

## 47. camera render texture can be set to material
**커밋:** `b1b708d2`
**[ ] CHECKOUT 필요 - Major 신규 기능**

### 설명
카메라 렌더 텍스처를 머티리얼에 설정할 수 있는 기능 추가. 보안 카메라, 거울 등 구현 가능.

### 주요 변경사항
- `wiRenderPath3D.cpp`: 카메라 렌더 텍스처 기능 구현 (197줄)
- `CameraComponentWindow.cpp`: 에디터 UI (142줄)
- `MaterialWindow.cpp`: 머티리얼에 카메라 텍스처 설정 (35줄)
- `wiScene_Components.h`: 관련 컴포넌트 추가 (15줄)
- `wiScene_Serializers.cpp`: 직렬화 지원 (20줄)

### 변경 파일 (14개)

---

## 48. added multithreading to camera component render
**커밋:** `98099891`

### 설명
카메라 컴포넌트 렌더링에 멀티스레딩 추가.

### 주요 변경사항
- `wiRenderPath3D.cpp`: 멀티스레드 렌더링 (178줄 수정)
- `wiScene_Components.h`: 스레드 안전 관련 추가 (1줄)

### 변경 파일 (4개)

---

## 49. camera feed mipmapping; stencil scaling check; nan fixup; other improvements
**커밋:** `591104d1`

### 설명
카메라 피드 밉매핑, 스텐실 스케일링 체크, NaN 수정 등 여러 개선 사항.

### 주요 변경사항
- `wiRenderPath3D.cpp`: 다양한 개선 (193줄)
- `wiScene.cpp`: 씬 관련 수정 (27줄)
- `volumetricLight_SpotPS.hlsl`: NaN 수정 (15줄)
- `meshlet_prepareCS.hlsl`: 메시렛 수정
- Lua 바인딩 업데이트

### 변경 파일 (12개)

---

## 50. linux string format compile fix
**커밋:** `528977b0`

### 설명
Linux 문자열 포맷 컴파일 오류 수정.

### 주요 변경사항
- `wiScene.cpp`: 포맷 문자열 수정 (1줄)

### 변경 파일 (1개)

---

## 51. added light color mask texture support
**커밋:** `d1578b01`

### 설명
라이트 컬러 마스크 텍스처 지원 추가. 라이트에 패턴/쿠키 텍스처 적용 가능.

### 주요 변경사항
- `LightWindow.cpp`: 에디터 UI (11줄)
- `lightingHF.hlsli`: 라이팅 셰이더 수정 (21줄)
- `globals.hlsli`: 전역 셰이더 수정 (25줄)
- `volumetricLight_PointPS.hlsl`: 포인트 라이트 볼류메트릭 (9줄)
- `volumetricLight_SpotPS.hlsl`: 스팟 라이트 볼류메트릭 (12줄)
- `wiMath.h`: 수학 헬퍼 추가 (17줄)
- `wiScene.cpp`: 씬 처리 (15줄)

### 변경 파일 (12개)

---

## 52. LOD and subset management fixes
**커밋:** `60d82edb`

### 설명
LOD 및 서브셋 관리 버그 수정.

### 주요 변경사항
- `wiScene_Components.cpp`: 컴포넌트 수정 (86줄)
- `wiScene_Components.h`: 인터페이스 추가 (9줄)
- `MeshWindow.cpp`: 에디터 수정 (25줄)
- `wiScene_BindLua.cpp`: Lua 바인딩 수정 (31줄)

### 변경 파일 (5개)

---

## 53. hdr window resize fix and cursor fix
**커밋:** `29fe3b75`

### 설명
HDR 윈도우 리사이즈 및 커서 관련 버그 수정.

### 주요 변경사항
- `wiApplication.cpp`: 수정 (2줄)
- `main_Windows.cpp`: 불필요한 코드 제거 (3줄)

### 변경 파일 (2개)

---

## 54. winapi cursor state improvements
**커밋:** `4235e2c6`

### 설명
WinAPI 커서 상태 관리 개선.

### 주요 변경사항
- `main_Windows.cpp`: 커서 상태 관리 (19줄)
- `wiInput.cpp`: 입력 처리 수정 (5줄)
- `wiInput.h`: 인터페이스 추가 (3줄)

### 변경 파일 (3개)

---

## 55. fix in imgui docking sample
**커밋:** `9a084b58`

### 설명
ImGui Docking 샘플 버그 수정.

### 주요 변경사항
- `Example_ImGui_Docking.cpp`: 샘플 수정 (7줄)

### 변경 파일 (1개)

---

## 56. linux: added editor drag and drop file support
**커밋:** `76549fa8`

### 설명
Linux 에디터에서 파일 드래그 앤 드롭 지원 추가.

### 주요 변경사항
- `main_SDL2.cpp`: SDL2 드래그 앤 드롭 처리 (4줄)

### 변경 파일 (1개)

---

## 57. added height field physics shape type
**커밋:** `bc128fe6`

### 설명
높이 필드(Height Field) 물리 셰이프 타입 추가. 터레인 물리에 유용.

### 주요 변경사항
- `wiPhysics_Jolt.cpp`: 높이 필드 셰이프 구현 (37줄)
- `wiTerrain.cpp`: 터레인 통합 (12줄)
- `wiPrimitive.cpp/h`: 프리미티브 지원 추가 (6줄)
- `wiScene_Components.h`: 셰이프 타입 추가 (1줄)
- `RigidBodyWindow.cpp`: 에디터 UI (1줄)

### 변경 파일 (7개)

---

## Checkout 추천 목록

다음 커밋들은 major 변경사항이거나 치명적인 버그 수정이므로 직접 테스트해볼 것을 추천합니다:

| 순번 | 커밋 | 제목 | 이유 |
|------|------|------|------|
| 1 | `4357dc5b` | Vehicle physics (#1053) | Major 신규 기능 - 차량 물리 |
| 2 | `545e859c` | Capsule shadows (#1055) | Major 렌더링 기능 |
| 3 | `fdaf5ed7` | Fix Lua stack overflow vulnerability (#1054) | 치명적 보안 버그 수정 |
| 5 | `808421db` | spherical harmonics (#1056) | Major GI 시스템 개선 |
| 15 | `bf27a9fb` | Linux hang fix | 치명적 버그 수정 (Linux) |
| 30 | `dc532889` | xbox gpu hang fix | 치명적 버그 수정 (Xbox) |
| 34 | `506749de` | Jolt: update to v5.3.0 (#1070) | Major 의존성 업데이트 |
| 47 | `b1b708d2` | camera render texture can be set to material | Major 신규 기능 |

### Checkout 방법 (참고)
```bash
# 특정 커밋 checkout (예: Vehicle physics)
git checkout 4357dc5b

# Remove special character 커밋 적용 (사용자 환경용)
git cherry-pick <remove-special-character-commit-hash>

# 다시 master로 돌아오기
git checkout master
```

---

## 3월 변경사항 요약

### 신규 기능
- **차량 물리** (Vehicle Physics)
- **캡슐 그림자** (Capsule Shadows)
- **구면 조화 함수 기반 GI** (Spherical Harmonics)
- **카메라 렌더 텍스처를 머티리얼에 적용**
- **크로스페이드 전환 효과**
- **라이트 컬러 마스크 텍스처**
- **높이 필드 물리 셰이프**
- **3D 폰트 컬링**
- **래그돌 고스트 모드**
- **PlayStation 터치패드 버튼 지원**
- **Linux 에디터 드래그 앤 드롭**

### 버그 수정
- **Lua 스택 오버플로우 취약점** (보안)
- **Linux hang 문제**
- **Xbox GPU hang 문제**
- **Linux 마우스 움직임**
- **Linux 텍스처 스트리밍**
- **모델 임포터 (FBX/GLTF)**
- **HDR 윈도우 리사이즈**
- **캐릭터 충돌 레이어 기본값**
- **LOD/서브셋 관리**

### 성능 개선
- **물리 락킹 감소**
- **잡 시스템 업데이트**
- **카메라 컴포넌트 멀티스레딩**
- **RenderPath2D 스텐실 합성 개선**

### 플랫폼/인프라
- **Jolt Physics v5.3.0 업데이트**
- **CMake 3.19 필수**
- **GitHub Actions UPX 제거** (바이러스 오탐지 해결)
- **DX12/Vulkan 안전성 업데이트 다수**
- **Address Sanitizer 지원**
