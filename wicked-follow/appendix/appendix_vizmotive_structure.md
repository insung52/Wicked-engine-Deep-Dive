# Appendix: VizMotive 엔진 전체 구조

---

## 솔루션 구성 (VizMotive2.sln)

VizMotive는 여러 프로젝트(`.vcxproj`)로 구성된 솔루션이다.
각 프로젝트가 하나의 DLL 또는 실행 파일을 만든다.

```
VizMotive2.sln
│
├── Engine Build
│   └── Engine_Windows.vcxproj     → Engine.dll
│       (Engine_SOURCE.vcxitems 공유 파일 포함)
│
├── Modules (Essential)
│   ├── GBackendDX12.vcxproj       → GBackendDX12.dll
│   ├── GBackendDX11.vcxproj       → GBackendDX11.dll
│   └── GBackendVulkan.vcxproj     → GBackendVulkan.dll
│
├── Shader Engines
│   └── ShaderEngine.vcxproj       → ShaderEngine.dll
│
├── Built-in Plugins
│   └── AssetIO.vcxproj            → AssetIO.dll
│
└── Examples (App Samples)
    ├── Sample01.vcxproj           → Sample01.exe
    ├── Sample02.vcxproj           → Sample02.exe
    └── ...
```

---

## 각 DLL의 역할

### Engine.dll (EngineCore/)

엔진의 핵심. 모든 공개 API와 렌더링 추상화 레이어를 담당한다.

```
EngineCore/
├── Utils/          → 유틸리티 (Allocator, JobSystem, Profiler 등)
├── Components/     → GPU 리소스 컴포넌트 (Texture, GPUBuffer 등)
├── HighAPIs/       → 외부에 공개되는 고수준 API (VzEngineAPIs, VzRenderer 등)
├── GBackend/       → 그래픽스 백엔드 추상화 인터페이스 (GBackend.h, GBackendDevice.h)
└── Common/         → 공통 타입 (Canvas, RenderPath 등)
```

`Texture`, `GPUBuffer` 같은 공개 리소스 타입이 여기 정의된다.
`internal_state`(불투명 포인터)를 통해 실제 DX12 구현과 연결한다.

### GBackendDX12.dll (GraphicsBackends/)

DX12 실제 구현. Engine.dll의 추상화 인터페이스를 구현한다.

```
GraphicsBackends/
├── GBackendDX12.cpp / .h         → DLL 진입점, GraphicsDevice 팩토리
├── GraphicsDevice_DX12.cpp / .h  → DX12 구현 본체
└── D3D12MemAlloc.cpp / .h        → D3D12 메모리 할당기
```

`Texture_DX12`, `Resource_DX12` 같은 내부 구현 타입이 여기 있다.
Engine.dll이 `GModuleLoader`를 통해 런타임에 로드한다.

### ShaderEngine.dll (EngineShaders/)

셰이더 렌더러. 실제 렌더 패스(DDGI, Shadow 등)의 셰이더 로직을 담당한다.

### AssetIO.dll (EnginePlugins/AssetIO/)

에셋 로딩 플러그인. FBX, glTF 등 파일 포맷 처리.

### app.exe (Examples/)

진입점. Engine.dll을 로드하고 씬을 구성하는 사용자 코드.

---

## EngineCore 내부 필터 구조

Visual Studio Solution Explorer에서 보이는 필터(폴더) 구조는
`Engine_SOURCE.vcxitems.filters`에 정의되어 있다.

### Utils (UTIL_EXPORT)

유틸리티 모음.

| 필터 | 의미 | 예시 파일 |
|------|------|-----------|
| `Public (Header Only)` | 헤더 온리. inline/template만 있어 DLL 경계 자유로움 | Spinlock.h, Timer.h, Color.h, vzMath.h, Allocator.h (현재) |
| `Public (APIs managed by core)` | 헤더에 선언, Engine.dll .cpp에 구현. dllexport로 공개 | JobSystem.h, Profiler.h, Backlog.h, Config.h |
| `Private` | Engine.dll 내부 구현. 외부 노출 안 됨 | ECS.h, Helpers2.h, JobSystem.cpp, Profiler.cpp |

**두 Public의 차이:**
- `Header Only`: 선언과 구현 모두 헤더에 있음. 호출하면 그 코드가 호출한 DLL 안에 컴파일됨 (cross-DLL 호출 없음)
- `APIs managed by core`: 헤더에 선언만. 실제 코드는 Engine.dll의 .cpp에 있음. 다른 DLL이 호출하면 cross-DLL 호출이 발생하지만, dllexport로 허용된 호출이므로 안전

### Components (CORE_EXPORT)

GPU 리소스 컴포넌트 구현.

| 필터 | 예시 |
|------|------|
| `Public (APIs)` | Components.h, GComponents.h — 외부에 공개되는 컴포넌트 타입 |
| `Private` / `Private\Component Detail` | TextureComponent.cpp, MaterialComponent.cpp 등 — 내부 구현 |

### HighAPIs (API_EXPORT)

앱이 직접 호출하는 고수준 API.

| 필터 | 예시 |
|------|------|
| `Public (APIs)` | VzEngineAPIs.h, VzRenderer.h 등 — 헤더 선언 |
| `Private (Manager)` | VzEngineManager.cpp — DeinitEngineLib 등 엔진 수명 관리 |
| `Private (Components)` | VzRenderer.cpp, VzTexture.cpp 등 — API 구현 |

### GBackend

그래픽스 백엔드 추상화 인터페이스. 헤더만 있고 구현은 각 GBackend*.dll에 있다.

| 파일 | 역할 |
|------|------|
| `GBackend.h` | `Texture`, `GPUBuffer`, `GraphicsDeviceChild` 등 공개 타입 정의 |
| `GBackendDevice.h` | `GraphicsDevice` 추상 인터페이스 (순수 가상 함수) |
| `GModuleLoader.h` | GBackend*.dll을 런타임에 로드하는 플러그인 로더 |

---

## DLL 간 의존 관계

```
app.exe
  ↓ 로드
Engine.dll  ←────────────────────────────────┐
  ├── GModuleLoader가 런타임에 로드:          │
  │     GBackendDX12.dll                     │ Engine.dll의 인터페이스를
  │     ShaderEngine.dll                     │ 구현 (GBackendDevice 상속)
  │     AssetIO.dll                          │
  └─────────────────────────────────────────►┘
```

- `app.exe`는 Engine.dll만 직접 링크한다. GBackendDX12.dll은 모른다.
- Engine.dll이 런타임에 `platform::LoadModule("GBackendDX12.dll")`로 로드한다 (`GModuleLoader.h` 참고).
- GBackendDX12.dll은 Engine.dll의 헤더(`GBackend.h`, `Allocator.h` 등)를 include해서 빌드된다.

---

## 핵심 데이터 흐름 예시: Texture 생성

```
app.exe:
  vzm::CreateTexture(...)  → Engine.dll 호출

Engine.dll:
  내부적으로 graphicsDevice->CreateTexture(...)  → GBackendDX12.dll 호출 (추상 인터페이스)

GBackendDX12.dll:
  Texture_DX12 생성
  shared_ptr<void> internal_state 생성 (allocator 포인터 직접 저장)
  return → Engine.dll로 전달

Engine.dll:
  Texture.internal_state = shared_ptr<void>{Texture_DX12 포인터, DX12.dll의 allocator}
  → 이 Texture 객체가 Engine.dll 안에서 복사/소멸되어도
    allocator 포인터가 DX12.dll을 직접 가리키므로 DLL 경계 문제 없음
```
