# Appendix: VizMotive 엔진 전체 구조

> 코드 분석 기준. 모든 내용은 실제 소스에서 확인한 것임.

---

## 1. DLL 구성

VizMotive는 **4종류의 DLL**로 구성된 플러그인 기반 엔진이다.

```
VizEngine.dll          ← 엔진 코어 (컴포넌트, 공개 API, 그래픽스 추상 인터페이스)
ShaderEngine.dll       ← 렌더링 파이프라인 구현 (어떻게 그릴지)
GBackendDX12.dll       ← DX12 API 래핑 (GPU에 명령 전달)
GBackendDX11.dll       ← DX11 API 래핑 (선택)
GBackendVulkan.dll     ← Vulkan API 래핑 (선택)
AssetIO.dll            ← 에셋 파일 임포터 (OBJ, PLY, STL, Gaussian Splat 등)
```

빌드 출력물 (Release 기준):

| DLL | 크기 | Import Lib |
|-----|------|-----------|
| VizEngine.dll | ~3.2 MB | VizEngine.lib (660 KB) |
| ShaderEngine.dll | ~1.3 MB | ShaderEngine.lib |
| GBackendDX12.dll | ~300 KB | GBackendDX12.lib (2.3 KB) |
| AssetIO.dll | ~5 MB | AssetIO.lib |
| dxcompiler.dll | ~21 MB | (외부 셰이더 컴파일러 도구, 직접 빌드 안 함) |

---

## 2. VS 솔루션 프로젝트 구조

```
VizMotive2.sln
├── Engine_Windows.vcxproj         → VizEngine.dll / VizEngined.dll(Debug)
│   └── 참조: Engine_SOURCE.vcxitems (소스 파일 목록)
│
├── GraphicsBackends/
│   ├── GBackendDX12.vcxproj       → GBackendDX12.dll
│   ├── GBackendDX11.vcxproj       → GBackendDX11.dll
│   └── GBackendVulkan.vcxproj     → GBackendVulkan.dll
│
├── ShaderEngine.vcxproj           → ShaderEngine.dll
│
├── AssetIO.vcxproj                → AssetIO.dll
│
└── Sample01~14.vcxproj            → 예제 실행파일들
```

**의존 관계 (링크 기준):**

```
GBackendDX12.dll  ──링크──▶ VizEngine.lib (import library)
ShaderEngine.dll  ──링크──▶ VizEngine.lib
AssetIO.dll       ──링크──▶ VizEngine.lib
Sample*.exe       ──링크──▶ VizEngine.lib
```

모든 DLL/exe가 VizEngine.lib(import library)를 링크한다.
VizEngine.dll이 제공하는 공개 심볼(컴포넌트 타입, UTIL_EXPORT 함수 등)을 직접 참조한다.

---

## 3. VizEngine.dll 내부 구조

**소스 위치:** `EngineCore/`
**소스 목록 관리:** `EngineCore/Engine_SOURCE.vcxitems` (Shared Items Project)

