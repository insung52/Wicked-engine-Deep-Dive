# VizMotive Engine - Metal Graphics Backend 구현 가이드

## 목차
1. [그래픽스 백엔드란?](#1-그래픽스-백엔드란)
2. [VizMotive 엔진의 백엔드 아키텍처](#2-vizmotive-엔진의-백엔드-아키텍처)
3. [Metal 백엔드 파일 구조](#3-metal-백엔드-파일-구조)
4. [플러그인 인터페이스](#4-플러그인-인터페이스)
5. [GraphicsDevice 인터페이스](#5-graphicsdevice-인터페이스)
6. [현재 구현 상태](#6-현재-구현-상태)
7. [빌드 방법](#7-빌드-방법)
8. [다음 단계](#8-다음-단계)

---

## 1. 그래픽스 백엔드란?

### 개념
**그래픽스 백엔드**는 게임 엔진과 GPU 사이의 **추상화 계층**입니다.

```
┌─────────────────────────────────────────────────────────┐
│                    게임/애플리케이션                       │
├─────────────────────────────────────────────────────────┤
│                     렌더링 엔진                           │
├─────────────────────────────────────────────────────────┤
│                  GraphicsDevice (추상)                    │
├──────────────┬──────────────┬──────────────┬────────────┤
│  DX12 백엔드  │  DX11 백엔드  │ Vulkan 백엔드 │ Metal 백엔드 │
├──────────────┴──────────────┴──────────────┴────────────┤
│                      GPU 하드웨어                         │
└─────────────────────────────────────────────────────────┘
```

### 왜 필요한가?
- **크로스 플랫폼**: Windows(DX12), macOS(Metal), Linux(Vulkan) 지원
- **코드 재사용**: 렌더링 로직은 한 번만 작성, 백엔드만 교체
- **유지보수**: API별 코드가 분리되어 관리 용이

### 주요 그래픽스 API
| API | 플랫폼 | 특징 |
|-----|--------|------|
| DirectX 12 | Windows, Xbox | 저수준, 고성능 |
| DirectX 11 | Windows | 중간 수준, 호환성 좋음 |
| Vulkan | Windows, Linux, Android | 저수준, 크로스플랫폼 |
| **Metal** | **macOS, iOS** | **Apple 전용, 저수준** |

---

## 2. VizMotive 엔진의 백엔드 아키텍처

### 플러그인 방식
VizMotive는 그래픽스 백엔드를 **동적 라이브러리(DLL/dylib)**로 분리합니다.

```
엔진 실행 시:
1. 설정에서 API 선택 (예: "METAL")
2. GBackendMetal.dylib 동적 로드
3. Initialize() 함수 호출
4. GraphicsDevice 포인터 획득
5. 렌더링 시작
```

### 장점
- 엔진 재컴파일 없이 백엔드 교체 가능
- 플랫폼별 빌드 분리
- 런타임에 백엔드 선택 가능

### 모듈 로더 (GModuleLoader.h)
```cpp
// 엔진이 백엔드를 로드하는 방식
struct GBackendLoader {
    bool Init(const std::string& api) {
        if (api == "DX12")   moduleName = "GBackendDX12";
        if (api == "DX11")   moduleName = "GBackendDX11";
        if (api == "VULKAN") moduleName = "GBackendVulkan";
        if (api == "METAL")  moduleName = "GBackendMetal";  // ← 추가됨

        // 함수 포인터 로드
        pluginInitializer = LoadModule("Initialize");
        pluginGetDev = LoadModule("GetGraphicsDevice");
        pluginDeinitializer = LoadModule("Deinitialize");
    }
};
```

---

## 3. Metal 백엔드 파일 구조

```
GraphicsBackends/
├── GBackendMetal.vcxproj      # Windows Visual Studio 프로젝트 (스텁용)
├── CMakeLists.txt             # CMake 빌드 설정 (크로스플랫폼)
│
├── GBackendMetal.h            # 플러그인 인터페이스 헤더
├── GBackendMetal.cpp          # Windows용 스텁 구현
├── GBackendMetal.mm           # macOS용 진입점 (Objective-C++)
│
├── GraphicsDevice_Metal.h     # Metal 디바이스 클래스 선언
└── GraphicsDevice_Metal.mm    # Metal 디바이스 구현
```

### 파일별 역할

| 파일 | 역할 |
|------|------|
| `GBackendMetal.h` | DLL export 함수 선언 (Initialize, GetGraphicsDevice, Deinitialize) |
| `GBackendMetal.mm` | 플러그인 진입점, GraphicsDevice_Metal 인스턴스 관리 |
| `GraphicsDevice_Metal.h` | GraphicsDevice 추상 클래스를 상속받은 Metal 구현 클래스 |
| `GraphicsDevice_Metal.mm` | 실제 Metal API 호출 코드 |
| `CMakeLists.txt` | macOS에서 빌드하기 위한 CMake 설정 |

### .mm 확장자란?
- **Objective-C++** 파일
- C++과 Objective-C를 혼합 사용 가능
- Metal API가 Objective-C 기반이라 필요

---

## 4. 플러그인 인터페이스

### 세 가지 필수 함수

모든 그래픽스 백엔드는 반드시 이 세 함수를 export해야 합니다:

```cpp
// GBackendMetal.h
namespace vz
{
    // 1. 초기화 - Metal 디바이스 생성
    extern "C" METAL_EXPORT bool Initialize(
        graphics::ValidationMode validationMode,
        graphics::GPUPreference preference
    );

    // 2. 디바이스 포인터 반환
    extern "C" METAL_EXPORT graphics::GraphicsDevice* GetGraphicsDevice();

    // 3. 정리 - 리소스 해제
    extern "C" METAL_EXPORT void Deinitialize();
}
```

### Export 매크로
```cpp
#ifdef __APPLE__
    // macOS: visibility 속성 사용
    #define METAL_EXPORT __attribute__((visibility("default")))
#else
    // Windows: dllexport 사용
    #define METAL_EXPORT __declspec(dllexport)
#endif
```

### 플러그인 진입점 구현 (GBackendMetal.mm)
```cpp
namespace vz
{
    // 전역 디바이스 인스턴스
    static std::unique_ptr<graphics::GraphicsDevice_Metal> graphicsDevice;

    bool Initialize(graphics::ValidationMode validationMode, graphics::GPUPreference preference)
    {
        if (graphicsDevice != nullptr)
            return false; // 이미 초기화됨

        graphicsDevice = std::make_unique<graphics::GraphicsDevice_Metal>();
        return graphicsDevice->Initialize(validationMode, preference);
    }

    graphics::GraphicsDevice* GetGraphicsDevice()
    {
        return graphicsDevice.get();
    }

    void Deinitialize()
    {
        graphicsDevice.reset();
    }
}
```

---

## 5. GraphicsDevice 인터페이스

### 추상 클래스 구조
`GraphicsDevice`는 모든 백엔드가 구현해야 하는 **추상 클래스**입니다.

```cpp
// GBackendDevice.h (엔진 코어)
class GraphicsDevice
{
public:
    virtual ~GraphicsDevice() = default;

    // === 리소스 생성 ===
    virtual bool CreateSwapChain(...) = 0;
    virtual bool CreateBuffer2(...) = 0;
    virtual bool CreateTexture(...) = 0;
    virtual bool CreateShader(...) = 0;
    virtual bool CreateSampler(...) = 0;
    virtual bool CreatePipelineState(...) = 0;

    // === 커맨드 리스트 ===
    virtual CommandList BeginCommandList(QUEUE_TYPE queue) = 0;
    virtual void SubmitCommandLists() = 0;

    // === 렌더링 명령 ===
    virtual void RenderPassBegin(...) = 0;
    virtual void RenderPassEnd(...) = 0;
    virtual void Draw(...) = 0;
    virtual void DrawIndexed(...) = 0;
    virtual void Dispatch(...) = 0;  // 컴퓨트 셰이더

    // === 리소스 바인딩 ===
    virtual void BindPipelineState(...) = 0;
    virtual void BindVertexBuffers(...) = 0;
    virtual void BindIndexBuffer(...) = 0;

    // ... 약 60개 이상의 가상 함수
};
```

### Metal 백엔드 클래스
```cpp
// GraphicsDevice_Metal.h
class GraphicsDevice_Metal : public GraphicsDevice
{
private:
    // Metal 네이티브 객체들
    id<MTLDevice> device = nil;           // GPU 디바이스
    id<MTLCommandQueue> commandQueue = nil; // 커맨드 큐

public:
    bool Initialize(ValidationMode validationMode, GPUPreference preference);

    // GraphicsDevice의 모든 가상 함수 구현
    bool CreateSwapChain(...) const override;
    bool CreateBuffer2(...) const override;
    // ... (모든 함수 오버라이드)
};
```

### 주요 메서드 카테고리

| 카테고리 | 함수 예시 | 설명 |
|---------|----------|------|
| **리소스 생성** | CreateBuffer, CreateTexture, CreateShader | GPU 리소스 할당 |
| **파이프라인** | CreatePipelineState, BindPipelineState | 렌더링 파이프라인 설정 |
| **커맨드** | BeginCommandList, SubmitCommandLists | GPU 명령 기록/제출 |
| **렌더패스** | RenderPassBegin, RenderPassEnd | 렌더 타겟 설정 |
| **드로우** | Draw, DrawIndexed, DrawInstanced | 실제 렌더링 |
| **컴퓨트** | Dispatch, DispatchIndirect | GPGPU 연산 |
| **복사** | CopyBuffer, CopyTexture | 리소스 간 데이터 복사 |
| **동기화** | WaitForGPU, Barrier | CPU-GPU 동기화 |

---

## 6. 현재 구현 상태

### 구현 완료
| 항목 | 상태 | 설명 |
|------|------|------|
| 플러그인 구조 | ✅ | Initialize/GetGraphicsDevice/Deinitialize |
| Metal 디바이스 초기화 | ✅ | MTLCreateSystemDefaultDevice() |
| 커맨드 큐 생성 | ✅ | [device newCommandQueue] |
| 디바이스 정보 조회 | ✅ | adapterName, driverDescription |
| 메모리 사용량 조회 | ✅ | recommendedMaxWorkingSetSize |
| CMake 빌드 설정 | ✅ | macOS/Windows 크로스플랫폼 |

### 스텁 (미구현)
| 항목 | Metal API | 우선순위 |
|------|-----------|----------|
| SwapChain | CAMetalLayer | 높음 |
| Buffer | MTLBuffer | 높음 |
| Texture | MTLTexture | 높음 |
| Shader | MTLLibrary, MTLFunction | 높음 |
| PipelineState | MTLRenderPipelineState | 높음 |
| CommandList | MTLCommandBuffer | 높음 |
| RenderPass | MTLRenderCommandEncoder | 높음 |
| Draw 명령 | drawPrimitives: | 중간 |
| Compute | MTLComputeCommandEncoder | 낮음 |

### 초기화 코드 상세
```objc
bool GraphicsDevice_Metal::Initialize(ValidationMode validationMode_, GPUPreference preference)
{
    this->validationMode = validationMode_;

    // 1. Metal 디바이스 생성 (시스템 기본 GPU)
    device = MTLCreateSystemDefaultDevice();
    if (!device) {
        NSLog(@"Metal is not supported on this device");
        return false;
    }

    // 2. 커맨드 큐 생성 (GPU 명령 제출용)
    commandQueue = [device newCommandQueue];
    if (!commandQueue) {
        NSLog(@"Failed to create Metal command queue");
        return false;
    }

    // 3. 디바이스 정보 저장
    adapterName = [[device name] UTF8String];  // 예: "Apple M1"
    driverDescription = "Apple Metal";

    NSLog(@"Metal device initialized: %@", [device name]);
    return true;
}
```

---

## 7. 빌드 방법

### macOS (CMake)
```bash
# 1. GraphicsBackends 디렉토리로 이동
cd GraphicsBackends

# 2. 빌드 디렉토리 생성
mkdir build && cd build

# 3. CMake 설정
cmake ..

# 4. 빌드
make

# 결과: libGBackendMetal.dylib
```

### macOS (Xcode)
```bash
cmake .. -G Xcode
open GBackendMetal.xcodeproj
```

### Windows (Visual Studio)
- `GBackendMetal.vcxproj`를 솔루션에 추가
- 빌드 → GBackendMetal.dll 생성 (스텁)

### CMakeLists.txt 핵심 내용
```cmake
# macOS에서만 Metal 프레임워크 링크
if(APPLE)
    find_library(METAL_FRAMEWORK Metal REQUIRED)
    find_library(METALKIT_FRAMEWORK MetalKit REQUIRED)
    find_library(FOUNDATION_FRAMEWORK Foundation REQUIRED)

    target_link_libraries(GBackendMetal PRIVATE
        ${METAL_FRAMEWORK}
        ${METALKIT_FRAMEWORK}
        ${FOUNDATION_FRAMEWORK}
    )

    # ARC(Automatic Reference Counting) 활성화
    target_compile_options(GBackendMetal PRIVATE -fobjc-arc)
endif()
```

---

## 8. 다음 단계

### 구현 로드맵

#### Phase 1: 기본 렌더링 (필수)
1. **SwapChain**: CAMetalLayer 설정, drawable 획득
2. **Buffer**: MTLBuffer 생성, 버텍스/인덱스 버퍼
3. **Texture**: MTLTexture 생성, 렌더 타겟
4. **Shader**: MSL(Metal Shading Language) 또는 SPIRV-Cross
5. **Pipeline**: MTLRenderPipelineState 생성
6. **CommandList**: MTLCommandBuffer + MTLRenderCommandEncoder
7. **Draw**: 삼각형 하나 그리기

#### Phase 2: 고급 기능
- Compute Shader (MTLComputeCommandEncoder)
- 멀티 스레드 렌더링
- 타임스탬프 쿼리

#### Phase 3: 최적화
- 메모리 풀링
- 파이프라인 캐시
- 리소스 힙

### 참고할 DX12 구현
Metal 구현 시 `GraphicsDevice_DX12.cpp`를 참고하면 좋습니다:
- 구조가 유사함 (저수준 API)
- 리소스 관리 패턴 참고
- 커맨드 버퍼 관리 방식 참고

### Metal vs DX12 대응표
| DX12 | Metal |
|------|-------|
| ID3D12Device | id\<MTLDevice\> |
| ID3D12CommandQueue | id\<MTLCommandQueue\> |
| ID3D12CommandList | id\<MTLCommandBuffer\> |
| ID3D12Resource (Buffer) | id\<MTLBuffer\> |
| ID3D12Resource (Texture) | id\<MTLTexture\> |
| ID3D12PipelineState | id\<MTLRenderPipelineState\> |
| D3D12_RENDER_TARGET_VIEW | MTLRenderPassDescriptor |
| Root Signature | Argument Buffer (Metal 3) |

---

## 부록: 용어 정리

| 용어 | 설명 |
|------|------|
| **Backend** | 특정 그래픽스 API(DX12, Metal 등)의 구현체 |
| **SwapChain** | 화면에 표시할 프레임 버퍼들의 집합 |
| **CommandList** | GPU에 보낼 명령들을 기록하는 버퍼 |
| **Pipeline State** | 셰이더, 블렌딩, 래스터화 등 렌더링 설정 묶음 |
| **RenderPass** | 렌더 타겟을 설정하고 그리기 명령을 실행하는 단위 |
| **Barrier** | GPU 리소스 접근 동기화를 위한 메모리 배리어 |
| **MTL** | Metal의 접두사 (Metal) |
| **ARC** | Automatic Reference Counting, Objective-C 메모리 관리 |
