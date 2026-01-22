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
**[ ] CHECKOUT 필요 - Major 신규 기능**

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
**[ ] CHECKOUT 필요 - Major 렌더링 기능**

### 설명
캡슐 형태 오브젝트(캐릭터 등)에 대한 그림자 기능 추가. 캐릭터의 부드러운 그림자 표현에 유용.

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

---

## 3. Fix Lua stack overflow vulnerability (#1054)
**커밋:** `fdaf5ed7`
**[x] CHECKOUT 필요 - 치명적 보안 버그 수정**

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

**[ ] CHECKOUT 필요 - Major GI 시스템 개선**

### 설명
구면 조화 함수(Spherical Harmonics) 기반 GI 시스템 대폭 개선. DDGI와 Surfel GI 모두에 적용.

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

---

## 6. ddgi backface pulling leak improvement
**커밋:** `3dfd5787`

**[ ] CHECKOUT 필요 - Major GI 시스템 개선**

### 설명
DDGI 백페이스 풀링 시 발생하는 빛 누수(leak) 현상 개선.

### 주요 변경사항
- `ddgi_raytraceCS.hlsl`: 백페이스 처리 개선
- `surfel_raytraceCS.hlsl`: Surfel GI도 동일 개선
- `wiScene.cpp`: 씬 처리 수정

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

**[ ] CHECKOUT 필요 - 시스템 개선**

### 설명
레이트레이싱 관련 버그 수정.

### 주요 변경사항
- `ddgi_raytraceCS.hlsl`: DDGI RT 수정
- `surfel_raytraceCS.hlsl`: Surfel RT 수정
- `renderlightmapPS.hlsl`: 라이트맵 렌더링 수정

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

### 설명
GPU 리소스 안전 초기화 추가.

### 주요 변경사항
- `wiRenderer.cpp`: 대폭 리팩토링 (332줄 수정, 실제로는 감소)
- `wiScene.cpp`: 씬 관련 안전 초기화

### 변경 파일 (3개)

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
디버그 셰이더의 테셀레이션 호환성 수정.

### 주요 변경사항
- `objectPS_simple.hlsl`: 수정 (1줄)
- `objectVS_simple.hlsl`: 수정 (1줄)

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
DX12 GPU validation 관련 작은 이슈 수정.

### 주요 변경사항
- `wiRenderer.cpp`: 수정 (3줄)

### 변경 파일 (1개)

---

## 29. dx12 separated cpu and gpu fences for improved safety
**커밋:** `93ebdaa6`

### 설명
DX12에서 CPU/GPU 펜스 분리로 안전성 향상.

### 주요 변경사항
- `wiGraphicsDevice_DX12.cpp`: 펜스 분리 구현 (41줄)
- `wiGraphicsDevice_DX12.h`: 관련 멤버 수정

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
FBX, GLTF 모델 임포터 버그 수정.

### 주요 변경사항
- `ModelImporter_FBX.cpp`: FBX 임포터 수정 (9줄)
- `ModelImporter_GLTF.cpp`: GLTF 임포터 수정 (9줄)

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
**[ ] CHECKOUT 필요 - Major 의존성 업데이트**

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

### 설명
RenderPath2D에서 스텐실 합성 개선.

### 주요 변경사항
- `wiRenderPath2D.cpp`: 대폭 리팩토링 (216줄 수정)
- `wiRenderer.cpp`: 스텐실 관련 함수 추가 (228줄)
- `wiRenderer.h`: 새로운 인터페이스 (14줄)
- `extractStencilBitPS.hlsl`: 새로운 셰이더 (12줄)
- `copyStencilBitPS.hlsl`: 개선 (37줄)

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

### 설명
HDR 관련 개선 사항.

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
RenderPath2D의 ResizeBuffers를 Update 대신 PreRender에서 호출하도록 변경.

### 주요 변경사항
- `wiRenderPath2D.cpp`: 호출 시점 변경 (26줄)
- `wiApplication.cpp`: 관련 수정 (40줄)

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