```
EngineCore/
├── CommonInclude.h                 공통 매크로, 수학 헬퍼
│
├── Common/                         엔진 기반 시스템
│   ├── Canvas.h/cpp                렌더 타겟 크기/DPI 추상화
│   ├── RenderPath.h                렌더 경로 기반 클래스 (추상)
│   ├── RenderPath2D.h/cpp          2D 렌더 경로
│   ├── RenderPath3D.h/cpp          3D 렌더 경로 (ShaderEngine이 구현체 제공)
│   ├── ResourceManager.h/cpp       리소스 캐싱/로딩 관리
│   └── Initializer.h/cpp           엔진 초기화 순서 관리
│
├── Components/                     ECS 컴포넌트 시스템
│   ├── Components.h                컴포넌트 타입 전방 선언
│   ├── GComponents.h               GPU 리소스 컴포넌트 (Texture, GPUBuffer 등)
│   ├── TextureComponent.cpp
│   ├── MaterialComponent.cpp
│   ├── RenderableComponent.cpp
│   ├── GeometryComponent.cpp
│   ├── CameraComponent.cpp
│   ├── AnimationComponent.cpp
│   └── Scene.cpp                   씬 그래프 (계층적 트랜스폼)
│
├── GBackend/                       그래픽스 추상화 레이어 (인터페이스 정의만)
│   ├── GBackend.h                  GPU 리소스 구조체 + 열거형
│   │                               (GPUBuffer, Texture, Shader, PipelineState 등)
│   ├── GBackendDevice.h            GraphicsDevice 순수 추상 클래스
│   │                               (CreateBuffer, CreateTexture, BeginCommandList 등)
│   └── GModuleLoader.h             DLL 런타임 로드 유틸리티
│                                   (GBackendLoader, GShaderEngineLoader)
│
├── HighAPIs/                       공개 C++ API (vzm 네임스페이스)
│   ├── VzEngineAPIs.h              InitEngineLib, DeinitEngineLib, New* 함수 선언
│   ├── VzEngineManager.cpp         위 함수들의 구현 + 컴포넌트 관리 테이블
│   ├── VzScene.h/cpp               씬 관리 API
│   ├── VzRenderer.h/cpp            렌더러 관리 API
│   ├── VzGeometry.h/cpp            메시 리소스 API
│   ├── VzMaterial.h/cpp            머티리얼 API
│   ├── VzTexture.h/cpp             텍스처 API
│   ├── VzActor.h/cpp               씬 노드 (계층 구조)
│   ├── VzCamera.h/cpp
│   ├── VzLight.h/cpp
│   └── VzAnimation.h/cpp
│
├── Utils/                          유틸리티 라이브러리
│   ├── Allocator.h/cpp             커스텀 shared_ptr (블록 풀링, UTIL_EXPORT)
│   ├── JobSystem.h/cpp             병렬 잡 시스템 (스레드 풀)
│   ├── ECS.h                       Entity Component System (64비트 VID 기반)
│   ├── Platform.h                  OS 추상화 (LoadLibrary/dlopen 래핑)
│   ├── Spinlock.h
│   ├── EventHandler.h/cpp
│   ├── Profiler.h/cpp
│   ├── Backlog.h/cpp               로깅
│   ├── Config.h/cpp
│   ├── vzMath.h                    수학 라이브러리
│   └── DirectXMath/                DirectX 수학 (벡터, 행렬, 충돌)
│
└── ThirdParty/                     외부 라이브러리 (헤더/소스 직접 포함)
    ├── stb_image.h/cpp
    ├── lodepng.h/cpp               PNG 코덱
    ├── meshoptimizer/              메시 최적화
    └── spdlog/                     로깅
```

---

## 4. ShaderEngine.dll 내부 구조

**소스 위치:** `EngineShaders/ShaderEngine/`

**실제 렌더링 파이프라인 로직**이 여기에 있다.
"어떤 오브젝트를 어떤 순서로 어떻게 그릴지"를 결정하는 DLL이다.

```
ShaderEngine/
├── ShaderEngine.h/cpp              플러그인 진입점 (Initialize, Deinitialize 등 export)
├── Renderer.h/cpp                  렌더러 전역 상수 및 핵심 렌더링 시스템
├── RenderPath3D_Detail.h/cpp       3D 렌더링 파이프라인 전체 (~93KB, ~1500줄)
├── RenderPath2D_Detail.cpp         2D 렌더링
├── SceneUpdate_Detail.cpp          씬 업데이트 (트랜스폼, 애니메이션, TLAS 등)
├── ShadowMap_Detail.cpp            그림자 맵
├── DeferredProcessing_Detail.cpp   디퍼드 처리
├── PostProcessingChain_Detail.cpp  포스트 프로세싱
├── GI_Detail.cpp                   글로벌 일루미네이션
├── GaussianSplatting_Detail.cpp    가우시안 스플래팅
├── DirectVolumes_Detail.cpp        볼류메트릭 렌더링
├── DebugRenderer.h/cpp             디버그 시각화
├── GPUBVH.h/cpp                    GPU BVH 빌드
├── ShaderLoader.h/cpp              셰이더 파일 로딩/캐싱
├── ShaderCompiler.h/cpp            HLSL 컴파일 (dxcompiler.dll 동적 로드)
├── Font.h/cpp                      폰트 렌더링
├── Image.h/cpp                     이미지 처리
├── TextureHelper.h/cpp             텍스처 유틸
└── SortLib.h/cpp                   GPU 정렬
```

**Renderer.h에 정의된 전역 포맷 상수 (예시):**

```cpp
FORMAT_depthbufferMain       = D32_FLOAT_S8X24_UINT
FORMAT_rendertargetMain      = R11G11B10_FLOAT
FORMAT_idbuffer              = R32_UINT
FORMAT_rendertargetShadowmap = R16G16B16A16_FLOAT
FORMAT_depthbufferShadowmap  = D16_UNORM
FORMAT_rendertargetEnvprobe  = R11G11B10_FLOAT
```

**ShaderEngine.dll이 외부에 export하는 함수들:**

```cpp
bool Initialize(GraphicsDevice*);
bool Deinitialize();
bool LoadRenderer();
bool ApplyConfiguration();
RenderPath3D* NewGRenderPath3D();
GScene*       NewGScene();
bool LoadShader(ShaderStage, Shader&, const std::string& name);
bool LoadShaders();
// + AddDeferredMIPGen, AddDeferredBlockCompression 등
```

---

## 5. GBackendDX12.dll 내부 구조

**소스 위치:** `GraphicsBackends/`

`GBackendDevice.h`의 `GraphicsDevice` 순수 가상 클래스에 대한 **DX12 구현체**다.
VizEngine.dll이 인터페이스를 정의하고, GBackendDX12.dll이 그 구현을 담는다.

```
GraphicsBackends/
├── GBackendDX12.h/cpp              플러그인 진입점 + 구현 연결
├── GraphicsDevice_DX12.h           GraphicsDeviceDX12 클래스 선언
├── GraphicsDevice_DX12.cpp         DX12 구현체
│   ├── CreateBuffer()
│   ├── CreateTexture()
│   ├── CreateShader()
│   ├── CreatePipelineState()
│   ├── BeginCommandList()
│   ├── SubmitCommandLists()
│   └── CopyAllocator               (UPLOAD 힙 → DEFAULT 힙 복사 전용 내부 시스템)
└── D3D12MemAlloc.h                 D3D12MA (DX12 메모리 서브할당 라이브러리)
```

**export 함수 (GBackendDX12.dll이 외부에 노출하는 것):**

```cpp
bool           Initialize(ValidationMode, GPUPreference);
bool           Deinitialize();
GraphicsDevice* GetGraphicsDevice();
```

---

## 6. AssetIO.dll 내부 구조

**소스 위치:** `EnginePlugins/AssetIO/`

모델/에셋 파일을 VizMotive 컴포넌트로 임포트하는 플러그인.

```
AssetIO/
├── ModelImporter_OBJ.cpp       Wavefront OBJ (tiny_obj_loader 사용)
├── ModelImporter_PLY.cpp       PLY 포맷
├── ModelImporter_STL.cpp       STL 포맷
├── ModelImporter_SPLAT.cpp     Gaussian Splatting 포맷
└── External/assimp/            Assimp 라이브러리 (CMake로 빌드 포함)
```

---

## 7. 플러그인 로딩 메커니즘

GBackend와 ShaderEngine은 **런타임에 동적 로드**된다.
컴파일 타임에 DX12/Vulkan을 결정하지 않고, `InitEngineLib()` 호출 시 인자로 선택한다.

**`EngineCore/Utils/Platform.h` — OS 추상화:**

```cpp
#ifdef _WIN32
    LoadLibraryA(name)
    GetProcAddress(handle, name)
#else
    dlopen(name, RTLD_LAZY)
    dlsym(handle, name)
#endif
```

**`EngineCore/GBackend/GModuleLoader.h` — 로더 구조체:**

```cpp
struct GBackendLoader {
    typedef bool(*PI_GraphicsInitializer)(ValidationMode, GPUPreference);
    typedef GraphicsDevice*(*PI_GetGraphicsDevice)();
    typedef bool(*PI_GraphicsDeinitializer)();

    PI_GraphicsInitializer   pluginInitializer;
    PI_GraphicsDeinitializer pluginDeinitializer;
    PI_GetGraphicsDevice     pluginGetDev;

    bool Init(const std::string& api) {
        // "DX12" → "GBackendDX12" 변환 후 LoadLibraryA
        // GetProcAddress로 위 함수 포인터 획득
    }
};

struct GShaderEngineLoader {
    PI_Initializer      pluginInitializer;
    PI_Deinitializer    pluginDeinitializer;
    PI_LoadRenderer     pluginLoadRenderer;
    PI_NewGRenderPath3D pluginNewGRenderPath3D;
    PI_NewGScene        pluginNewGScene;
    PI_LoadShader       pluginLoadShader;
    PI_LoadShaders      pluginLoadShaders;
    // + AddDeferredMIPGen 등 선택적 함수들
};
```

**`EngineCore/HighAPIs/VzEngineManager.cpp` — 실제 사용:**

```cpp
GBackendLoader    graphicsBackend;  // 전역
GShaderEngineLoader shaderEngine;  // 전역

bool InitEngineLib(...)
{
    graphicsBackend.Init("DX12");           // GBackendDX12.dll 로드
    graphicsBackend.pluginInitializer(...); // DX12 디바이스 초기화

    shaderEngine.Init("ShaderEngine");      // ShaderEngine.dll 로드
    shaderEngine.pluginInitializer(device); // 렌더러 초기화
}

bool DeinitEngineLib()
{
    // DestroyAll() → backend Deinit() → shaderEngine Deinit() 순서 보장
    vzcompmanager::DestroyAll();
    graphicsBackend.pluginDeinitializer();
    shaderEngine.pluginDeinitializer();
}
```

---

## 8. 공개 API (vzm 네임스페이스)

사용자가 VizMotive를 쓸 때 include하는 헤더: `EngineCore/HighAPIs/VzEngineAPIs.h`

```cpp
namespace vzm {

// 초기화
bool InitEngineLib(const ParamMap<std::string>& arguments);
bool DeinitEngineLib();

// 씬/렌더러
VzScene*    NewScene(const std::string& name);
VzRenderer* NewRenderer(const std::string& name);

// 리소스
VzGeometry* NewGeometry(const std::string& name);
VzMaterial* NewMaterial(const std::string& name);
VzTexture*  NewTexture(const std::string& name);

// 씬 노드
VzActor*           NewActorNode(const std::string& name, VID parentVid);
VzCamera*          NewCamera(const std::string& name, VID parentVid);
VzLight*           NewLight(const std::string& name, VID parentVid);
VzActorStaticMesh* NewActorStaticMesh(const std::string& name,
                       GeometryVID, MaterialVID, VID parentVid);
VzActorGSplat*     NewActorGSplat(...);
VzActorVolume*     NewActorVolume(...);

} // namespace vzm
```

**VID (Virtual ID) 시스템:**

모든 리소스는 64비트 VID로 식별된다.

```cpp
using VID         = uint64_t;
using SceneVID    = VID;
using RendererVID = VID;
using GeometryVID = VID;
using MaterialVID = VID;
using TextureVID  = VID;

// VzEngineManager.cpp 내부 관리 테이블
std::unordered_map<SceneVID,    VzScene*>    scenes;
std::unordered_map<RendererVID, VzRenderer*> renderers;
std::unordered_map<GeometryVID, VzGeometry*> geometries;
std::unordered_map<MaterialVID, VzMaterial*> materials;
std::unordered_map<TextureVID,  VzTexture*>  textures;
```

---

## 9. 레이어 구조 요약

```
┌──────────────────────────────────────────────────────────────┐
│           사용자 코드 (Sample01.exe 등)                        │
│           #include "VzEngineAPIs.h"                          │
└─────────────────────────┬────────────────────────────────────┘
                          │ vzm::InitEngineLib / NewScene 등
┌─────────────────────────▼────────────────────────────────────┐
│                     VizEngine.dll                             │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │  HighAPIs    │  │  Components  │  │  Utils             │  │
│  │  (vzm API)   │  │  (ECS, Scene)│  │  (JobSystem,       │  │
│  └──────────────┘  └──────────────┘  │   Allocator, ...)  │  │
│  ┌──────────────────────────────┐    └────────────────────┘  │
│  │  GBackend (인터페이스 정의)   │                             │
│  │  GraphicsDevice (순수 가상)  │                             │
│  └────────────┬─────────────────┘                            │
└───────────────│──────────────────────────────────────────────┘
   런타임 로드   │ LoadLibraryA + GetProcAddress
    ┌───────────┴──────────────┐
    ▼                          ▼
┌──────────────┐    ┌────────────────────────────────┐
│ShaderEngine  │    │  GBackendDX12.dll              │
│  .dll        │    │  (GraphicsDevice 구현체)        │
│              │    │                                │
│ Renderer.cpp │    │  CreateBuffer / CreateTexture  │
│ RenderPath3D │    │  BeginCommandList              │
│ ShadowMap    │───▶│  SubmitCommandLists            │
│ PostProcess  │    │  CopyAllocator                 │
│ GI, DDGI ... │    └────────────────┬───────────────┘
└──────────────┘                     │
                                     ▼
                              DirectX 12 API
                                   GPU
```

**각 레이어 역할 요약:**

| 레이어 | DLL | 역할 |
|--------|-----|------|
| 공개 API | VizEngine.dll | 사용자 진입점, 리소스 생명주기 |
| 컴포넌트/씬 | VizEngine.dll | 씬 데이터 보관 (무엇이 있는가) |
| 그래픽스 인터페이스 | VizEngine.dll | GraphicsDevice 추상 정의 |
| 렌더링 파이프라인 | ShaderEngine.dll | 어떻게 그릴지 결정 (컬링, 패스 구성) |
| DX12 래핑 | GBackendDX12.dll | GPU에 명령 전달 (DX12 API 호출) |

---

## 10. 한 프레임의 렌더링 흐름 (개요)

```
1. VzRenderer::Render() 호출  →  VizEngine.dll (HighAPIs)
           ↓
2. shaderEngine.pluginLoadRenderer() 등 함수 포인터 호출  →  ShaderEngine.dll 진입
           ↓
3. RenderPath3D_Detail.cpp 실행  →  ShaderEngine.dll
   - SceneUpdate: 트랜스폼, TLAS 빌드
   - Shadow Pass, GBuffer Pass, Lighting Pass
   - PostProcess, GI, ...
   - 각 패스에서 device->BeginCommandList() / device->Draw() 등 호출
           ↓
4. GraphicsDevice_DX12 함수 실행  →  GBackendDX12.dll
   - DX12 커맨드리스트에 명령 기록
           ↓
5. device->SubmitCommandLists()  →  GPU 큐에 제출 → GPU 실행
```

---

## 11. 셰이더 컴파일

```
ShaderEngine.dll
  └─ ShaderCompiler.cpp
       ├─ 런타임에 dxcompiler.dll 동적 로드 (HLSL6 → DXIL)
       └─ 또는 d3dcompiler_47.dll          (HLSL5 → DXBC)

지원 포맷:
  HLSL5 → DXBC   (DX11 호환)
  HLSL6 → DXIL   (DX12 기본)
  HLSL6 → SPIR-V (Vulkan)
  HLSL6 → Xbox Series 전용 포맷
  HLSL6 → PS5 전용 포맷
```
