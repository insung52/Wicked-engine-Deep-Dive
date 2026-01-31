# VizMotive Engine - Metal Graphics Backend 구현 가이드

## 목차
1. [전체 실행 흐름 (Big Picture)](#1-전체-실행-흐름-big-picture) ⭐ **먼저 읽으세요!**
2. [그래픽스 백엔드란?](#2-그래픽스-백엔드란)
3. [VizMotive 엔진의 백엔드 아키텍처](#3-vizmotive-엔진의-백엔드-아키텍처)
4. [Metal 백엔드 파일 구조](#4-metal-백엔드-파일-구조)
5. [플러그인 인터페이스](#5-플러그인-인터페이스)
6. [GraphicsDevice 인터페이스](#6-graphicsdevice-인터페이스)
7. [포맷 변환 시스템](#7-포맷-변환-시스템)
8. [리소스 생성 구현](#8-리소스-생성-구현)
9. [렌더링 파이프라인](#9-렌더링-파이프라인)
10. [커맨드 리스트와 프레임 동기화](#10-커맨드-리스트와-프레임-동기화)
11. [Metal Triangle 샘플](#11-metal-triangle-샘플)
12. [현재 구현 상태](#12-현재-구현-상태)
13. [빌드 방법](#13-빌드-방법)
14. [다음 단계](#14-다음-단계)

---

## 1. 전체 실행 흐름 (Big Picture)

### 1.1 프로그램 생명주기 개요

C 언어의 `main()` 함수처럼, 그래픽스 애플리케이션도 명확한 실행 순서가 있습니다:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        프로그램 생명주기                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐                                                    │
│  │   1. 시작       │  main() 진입                                        │
│  │   (Startup)     │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │   2. 초기화     │  GraphicsDevice 생성, SwapChain 생성                │
│  │   (Initialize)  │  한 번만 실행                                       │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │   3. 리소스     │  Buffer, Texture, Shader, Pipeline 생성            │
│  │   (Resources)   │  한 번만 실행 (또는 필요시 동적 생성)                 │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │   4. 렌더 루프  │◀─────┐  매 프레임 반복 (60fps = 초당 60번)          │
│  │   (Render Loop) │      │                                             │
│  └────────┬────────┘      │                                             │
│           │               │                                             │
│           └───────────────┘  (윈도우 닫기 전까지 계속)                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │   5. 정리       │  리소스 해제, GraphicsDevice 소멸                    │
│  │   (Cleanup)     │  한 번만 실행                                       │
│  └─────────────────┘                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 시간순 전체 흐름 (상세)

아래는 프로그램 시작부터 종료까지 호출되는 함수들을 **시간순**으로 나열한 것입니다:

```
═══════════════════════════════════════════════════════════════════════════
                              시간 →
═══════════════════════════════════════════════════════════════════════════

[1. 프로그램 시작] ─────────────────────────────────────────────────────────

    main()
      │
      ├─► 윈도우 생성 (NSWindow / HWND)
      │
      └─► GraphicsDevice 초기화
            │
            ├─► device->Initialize()
            │     • MTLCreateSystemDefaultDevice()     // GPU 획득
            │     • [device newCommandQueue]           // 커맨드 큐 생성
            │     • dispatch_semaphore_create()        // 동기화 준비
            │
            └─► device->CreateSwapChain()
                  • CAMetalLayer 생성                  // 화면 출력용 레이어
                  • 윈도우에 연결

[2. 리소스 생성 (한 번만)] ─────────────────────────────────────────────────

    // 렌더링에 필요한 모든 리소스를 미리 생성

    device->CreateBuffer()       ──► 버텍스/인덱스 데이터 업로드
    device->CreateTexture()      ──► 이미지 데이터 업로드
    device->CreateShader()       ──► 셰이더 컴파일
    device->CreateSampler()      ──► 텍스처 샘플링 설정
    device->CreatePipelineState() ──► 렌더링 파이프라인 조립

[3. 렌더 루프 (매 프레임 반복)] ────────────────────────────────────────────

    while (running)  // 보통 60fps로 실행
    {
        ┌─────────────────────────────────────────────────────────────┐
        │ Frame N                                                      │
        │                                                              │
        │  // 3-1. 커맨드 기록 시작                                     │
        │  cmd = device->BeginCommandList()                            │
        │                                                              │
        │  // 3-2. 렌더패스 시작 (렌더 타겟 설정)                         │
        │  device->RenderPassBegin(&swapChain, cmd)                    │
        │    └─► 화면 클리어, drawable 획득                              │
        │                                                              │
        │  // 3-3. 상태 바인딩 (무엇으로 그릴지 설정)                      │
        │  device->BindViewports(...)         // 뷰포트 설정            │
        │  device->BindScissorRects(...)      // 시저 설정              │
        │  device->BindPipelineState(...)     // 셰이더+상태 바인딩      │
        │  device->BindVertexBuffers(...)     // 버텍스 데이터 바인딩    │
        │  device->BindIndexBuffer(...)       // 인덱스 데이터 바인딩    │
        │  device->BindConstantBuffer(...)    // 상수 버퍼 바인딩        │
        │  device->BindResource(texture, ...) // 텍스처 바인딩          │
        │  device->BindSampler(...)           // 샘플러 바인딩          │
        │                                                              │
        │  // 3-4. Draw 명령 (GPU야, 그려라!)                           │
        │  device->Draw(vertexCount, ...)     // 또는                  │
        │  device->DrawIndexed(indexCount, ...)                        │
        │                                                              │
        │  // 3-5. 렌더패스 종료                                        │
        │  device->RenderPassEnd(cmd)                                  │
        │                                                              │
        │  // 3-6. GPU에 제출 + 화면 표시                                │
        │  device->SubmitCommandLists()                                │
        │    └─► [commandBuffer commit]                                │
        │    └─► [commandBuffer presentDrawable]                       │
        │                                                              │
        └─────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
        ┌─────────────────────────────────────────────────────────────┐
        │ Frame N+1  (반복...)                                         │
        └─────────────────────────────────────────────────────────────┘
    }

[4. 프로그램 종료] ─────────────────────────────────────────────────────────

    device->WaitForGPU()    // GPU 작업 완료 대기

    // 리소스 자동 해제 (ARC / 소멸자)
    delete device           // GraphicsDevice 소멸

    // 윈도우 닫기

═══════════════════════════════════════════════════════════════════════════
```

### 1.3 렌더 루프 한 프레임 상세

렌더 루프의 **한 프레임**을 더 자세히 살펴보면:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         한 프레임의 렌더링 과정                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   CPU 측 (앱 코드)                      GPU 측 (하드웨어)                 │
│   ──────────────────                   ──────────────────               │
│                                                                         │
│   ① BeginCommandList()                                                  │
│      └─► 빈 커맨드 버퍼 생성            (대기 중...)                      │
│                                                                         │
│   ② RenderPassBegin()                                                   │
│      └─► 렌더 타겟 설정                                                  │
│      └─► 렌더 인코더 생성                                                │
│                                                                         │
│   ③ BindPipelineState()                                                 │
│      └─► 어떤 셰이더 사용할지          명령                              │
│                                        기록                              │
│   ④ BindVertexBuffers()                 중                              │
│      └─► 어떤 데이터 그릴지            ─┬─                              │
│                                         │                               │
│   ⑤ Draw(3, 0)                          │                               │
│      └─► "삼각형 그려라" 명령           │                               │
│                                         │                               │
│   ⑥ RenderPassEnd()                     │                               │
│      └─► 인코더 종료                   ─┘                               │
│                                                                         │
│   ⑦ SubmitCommandLists()               ─────────────────────►          │
│      └─► GPU에 커맨드 전송                    GPU가 명령 실행            │
│      └─► Present 요청                        ┌──────────────┐           │
│                                              │ 버텍스 처리   │           │
│   (CPU는 다음 프레임 준비 가능)               │ 래스터화     │           │
│                                              │ 프래그먼트    │           │
│                                              │ 화면 출력     │           │
│                                              └──────────────┘           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.4 코드로 보는 전체 흐름

실제 코드에서 위 흐름이 어떻게 구현되는지 보겠습니다:

```cpp
// ═══════════════════════════════════════════════════════════════════════
// main.mm - 프로그램 진입점
// ═══════════════════════════════════════════════════════════════════════

int main()
{
    // ═══════════════════════════════════════════════════════════════════
    // [1단계] 초기화 - 프로그램 시작 시 한 번만 실행
    // ═══════════════════════════════════════════════════════════════════

    // 1-1. 윈도우 생성
    Window* window = CreateWindow(800, 600, "My App");

    // 1-2. Graphics Device 생성 및 초기화
    GraphicsDevice_Metal* device = new GraphicsDevice_Metal();
    device->Initialize(ValidationMode::Disabled, GPUPreference::Discrete);
    //                 ↑ 디버그 검증 비활성화        ↑ 외장 GPU 선호

    // 1-3. SwapChain 생성 (화면 출력용)
    SwapChainDesc swapDesc = {};
    swapDesc.width = 800;
    swapDesc.height = 600;
    swapDesc.format = Format::B8G8R8A8_UNORM;  // 픽셀 포맷
    swapDesc.buffer_count = 2;                  // 더블 버퍼링
    swapDesc.vsync = true;                      // 수직 동기화
    swapDesc.clear_color[0] = 0.2f;             // 클리어 색상 (R)
    swapDesc.clear_color[1] = 0.2f;             // (G)
    swapDesc.clear_color[2] = 0.3f;             // (B)
    swapDesc.clear_color[3] = 1.0f;             // (A)

    SwapChain swapChain;
    device->CreateSwapChain(&swapDesc, window, &swapChain);

    // ═══════════════════════════════════════════════════════════════════
    // [2단계] 리소스 생성 - 렌더링에 필요한 데이터 준비 (한 번만)
    // ═══════════════════════════════════════════════════════════════════

    // 2-1. 버텍스 데이터 정의
    Vertex vertices[] = {
        { {  0.0f,  0.5f }, { 1.0f, 0.0f, 0.0f } },  // 상단 (빨강)
        { { -0.5f, -0.5f }, { 0.0f, 1.0f, 0.0f } },  // 좌하단 (녹색)
        { {  0.5f, -0.5f }, { 0.0f, 0.0f, 1.0f } },  // 우하단 (파랑)
    };

    // 2-2. 버텍스 버퍼 생성
    GPUBufferDesc bufferDesc = {};
    bufferDesc.size = sizeof(vertices);
    bufferDesc.usage = Usage::UPLOAD;           // CPU에서 쓰기 가능
    bufferDesc.bind_flags = BindFlag::VERTEX_BUFFER;

    GPUBuffer vertexBuffer;
    device->CreateBuffer(&bufferDesc, vertices, &vertexBuffer);

    // 2-3. 셰이더 생성 (컴파일)
    Shader vertexShader, pixelShader;
    device->CreateShader(ShaderStage::VS, shaderSource, strlen(shaderSource), &vertexShader);
    device->CreateShader(ShaderStage::PS, shaderSource, strlen(shaderSource), &pixelShader);

    // 2-4. 입력 레이아웃 정의 (버텍스 구조 설명)
    InputLayout inputLayout;
    inputLayout.elements = {
        { "POSITION", 0, Format::R32G32_FLOAT,   0, offsetof(Vertex, position) },
        { "COLOR",    0, Format::R32G32B32_FLOAT, 0, offsetof(Vertex, color) },
    };

    // 2-5. 파이프라인 상태 생성 (모든 렌더링 설정 조합)
    RenderPassInfo passInfo = {};
    passInfo.rt_count = 1;
    passInfo.rt_formats[0] = Format::B8G8R8A8_UNORM;

    PipelineStateDesc psoDesc = {};
    psoDesc.vs = &vertexShader;          // 버텍스 셰이더
    psoDesc.ps = &pixelShader;           // 픽셀 셰이더
    psoDesc.il = &inputLayout;           // 입력 레이아웃
    psoDesc.pt = PrimitiveTopology::TRIANGLELIST;  // 삼각형 리스트

    PipelineState pipelineState;
    device->CreatePipelineState(&psoDesc, &pipelineState, &passInfo);

    // ═══════════════════════════════════════════════════════════════════
    // [3단계] 렌더 루프 - 매 프레임 반복 (60fps)
    // ═══════════════════════════════════════════════════════════════════

    while (window->IsOpen())  // 윈도우가 열려있는 동안 반복
    {
        // ─────────────────────────────────────────────────────────────
        // 3-1. 커맨드 리스트 시작
        // ─────────────────────────────────────────────────────────────
        CommandList cmd = device->BeginCommandList(QUEUE_GRAPHICS);

        // ─────────────────────────────────────────────────────────────
        // 3-2. 렌더패스 시작 (렌더 타겟 = SwapChain 백버퍼)
        // ─────────────────────────────────────────────────────────────
        device->RenderPassBegin(&swapChain, cmd);
        // → 내부적으로: 화면 클리어 + drawable 획득

        // ─────────────────────────────────────────────────────────────
        // 3-3. 뷰포트 & 시저 설정
        // ─────────────────────────────────────────────────────────────
        Viewport viewport = { 0, 0, 800, 600, 0.0f, 1.0f };
        device->BindViewports(1, &viewport, cmd);

        Rect scissor = { 0, 0, 800, 600 };
        device->BindScissorRects(1, &scissor, cmd);

        // ─────────────────────────────────────────────────────────────
        // 3-4. 파이프라인 바인딩 (어떤 셰이더/설정으로 그릴지)
        // ─────────────────────────────────────────────────────────────
        device->BindPipelineState(&pipelineState, cmd);

        // ─────────────────────────────────────────────────────────────
        // 3-5. 버텍스 버퍼 바인딩 (어떤 데이터를 그릴지)
        // ─────────────────────────────────────────────────────────────
        const GPUBuffer* vbs[] = { &vertexBuffer };
        uint32_t strides[] = { sizeof(Vertex) };
        uint64_t offsets[] = { 0 };
        device->BindVertexBuffers(vbs, 0, 1, strides, offsets, cmd);

        // ─────────────────────────────────────────────────────────────
        // 3-6. Draw 명령 (GPU야, 그려라!)
        // ─────────────────────────────────────────────────────────────
        device->Draw(3, 0, cmd);  // 3개의 버텍스, 0번부터 시작
        //           ↑     ↑
        //      버텍스 수  시작 위치

        // ─────────────────────────────────────────────────────────────
        // 3-7. 렌더패스 종료
        // ─────────────────────────────────────────────────────────────
        device->RenderPassEnd(cmd);

        // ─────────────────────────────────────────────────────────────
        // 3-8. GPU에 제출 + 화면에 표시
        // ─────────────────────────────────────────────────────────────
        device->SubmitCommandLists();
        // → 내부적으로: commit + presentDrawable + 세마포어 동기화
    }

    // ═══════════════════════════════════════════════════════════════════
    // [4단계] 정리 - 프로그램 종료 시 한 번만 실행
    // ═══════════════════════════════════════════════════════════════════

    device->WaitForGPU();  // GPU 작업 완료 대기
    delete device;         // 디바이스 소멸 (리소스 자동 해제)

    return 0;
}
```

### 1.5 함수 호출 횟수 요약

| 함수 | 호출 시점 | 호출 횟수 |
|------|----------|----------|
| `Initialize()` | 프로그램 시작 | **1번** |
| `CreateSwapChain()` | 프로그램 시작 | **1번** |
| `CreateBuffer()` | 리소스 생성 | **1번** (동적 생성 제외) |
| `CreateTexture()` | 리소스 생성 | **1번** (동적 생성 제외) |
| `CreateShader()` | 리소스 생성 | **1번** |
| `CreatePipelineState()` | 리소스 생성 | **1번** |
| `BeginCommandList()` | 렌더 루프 | **매 프레임** |
| `RenderPassBegin()` | 렌더 루프 | **매 프레임** (여러 패스 가능) |
| `BindPipelineState()` | 렌더 루프 | **매 프레임** (여러 번 가능) |
| `BindVertexBuffers()` | 렌더 루프 | **매 프레임** (여러 번 가능) |
| `Draw()` / `DrawIndexed()` | 렌더 루프 | **매 프레임** (여러 번) |
| `RenderPassEnd()` | 렌더 루프 | **매 프레임** |
| `SubmitCommandLists()` | 렌더 루프 | **매 프레임** |
| `WaitForGPU()` | 프로그램 종료 | **1번** |

### 1.6 왜 이런 순서인가?

```
Q: 왜 리소스를 먼저 만들어야 하나요?
A: GPU는 데이터(버텍스, 텍스처)가 GPU 메모리에 있어야 사용할 수 있습니다.
   매 프레임마다 업로드하면 너무 느리므로, 미리 한 번 만들어 둡니다.

Q: 왜 파이프라인을 미리 만들어야 하나요?
A: 파이프라인 생성은 비용이 큽니다 (셰이더 컴파일 포함).
   렌더 루프에서는 이미 만들어진 파이프라인을 "바인딩"만 합니다.

Q: 왜 BeginCommandList → ... → Submit 순서인가요?
A: GPU는 명령들을 배치(batch)로 받아서 실행합니다.
   Begin에서 시작, 중간에 명령 기록, Submit에서 GPU로 전송합니다.

Q: 왜 RenderPassBegin/End로 감싸야 하나요?
A: 렌더 타겟(어디에 그릴지)을 명확히 해야 GPU가 알 수 있습니다.
   한 패스 내에서만 해당 렌더 타겟에 그릴 수 있습니다.
```

### 1.7 여러 오브젝트 그리기

실제 게임에서는 한 프레임에 여러 오브젝트를 그립니다:

```cpp
while (running)
{
    CommandList cmd = device->BeginCommandList(QUEUE_GRAPHICS);
    device->RenderPassBegin(&swapChain, cmd);

    // 뷰포트, 시저 설정 (한 번)
    device->BindViewports(1, &viewport, cmd);
    device->BindScissorRects(1, &scissor, cmd);

    // ─────────── 오브젝트 1: 캐릭터 ───────────
    device->BindPipelineState(&characterPipeline, cmd);
    device->BindVertexBuffers(&characterMesh, ...);
    device->BindResource(&characterTexture, 0, cmd);
    device->DrawIndexed(characterIndexCount, 0, 0, cmd);

    // ─────────── 오브젝트 2: 배경 ───────────
    device->BindPipelineState(&backgroundPipeline, cmd);
    device->BindVertexBuffers(&backgroundMesh, ...);
    device->BindResource(&backgroundTexture, 0, cmd);
    device->DrawIndexed(backgroundIndexCount, 0, 0, cmd);

    // ─────────── 오브젝트 3: UI ───────────
    device->BindPipelineState(&uiPipeline, cmd);
    device->BindVertexBuffers(&uiQuad, ...);
    device->BindResource(&uiTexture, 0, cmd);
    device->Draw(6, 0, cmd);  // UI 쿼드

    device->RenderPassEnd(cmd);
    device->SubmitCommandLists();
}
```

---

## 2. 그래픽스 백엔드란?

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

## 3. VizMotive 엔진의 백엔드 아키텍처

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

## 4. Metal 백엔드 파일 구조

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
└── GraphicsDevice_Metal.mm    # Metal 디바이스 구현 (~1900 lines)

Examples/
└── MetalTriangleSample/
    └── main.mm                # RGB 삼각형 샘플 (~300 lines)
```

### 파일별 역할

| 파일 | 역할 |
|------|------|
| `GBackendMetal.h` | DLL export 함수 선언 (Initialize, GetGraphicsDevice, Deinitialize) |
| `GBackendMetal.mm` | 플러그인 진입점, GraphicsDevice_Metal 인스턴스 관리 |
| `GraphicsDevice_Metal.h` | GraphicsDevice 추상 클래스를 상속받은 Metal 구현 클래스 |
| `GraphicsDevice_Metal.mm` | 실제 Metal API 호출 코드 (포맷 변환, 리소스 생성, 렌더링 등) |
| `CMakeLists.txt` | macOS에서 빌드하기 위한 CMake 설정 |
| `main.mm` (샘플) | Metal 백엔드를 사용한 삼각형 렌더링 데모 |

### .mm 확장자란?
- **Objective-C++** 파일
- C++과 Objective-C를 혼합 사용 가능
- Metal API가 Objective-C 기반이라 필요

---

## 5. 플러그인 인터페이스

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

## 6. GraphicsDevice 인터페이스

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

### Metal 백엔드 클래스 (GraphicsDevice_Metal.h)
```cpp
class GraphicsDevice_Metal : public GraphicsDevice
{
private:
    // Metal 네이티브 객체들
    id<MTLDevice> device = nil;               // GPU 디바이스
    id<MTLCommandQueue> commandQueue = nil;   // 커맨드 큐

    // 프레임 동기화
    dispatch_semaphore_t frameSemaphore = nullptr;  // 트리플 버퍼링용
    uint64_t completedFrameCount = 0;

    // 커맨드 리스트 풀
    std::vector<CommandList_Metal> commandLists;
    std::vector<CommandList> activeCommandLists;

    // 지연 해제 핸들러
    AllocationHandler_Metal allocationHandler;

public:
    bool Initialize(ValidationMode validationMode, GPUPreference preference);

    // 모든 가상 함수 구현...
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

## 7. 포맷 변환 시스템

### 포맷 변환이란?
엔진은 플랫폼 독립적인 `Format` 열거형을 사용하지만, Metal은 `MTLPixelFormat`을 사용합니다.
백엔드는 이 둘 사이를 변환해야 합니다.

```
엔진 Format         →  Metal Format
─────────────────────────────────────
Format::R8G8B8A8_UNORM  →  MTLPixelFormatRGBA8Unorm
Format::D32_FLOAT       →  MTLPixelFormatDepth32Float
Format::BC7_UNORM       →  MTLPixelFormatBC7_RGBAUnorm
```

### 텍스처 포맷 변환 (_ConvertFormat)
```objc
inline MTLPixelFormat _ConvertFormat(Format format)
{
    switch (format)
    {
    // 일반 컬러 포맷
    case Format::R8G8B8A8_UNORM:      return MTLPixelFormatRGBA8Unorm;
    case Format::R8G8B8A8_UNORM_SRGB: return MTLPixelFormatRGBA8Unorm_sRGB;
    case Format::B8G8R8A8_UNORM:      return MTLPixelFormatBGRA8Unorm;
    case Format::R32G32B32A32_FLOAT:  return MTLPixelFormatRGBA32Float;
    case Format::R16G16B16A16_FLOAT:  return MTLPixelFormatRGBA16Float;

    // 깊이 포맷
    case Format::D32_FLOAT:           return MTLPixelFormatDepth32Float;
    case Format::D24_UNORM_S8_UINT:   return MTLPixelFormatDepth24Unorm_Stencil8;
    case Format::D32_FLOAT_S8X24_UINT: return MTLPixelFormatDepth32Float_Stencil8;

    // 압축 포맷 (BC = Block Compression)
    case Format::BC1_UNORM:           return MTLPixelFormatBC1_RGBA;
    case Format::BC3_UNORM:           return MTLPixelFormatBC3_RGBA;
    case Format::BC7_UNORM:           return MTLPixelFormatBC7_RGBAUnorm;

    default:                          return MTLPixelFormatInvalid;
    }
}
```

### 버텍스 포맷 변환 (_ConvertFormat_VertexInput)
버텍스 셰이더 입력을 위한 별도의 변환 함수가 필요합니다:

```objc
inline MTLVertexFormat _ConvertFormat_VertexInput(Format format)
{
    switch (format)
    {
    // 위치 데이터 (float)
    case Format::R32G32B32A32_FLOAT: return MTLVertexFormatFloat4;
    case Format::R32G32B32_FLOAT:    return MTLVertexFormatFloat3;
    case Format::R32G32_FLOAT:       return MTLVertexFormatFloat2;
    case Format::R32_FLOAT:          return MTLVertexFormatFloat;

    // 컬러 데이터 (normalized)
    case Format::R8G8B8A8_UNORM:     return MTLVertexFormatUChar4Normalized;
    case Format::R16G16B16A16_UNORM: return MTLVertexFormatUShort4Normalized;

    // 정수 데이터
    case Format::R32G32B32A32_UINT:  return MTLVertexFormatUInt4;
    case Format::R32G32B32A32_SINT:  return MTLVertexFormatInt4;

    default:                         return MTLVertexFormatInvalid;
    }
}
```

### 기타 변환 함수들
Metal 백엔드에는 다양한 변환 함수가 구현되어 있습니다:

| 함수 | 용도 | 예시 |
|------|------|------|
| `_ConvertPrimitiveTopology` | 프리미티브 타입 | TRIANGLELIST → MTLPrimitiveTypeTriangle |
| `_ConvertComparisonFunc` | 깊이/스텐실 비교 | LESS → MTLCompareFunctionLess |
| `_ConvertCullMode` | 컬링 모드 | BACK → MTLCullModeBack |
| `_ConvertBlend` | 블렌드 팩터 | SRC_ALPHA → MTLBlendFactorSourceAlpha |
| `_ConvertBlendOp` | 블렌드 연산 | ADD → MTLBlendOperationAdd |
| `_ConvertTextureAddressMode` | UV 래핑 | WRAP → MTLSamplerAddressModeRepeat |
| `_ConvertFilter_MinMag` | 필터링 | LINEAR → MTLSamplerMinMagFilterLinear |
| `_ConvertStencilOp` | 스텐실 연산 | REPLACE → MTLStencilOperationReplace |
| `_ConvertColorWriteMask` | 컬러 마스크 | ENABLE_RGB → RGB bits |

---

## 8. 리소스 생성 구현

### 8.1 SwapChain 생성

**SwapChain**이란?
- 화면에 표시할 프레임 버퍼들의 집합
- 더블/트리플 버퍼링을 통해 끊김 없는 렌더링 제공
- Metal에서는 `CAMetalLayer`로 구현

```objc
bool GraphicsDevice_Metal::CreateSwapChain(const SwapChainDesc* desc,
                                           window_type window,
                                           SwapChain* swapchain) const
{
    auto internal_state = std::make_shared<SwapChain_Metal>();
    swapchain->internal_state = internal_state;
    swapchain->desc = *desc;

    // 1. NSView에서 CAMetalLayer 생성
    NSView* nsView = (__bridge NSView*)window;
    CAMetalLayer* metalLayer = [CAMetalLayer layer];

    // 2. Metal 디바이스 및 포맷 설정
    metalLayer.device = device;
    metalLayer.pixelFormat = _ConvertFormat(desc->format);  // 예: BGRA8
    metalLayer.framebufferOnly = YES;  // 최적화: 렌더타겟 전용
    metalLayer.drawableSize = CGSizeMake(desc->width, desc->height);

    // 3. VSync 설정
    if (@available(macOS 10.13, *))
    {
        metalLayer.displaySyncEnabled = desc->vsync;
    }

    // 4. 뷰에 레이어 연결
    nsView.wantsLayer = YES;
    nsView.layer = metalLayer;

    internal_state->metalLayer = metalLayer;
    return true;
}
```

### 8.2 Buffer 생성

**GPUBuffer**란?
- GPU 메모리에 저장되는 데이터 블록
- 용도: 버텍스 데이터, 인덱스 데이터, 상수 버퍼, 등

```objc
bool GraphicsDevice_Metal::CreateBuffer2(const GPUBufferDesc* desc,
                                         const std::function<void(void*)>& init_callback,
                                         GPUBuffer* buffer, ...) const
{
    auto internal_state = std::make_shared<Resource_Metal>();

    // 1. 메모리 모드 선택
    MTLResourceOptions options;
    switch (desc->usage)
    {
    case Usage::DEFAULT:   // GPU 전용
        options = MTLResourceStorageModePrivate;
        break;
    case Usage::UPLOAD:    // CPU → GPU 업로드용
        options = MTLResourceStorageModeShared | MTLResourceCPUCacheModeWriteCombined;
        break;
    case Usage::READBACK:  // GPU → CPU 읽기용
        options = MTLResourceStorageModeShared;
        break;
    }

    // 2. CPU 접근 가능한 버퍼 (UPLOAD, READBACK)
    if (desc->usage == Usage::UPLOAD || desc->usage == Usage::READBACK)
    {
        internal_state->buffer = [device newBufferWithLength:desc->size options:options];
        internal_state->mapped_data = [internal_state->buffer contents];

        // 초기 데이터 복사
        if (init_callback && internal_state->mapped_data)
        {
            init_callback(internal_state->mapped_data);
        }
    }
    // 3. GPU 전용 버퍼 (DEFAULT) - 스테이징 버퍼 사용
    else if (init_callback)
    {
        // 임시 CPU 버퍼 생성
        id<MTLBuffer> stagingBuffer = [device newBufferWithLength:desc->size
            options:MTLResourceStorageModeShared];
        init_callback([stagingBuffer contents]);

        // GPU 버퍼 생성
        internal_state->buffer = [device newBufferWithLength:desc->size
            options:MTLResourceStorageModePrivate];

        // Blit으로 복사 (나중에 커맨드 버퍼에서 실행)
        // stagingBuffer → internal_state->buffer
    }

    return true;
}
```

### 8.3 Texture 생성

**Texture**란?
- 2D/3D 이미지 데이터
- 용도: 표면 텍스처, 렌더 타겟, 깊이 버퍼

```objc
bool GraphicsDevice_Metal::CreateTexture(const TextureDesc* desc,
                                         const SubresourceData* initial_data,
                                         Texture* texture, ...) const
{
    auto internal_state = std::make_shared<Texture_Metal>();

    // 1. 텍스처 디스크립터 설정
    MTLTextureDescriptor* texDesc = [[MTLTextureDescriptor alloc] init];

    // 타입 설정
    switch (desc->type)
    {
    case TextureDesc::Type::TEXTURE_1D:
        texDesc.textureType = MTLTextureType1D;
        break;
    case TextureDesc::Type::TEXTURE_2D:
        texDesc.textureType = (desc->sample_count > 1) ?
            MTLTextureType2DMultisample : MTLTextureType2D;
        break;
    case TextureDesc::Type::TEXTURE_3D:
        texDesc.textureType = MTLTextureType3D;
        break;
    case TextureDesc::Type::TEXTURE_CUBE:
        texDesc.textureType = MTLTextureTypeCube;
        break;
    }

    texDesc.pixelFormat = _ConvertFormat(desc->format);
    texDesc.width = desc->width;
    texDesc.height = desc->height;
    texDesc.depth = desc->depth;
    texDesc.mipmapLevelCount = desc->mip_levels;
    texDesc.arrayLength = desc->array_size;
    texDesc.sampleCount = desc->sample_count;

    // 2. 사용 플래그 설정
    MTLTextureUsage usage = MTLTextureUsageShaderRead;
    if (has_flag(desc->bind_flags, BindFlag::RENDER_TARGET))
        usage |= MTLTextureUsageRenderTarget;
    if (has_flag(desc->bind_flags, BindFlag::UNORDERED_ACCESS))
        usage |= MTLTextureUsageShaderWrite;
    texDesc.usage = usage;

    // 3. 저장 모드 결정
    texDesc.storageMode = MTLStorageModePrivate;  // GPU 전용이 기본
    if (desc->usage == Usage::UPLOAD)
        texDesc.storageMode = MTLStorageModeShared;

    // 4. 텍스처 생성
    internal_state->texture = [device newTextureWithDescriptor:texDesc];

    // 5. 초기 데이터 업로드 (있는 경우)
    if (initial_data)
    {
        // 각 밉맵 레벨과 배열 슬라이스에 대해 데이터 복사
        for (uint32_t slice = 0; slice < desc->array_size; ++slice)
        {
            for (uint32_t mip = 0; mip < desc->mip_levels; ++mip)
            {
                // ... replaceRegion으로 데이터 복사
            }
        }
    }

    return true;
}
```

### 8.4 Shader 생성

**Shader**란?
- GPU에서 실행되는 프로그램
- 버텍스 셰이더: 정점 변환
- 픽셀(프래그먼트) 셰이더: 픽셀 색상 계산

```objc
bool GraphicsDevice_Metal::CreateShader(ShaderStage stage,
                                        const void* shadercode,
                                        size_t shadercode_size,
                                        Shader* shader) const
{
    auto internal_state = std::make_shared<Shader_Metal>();
    internal_state->stage = stage;

    // 1. 셰이더 소스를 NSString으로 변환
    NSString* source = [[NSString alloc] initWithBytes:shadercode
                                                length:shadercode_size
                                              encoding:NSUTF8StringEncoding];

    // 2. 컴파일 옵션 설정
    MTLCompileOptions* options = [[MTLCompileOptions alloc] init];
    options.languageVersion = MTLLanguageVersion2_4;  // Metal Shading Language 2.4

    // 3. 라이브러리 컴파일 (Metal은 런타임 컴파일 지원)
    NSError* error = nil;
    internal_state->library = [device newLibraryWithSource:source
                                                   options:options
                                                     error:&error];
    if (error)
    {
        NSLog(@"Shader compilation error: %@", error.localizedDescription);
        return false;
    }

    // 4. 진입점 함수 찾기
    NSString* entryPoint;
    switch (stage)
    {
    case ShaderStage::VS: entryPoint = @"vertexMain"; break;
    case ShaderStage::PS: entryPoint = @"fragmentMain"; break;
    case ShaderStage::CS: entryPoint = @"computeMain"; break;
    default: return false;
    }

    internal_state->function = [internal_state->library newFunctionWithName:entryPoint];
    internal_state->entryPoint = [entryPoint UTF8String];

    return internal_state->function != nil;
}
```

### 8.5 Sampler 생성

**Sampler**란?
- 텍스처 샘플링 방법 정의
- 필터링, UV 래핑 방식 등 설정

```objc
bool GraphicsDevice_Metal::CreateSampler(const SamplerDesc* desc, Sampler* sampler) const
{
    auto internal_state = std::make_shared<Sampler_Metal>();

    MTLSamplerDescriptor* samplerDesc = [[MTLSamplerDescriptor alloc] init];

    // 필터링 모드
    samplerDesc.minFilter = _ConvertFilter_MinMag(desc->filter);
    samplerDesc.magFilter = _ConvertFilter_MinMag(desc->filter);
    samplerDesc.mipFilter = _ConvertFilter_Mip(desc->filter);

    // UV 래핑 모드
    samplerDesc.sAddressMode = _ConvertTextureAddressMode(desc->address_u);
    samplerDesc.tAddressMode = _ConvertTextureAddressMode(desc->address_v);
    samplerDesc.rAddressMode = _ConvertTextureAddressMode(desc->address_w);

    // 이방성 필터링 (Anisotropic)
    samplerDesc.maxAnisotropy = desc->max_anisotropy;

    // 깊이 비교 (섀도우 매핑용)
    if (desc->comparison_func != ComparisonFunc::NEVER)
    {
        samplerDesc.compareFunction = _ConvertComparisonFunc(desc->comparison_func);
    }

    // LOD 클램핑
    samplerDesc.lodMinClamp = desc->min_lod;
    samplerDesc.lodMaxClamp = desc->max_lod;

    internal_state->sampler = [device newSamplerStateWithDescriptor:samplerDesc];
    return internal_state->sampler != nil;
}
```

### 8.6 PipelineState 생성

**PipelineState**란?
- 렌더링에 필요한 모든 상태를 묶은 객체
- 셰이더, 블렌딩, 깊이테스트, 래스터화 설정 등 포함

```objc
bool GraphicsDevice_Metal::CreatePipelineState(const PipelineStateDesc* desc,
                                               PipelineState* pso,
                                               const RenderPassInfo* renderpass_info) const
{
    auto internal_state = std::make_shared<PipelineState_Metal>();

    // 1. 렌더 파이프라인 디스크립터
    MTLRenderPipelineDescriptor* pipelineDesc = [[MTLRenderPipelineDescriptor alloc] init];

    // 2. 셰이더 연결
    if (desc->vs && desc->vs->internal_state)
    {
        auto vs_internal = to_internal<Shader_Metal>(desc->vs);
        pipelineDesc.vertexFunction = vs_internal->function;
    }
    if (desc->ps && desc->ps->internal_state)
    {
        auto ps_internal = to_internal<Shader_Metal>(desc->ps);
        pipelineDesc.fragmentFunction = ps_internal->function;
    }

    // 3. 버텍스 레이아웃 설정 (Input Layout)
    if (desc->il && !desc->il->elements.empty())
    {
        MTLVertexDescriptor* vertexDesc = [[MTLVertexDescriptor alloc] init];

        for (size_t i = 0; i < desc->il->elements.size(); ++i)
        {
            const auto& element = desc->il->elements[i];

            vertexDesc.attributes[i].format = _ConvertFormat_VertexInput(element.format);
            vertexDesc.attributes[i].offset = element.aligned_byte_offset;
            vertexDesc.attributes[i].bufferIndex = element.input_slot;
        }

        // 버텍스 버퍼 레이아웃
        vertexDesc.layouts[0].stride = /* 버텍스 크기 계산 */;
        vertexDesc.layouts[0].stepRate = 1;
        vertexDesc.layouts[0].stepFunction = MTLVertexStepFunctionPerVertex;

        pipelineDesc.vertexDescriptor = vertexDesc;
    }

    // 4. 렌더 타겟 포맷 설정
    if (renderpass_info)
    {
        for (uint32_t i = 0; i < renderpass_info->rt_count; ++i)
        {
            pipelineDesc.colorAttachments[i].pixelFormat =
                _ConvertFormat(renderpass_info->rt_formats[i]);
        }
        if (renderpass_info->ds_format != Format::UNKNOWN)
        {
            pipelineDesc.depthAttachmentPixelFormat =
                _ConvertFormat(renderpass_info->ds_format);
        }
    }

    // 5. 블렌드 상태 설정
    if (desc->bs)
    {
        for (size_t i = 0; i < desc->bs->render_target.size(); ++i)
        {
            const auto& rt = desc->bs->render_target[i];
            auto& attachment = pipelineDesc.colorAttachments[i];

            attachment.blendingEnabled = rt.blend_enable;
            attachment.sourceRGBBlendFactor = _ConvertBlend(rt.src_blend);
            attachment.destinationRGBBlendFactor = _ConvertBlend(rt.dest_blend);
            attachment.rgbBlendOperation = _ConvertBlendOp(rt.blend_op);
            attachment.sourceAlphaBlendFactor = _ConvertBlend(rt.src_blend_alpha);
            attachment.destinationAlphaBlendFactor = _ConvertBlend(rt.dest_blend_alpha);
            attachment.alphaBlendOperation = _ConvertBlendOp(rt.blend_op_alpha);
            attachment.writeMask = _ConvertColorWriteMask(rt.render_target_write_mask);
        }
    }

    // 6. 파이프라인 상태 생성
    NSError* error = nil;
    internal_state->renderPipeline = [device newRenderPipelineStateWithDescriptor:pipelineDesc
                                                                            error:&error];

    // 7. 깊이-스텐실 상태 (별도 객체)
    if (desc->dss)
    {
        MTLDepthStencilDescriptor* dsDesc = [[MTLDepthStencilDescriptor alloc] init];
        dsDesc.depthWriteEnabled = desc->dss->depth_write_mask == DepthWriteMask::ALL;
        dsDesc.depthCompareFunction = _ConvertComparisonFunc(desc->dss->depth_func);
        // ... 스텐실 설정

        internal_state->depthStencilState = [device newDepthStencilStateWithDescriptor:dsDesc];
    }

    // 8. 래스터라이저 상태 저장 (인코더에서 적용)
    if (desc->rs)
    {
        internal_state->cullMode = _ConvertCullMode(desc->rs->cull_mode);
        internal_state->frontFace = desc->rs->front_counter_clockwise ?
            MTLWindingCounterClockwise : MTLWindingClockwise;
        internal_state->fillMode = desc->rs->fill_mode == FillMode::WIREFRAME ?
            MTLTriangleFillModeLines : MTLTriangleFillModeFill;
    }

    // 9. 프리미티브 타입 저장
    internal_state->primitiveType = _ConvertPrimitiveTopology(desc->pt);

    return true;
}
```

---

## 9. 렌더링 파이프라인

### 9.1 렌더패스 시작 (RenderPassBegin)

**RenderPass**란?
- 렌더 타겟(프레임버퍼)에 그리기 위한 컨텍스트
- Metal에서는 `MTLRenderCommandEncoder` 생성

#### SwapChain을 렌더 타겟으로 사용:
```objc
void GraphicsDevice_Metal::RenderPassBegin(const SwapChain* swapchain, CommandList cmd)
{
    auto swapchain_internal = to_internal<SwapChain_Metal>(swapchain);
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    // 1. 다음 drawable 획득 (화면에 표시될 텍스처)
    swapchain_internal->currentDrawable = [swapchain_internal->metalLayer nextDrawable];

    // 2. 렌더패스 디스크립터 생성
    MTLRenderPassDescriptor* renderPassDesc = [MTLRenderPassDescriptor renderPassDescriptor];

    // 3. 컬러 어태치먼트 설정
    MTLRenderPassColorAttachmentDescriptor* colorAttachment =
        renderPassDesc.colorAttachments[0];
    colorAttachment.texture = swapchain_internal->currentDrawable.texture;
    colorAttachment.loadAction = MTLLoadActionClear;  // 클리어 후 시작
    colorAttachment.storeAction = MTLStoreActionStore; // 결과 저장
    colorAttachment.clearColor = MTLClearColorMake(
        swapchain_internal->desc.clear_color[0],  // R
        swapchain_internal->desc.clear_color[1],  // G
        swapchain_internal->desc.clear_color[2],  // B
        swapchain_internal->desc.clear_color[3]); // A

    // 4. 렌더 인코더 생성
    cmdList->renderEncoder = [cmdList->commandBuffer
        renderCommandEncoderWithDescriptor:renderPassDesc];
    cmdList->renderEncoder.label = @"SwapChainRenderEncoder";
    cmdList->isInsideRenderPass = true;

    // 5. drawable 저장 (Present용)
    cmdList->currentDrawable = swapchain_internal->currentDrawable;
}
```

#### 커스텀 렌더 타겟 사용:
```objc
void GraphicsDevice_Metal::RenderPassBegin(const RenderPassImage* images,
                                           uint32_t image_count,
                                           CommandList cmd,
                                           RenderPassFlags flags)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    MTLRenderPassDescriptor* renderPassDesc = [MTLRenderPassDescriptor renderPassDescriptor];
    uint32_t colorAttachmentIndex = 0;

    for (uint32_t i = 0; i < image_count; ++i)
    {
        const RenderPassImage& image = images[i];
        auto tex_internal = to_internal<Texture_Metal>(image.texture);

        switch (image.type)
        {
        case RenderPassImage::Type::RENDERTARGET:
        {
            auto& attachment = renderPassDesc.colorAttachments[colorAttachmentIndex];
            attachment.texture = tex_internal->texture;

            // Load Action (시작 시 동작)
            switch (image.loadop)
            {
            case RenderPassImage::LoadOp::LOAD:     // 기존 내용 유지
                attachment.loadAction = MTLLoadActionLoad;
                break;
            case RenderPassImage::LoadOp::CLEAR:    // 클리어
                attachment.loadAction = MTLLoadActionClear;
                attachment.clearColor = MTLClearColorMake(...);
                break;
            case RenderPassImage::LoadOp::DONTCARE: // 상관없음 (최적화)
                attachment.loadAction = MTLLoadActionDontCare;
                break;
            }

            // Store Action (끝날 때 동작)
            attachment.storeAction = (image.storeop == RenderPassImage::StoreOp::STORE) ?
                MTLStoreActionStore : MTLStoreActionDontCare;

            colorAttachmentIndex++;
            break;
        }
        case RenderPassImage::Type::DEPTH_STENCIL:
        {
            renderPassDesc.depthAttachment.texture = tex_internal->texture;
            // ... 깊이/스텐실 설정
            break;
        }
        }
    }

    cmdList->renderEncoder = [cmdList->commandBuffer
        renderCommandEncoderWithDescriptor:renderPassDesc];
    cmdList->isInsideRenderPass = true;
}
```

### 9.2 상태 바인딩

#### 파이프라인 바인딩:
```objc
void GraphicsDevice_Metal::BindPipelineState(const PipelineState* pso, CommandList cmd)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);
    auto pso_internal = to_internal<PipelineState_Metal>(pso);

    cmdList->boundPipeline = pso_internal;

    if (cmdList->renderEncoder)
    {
        // 렌더 파이프라인 상태
        [cmdList->renderEncoder setRenderPipelineState:pso_internal->renderPipeline];

        // 깊이-스텐실 상태
        if (pso_internal->depthStencilState)
        {
            [cmdList->renderEncoder setDepthStencilState:pso_internal->depthStencilState];
        }

        // 래스터라이저 상태 (인코더에서 직접 설정)
        [cmdList->renderEncoder setCullMode:pso_internal->cullMode];
        [cmdList->renderEncoder setFrontFacingWinding:pso_internal->frontFace];
        [cmdList->renderEncoder setTriangleFillMode:pso_internal->fillMode];
        [cmdList->renderEncoder setDepthBias:pso_internal->depthBias
            slopeScale:pso_internal->slopeScaledDepthBias
            clamp:pso_internal->depthBiasClamp];
    }
}
```

#### 버텍스 버퍼 바인딩:
```objc
void GraphicsDevice_Metal::BindVertexBuffers(const GPUBuffer* const* vertexBuffers,
                                             uint32_t slot,
                                             uint32_t count,
                                             const uint32_t* strides,
                                             const uint64_t* offsets,
                                             CommandList cmd)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    if (cmdList->renderEncoder)
    {
        for (uint32_t i = 0; i < count; ++i)
        {
            if (vertexBuffers[i] && vertexBuffers[i]->internal_state)
            {
                auto buffer_internal = to_internal<Resource_Metal>(vertexBuffers[i]);
                uint64_t offset = offsets ? offsets[i] : 0;

                // Metal에서 버텍스 버퍼는 버퍼 인덱스로 바인딩
                [cmdList->renderEncoder setVertexBuffer:buffer_internal->buffer
                                                 offset:offset
                                                atIndex:slot + i];
            }
        }
    }
}
```

#### 뷰포트 및 시저 설정:
```objc
void GraphicsDevice_Metal::BindViewports(uint32_t NumViewports,
                                         const Viewport* pViewports,
                                         CommandList cmd)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    if (cmdList->renderEncoder)
    {
        cmdList->viewports.resize(NumViewports);
        for (uint32_t i = 0; i < NumViewports; ++i)
        {
            cmdList->viewports[i].originX = pViewports[i].top_left_x;
            cmdList->viewports[i].originY = pViewports[i].top_left_y;
            cmdList->viewports[i].width = pViewports[i].width;
            cmdList->viewports[i].height = pViewports[i].height;
            cmdList->viewports[i].znear = pViewports[i].min_depth;
            cmdList->viewports[i].zfar = pViewports[i].max_depth;
        }
        [cmdList->renderEncoder setViewports:cmdList->viewports.data() count:NumViewports];
    }
}
```

### 9.3 Draw 명령

#### 기본 Draw:
```objc
void GraphicsDevice_Metal::Draw(uint32_t vertexCount,
                                uint32_t startVertexLocation,
                                CommandList cmd)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    if (cmdList->renderEncoder && cmdList->boundPipeline)
    {
        [cmdList->renderEncoder drawPrimitives:cmdList->boundPipeline->primitiveType
                                   vertexStart:startVertexLocation
                                   vertexCount:vertexCount];
    }
}
```

#### 인덱스 Draw:
```objc
void GraphicsDevice_Metal::DrawIndexed(uint32_t indexCount,
                                       uint32_t startIndexLocation,
                                       int32_t baseVertexLocation,
                                       CommandList cmd)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    if (cmdList->renderEncoder && cmdList->boundPipeline && cmdList->boundIndexBuffer)
    {
        uint32_t indexSize = (cmdList->indexType == MTLIndexTypeUInt16) ? 2 : 4;
        uint64_t indexBufferOffset = cmdList->indexBufferOffset + startIndexLocation * indexSize;

        [cmdList->renderEncoder drawIndexedPrimitives:cmdList->boundPipeline->primitiveType
                                           indexCount:indexCount
                                            indexType:cmdList->indexType
                                          indexBuffer:cmdList->boundIndexBuffer
                                    indexBufferOffset:indexBufferOffset
                                        instanceCount:1
                                           baseVertex:baseVertexLocation
                                         baseInstance:0];
    }
}
```

#### 인스턴스 Draw:
```objc
void GraphicsDevice_Metal::DrawInstanced(uint32_t vertexCount,
                                         uint32_t instanceCount,
                                         uint32_t startVertexLocation,
                                         uint32_t startInstanceLocation,
                                         CommandList cmd)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    if (cmdList->renderEncoder && cmdList->boundPipeline)
    {
        [cmdList->renderEncoder drawPrimitives:cmdList->boundPipeline->primitiveType
                                   vertexStart:startVertexLocation
                                   vertexCount:vertexCount
                                 instanceCount:instanceCount
                                  baseInstance:startInstanceLocation];
    }
}
```

### 9.4 렌더패스 종료

```objc
void GraphicsDevice_Metal::RenderPassEnd(CommandList cmd)
{
    CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);

    if (cmdList->renderEncoder)
    {
        [cmdList->renderEncoder endEncoding];  // 인코더 종료
        cmdList->renderEncoder = nil;
    }

    cmdList->isInsideRenderPass = false;
    cmdList->boundPipeline = nullptr;
}
```

---

## 10. 커맨드 리스트와 프레임 동기화

### 10.1 커맨드 리스트 개념

**커맨드 리스트**란?
- GPU에 보낼 명령들을 기록하는 버퍼
- 렌더링 명령, 복사 명령, 컴퓨트 명령 등 포함
- Metal에서는 `MTLCommandBuffer` + 인코더들

```
┌─────────────────────────────────────────────────────────┐
│                    CommandList                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  MTLCommandBuffer                                │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  MTLRenderCommandEncoder (렌더링 명령)    │   │   │
│  │  │  - setRenderPipelineState               │   │   │
│  │  │  - setVertexBuffer                      │   │   │
│  │  │  - drawPrimitives                       │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  MTLBlitCommandEncoder (복사 명령)        │   │   │
│  │  │  - copyFromBuffer                        │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  │  ┌─────────────────────────────────────────┐   │   │
│  │  │  MTLComputeCommandEncoder (컴퓨트 명령)   │   │   │
│  │  │  - setComputePipelineState              │   │   │
│  │  │  - dispatchThreadgroups                 │   │   │
│  │  └─────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 10.2 CommandList 내부 구조

```cpp
struct CommandList_Metal
{
    id<MTLCommandBuffer> commandBuffer = nil;
    id<MTLRenderCommandEncoder> renderEncoder = nil;
    id<MTLComputeCommandEncoder> computeEncoder = nil;
    id<MTLBlitCommandEncoder> blitEncoder = nil;

    QUEUE_TYPE queueType = QUEUE_GRAPHICS;
    bool isInsideRenderPass = false;

    // 현재 바인딩된 상태
    const PipelineState_Metal* boundPipeline = nullptr;
    id<MTLBuffer> boundIndexBuffer = nil;
    MTLIndexType indexType = MTLIndexTypeUInt16;
    uint64_t indexBufferOffset = 0;

    // 뷰포트/시저
    std::vector<MTLViewport> viewports;
    std::vector<MTLScissorRect> scissorRects;

    // Present용 drawable
    id<CAMetalDrawable> currentDrawable = nil;

    void Reset()
    {
        commandBuffer = nil;
        renderEncoder = nil;
        computeEncoder = nil;
        blitEncoder = nil;
        isInsideRenderPass = false;
        boundPipeline = nullptr;
        boundIndexBuffer = nil;
        currentDrawable = nil;
        viewports.clear();
        scissorRects.clear();
    }
};
```

### 10.3 BeginCommandList

```objc
CommandList GraphicsDevice_Metal::BeginCommandList(QUEUE_TYPE queue)
{
    std::lock_guard<std::mutex> lock(cmdListMutex);

    // 1. 커맨드 리스트 인덱스 할당
    uint32_t index = nextCommandListIndex++;
    if (index >= commandLists.size())
    {
        commandLists.resize(index + 1);
        frameAllocators.resize(index + 1);
    }

    // 2. 커맨드 리스트 초기화
    CommandList_Metal& cmdList = commandLists[index];
    cmdList.Reset();
    cmdList.queueType = queue;

    // 3. Metal 커맨드 버퍼 생성
    cmdList.commandBuffer = [commandQueue commandBuffer];
    cmdList.commandBuffer.label = @"CommandList";

    // 4. 핸들 반환
    CommandList cmd;
    cmd.internal_state = &cmdList;
    activeCommandLists.push_back(cmd);

    return cmd;
}
```

### 10.4 트리플 버퍼링과 프레임 동기화

**트리플 버퍼링**이란?
- 3개의 프레임 버퍼를 번갈아 사용
- GPU가 프레임 N을 렌더링하는 동안, CPU는 프레임 N+1을 준비
- 끊김 없는 렌더링과 CPU-GPU 병렬 처리

```
시간 →
───────────────────────────────────────────────────────
CPU:  [Frame 0 준비] [Frame 1 준비] [Frame 2 준비] [Frame 0 준비] ...
GPU:              [Frame 0 렌더] [Frame 1 렌더] [Frame 2 렌더] ...
화면:                          [Frame 0 표시] [Frame 1 표시] ...
───────────────────────────────────────────────────────
```

**세마포어로 동기화:**
```objc
// 초기화 시: 3개 슬롯으로 세마포어 생성
frameSemaphore = dispatch_semaphore_create(BUFFERCOUNT);  // BUFFERCOUNT = 2 or 3

// 프레임 시작 시: 슬롯 대기
dispatch_semaphore_wait(frameSemaphore, DISPATCH_TIME_FOREVER);

// 프레임 완료 시: 슬롯 반환
[commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
    dispatch_semaphore_signal(frameSemaphore);
}];
```

### 10.5 SubmitCommandLists

```objc
void GraphicsDevice_Metal::SubmitCommandLists()
{
    std::lock_guard<std::mutex> lock(cmdListMutex);

    // 1. 프레임 슬롯 대기 (트리플 버퍼링)
    dispatch_semaphore_wait(frameSemaphore, DISPATCH_TIME_FOREVER);

    uint64_t currentFrame = FRAMECOUNT;

    // 2. 모든 활성 커맨드 리스트 처리
    for (auto& cmd : activeCommandLists)
    {
        CommandList_Metal* cmdList = static_cast<CommandList_Metal*>(cmd.internal_state);
        if (cmdList && cmdList->commandBuffer)
        {
            // 3. 활성 인코더 종료
            if (cmdList->renderEncoder)
            {
                [cmdList->renderEncoder endEncoding];
                cmdList->renderEncoder = nil;
            }
            if (cmdList->computeEncoder)
            {
                [cmdList->computeEncoder endEncoding];
                cmdList->computeEncoder = nil;
            }
            if (cmdList->blitEncoder)
            {
                [cmdList->blitEncoder endEncoding];
                cmdList->blitEncoder = nil;
            }

            // 4. SwapChain drawable 표시
            if (cmdList->currentDrawable)
            {
                [cmdList->commandBuffer presentDrawable:cmdList->currentDrawable];
                cmdList->currentDrawable = nil;
            }

            // 5. 완료 핸들러 등록 (세마포어 신호 + 리소스 정리)
            __block dispatch_semaphore_t sem = frameSemaphore;
            __block uint64_t frame = currentFrame;
            __block GraphicsDevice_Metal* self = this;

            [cmdList->commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
                dispatch_semaphore_signal(sem);           // 슬롯 반환
                self->completedFrameCount = frame;        // 완료 프레임 기록
                self->allocationHandler.Update(frame);    // 지연 해제 처리
            }];

            // 6. GPU에 제출
            [cmdList->commandBuffer commit];
        }
    }

    // 7. 다음 프레임 준비
    activeCommandLists.clear();
    nextCommandListIndex = 0;

    // 8. 프레임 할당자 리셋
    for (auto& allocator : frameAllocators)
    {
        allocator.reset();
    }

    FRAMECOUNT++;
}
```

### 10.6 지연 해제 (Deferred Destruction)

GPU가 아직 사용 중인 리소스를 즉시 해제하면 크래시가 발생합니다.
지연 해제 시스템은 리소스를 몇 프레임 후에 안전하게 해제합니다.

```cpp
struct AllocationHandler_Metal
{
    std::mutex mutex;
    std::deque<std::pair<uint64_t, id<MTLBuffer>>> destroyedBuffers;
    std::deque<std::pair<uint64_t, id<MTLTexture>>> destroyedTextures;

    // 리소스를 해제 대기열에 추가
    void DeferredDestroy(id<MTLBuffer> buffer, uint64_t frameCount)
    {
        std::lock_guard<std::mutex> lock(mutex);
        destroyedBuffers.push_back({frameCount, buffer});
    }

    // 완료된 프레임의 리소스 해제
    void Update(uint64_t completedFrame)
    {
        std::lock_guard<std::mutex> lock(mutex);

        // completedFrame 이전에 등록된 리소스는 안전하게 해제 가능
        while (!destroyedBuffers.empty() &&
               destroyedBuffers.front().first <= completedFrame)
        {
            destroyedBuffers.pop_front();  // ARC가 자동 해제
        }
        // ... 텍스처도 동일하게 처리
    }
};
```

---

## 11. Metal Triangle 샘플

### 11.1 샘플 개요

Metal 백엔드가 제대로 작동하는지 검증하기 위한 기본 샘플입니다.
RGB 컬러의 삼각형 하나를 화면에 렌더링합니다.

**샘플 위치:** `Examples/MetalTriangleSample/main.mm`

```
┌────────────────────────────────┐
│                                │
│         🔴 (빨강)               │
│        /    \                  │
│       /      \                 │
│      /        \                │
│     /          \               │
│    🟢──────────🔵              │
│  (녹색)      (파랑)             │
│                                │
└────────────────────────────────┘
```

### 11.2 MSL 셰이더 (Metal Shading Language)

```metal
#include <metal_stdlib>
using namespace metal;

// 버텍스 입력 구조체
struct VertexIn
{
    float2 position [[attribute(0)]];  // 위치 (x, y)
    float3 color    [[attribute(1)]];  // 컬러 (r, g, b)
};

// 버텍스 출력 / 프래그먼트 입력 구조체
struct VertexOut
{
    float4 position [[position]];  // 클립 공간 위치 (필수)
    float3 color;                  // 보간될 컬러
};

// 버텍스 셰이더
vertex VertexOut vertexMain(VertexIn in [[stage_in]])
{
    VertexOut out;
    // 2D 위치를 4D 클립 공간으로 변환
    out.position = float4(in.position, 0.0, 1.0);
    out.color = in.color;  // 컬러 전달
    return out;
}

// 프래그먼트(픽셀) 셰이더
fragment float4 fragmentMain(VertexOut in [[stage_in]])
{
    // RGB + 불투명 알파
    return float4(in.color, 1.0);
}
```

**MSL 특징:**
- `[[attribute(N)]]`: 버텍스 버퍼의 N번째 속성
- `[[position]]`: 래스터라이저에 전달될 클립 공간 위치
- `[[stage_in]]`: 이전 스테이지에서 입력 받음
- C++11 스타일 문법 기반

### 11.3 버텍스 데이터

```cpp
// 버텍스 구조체 정의
struct Vertex
{
    float position[2];  // x, y 좌표
    float color[3];     // r, g, b 컬러
};

// 삼각형 버텍스 데이터 (NDC 좌표계: -1 ~ +1)
Vertex vertices[] = {
    { {  0.0f,  0.5f }, { 1.0f, 0.0f, 0.0f } },  // Top - Red
    { { -0.5f, -0.5f }, { 0.0f, 1.0f, 0.0f } },  // Bottom Left - Green
    { {  0.5f, -0.5f }, { 0.0f, 0.0f, 1.0f } },  // Bottom Right - Blue
};
```

**NDC (Normalized Device Coordinates):**
```
        +Y (0.5)
          │
          │    🔴
          │   /  \
  -X ─────┼─────── +X
   (-0.5) │ (0.5)
         🟢─────🔵
          │
        -Y (-0.5)
```

### 11.4 전체 샘플 코드 흐름

```objc
@implementation AppDelegate

- (void)applicationDidFinishLaunching:(NSNotification*)notification
{
    // ========== 1. 윈도우 생성 ==========
    NSRect frame = NSMakeRect(100, 100, 800, 600);
    self.window = [[NSWindow alloc] initWithContentRect:frame
                                              styleMask:...
                                                backing:NSBackingStoreBuffered
                                                  defer:NO];
    [self.window setTitle:@"Metal Triangle Sample"];
    [self.window makeKeyAndOrderFront:nil];

    // ========== 2. Graphics Device 초기화 ==========
    _device = new GraphicsDevice_Metal();
    if (!_device->Initialize(ValidationMode::Disabled, GPUPreference::Discrete))
    {
        NSLog(@"Failed to initialize Metal device");
        return;
    }
    NSLog(@"Metal device initialized: %s", _device->GetAdapterName().c_str());

    // ========== 3. SwapChain 생성 ==========
    SwapChainDesc swapChainDesc = {};
    swapChainDesc.width = 800;
    swapChainDesc.height = 600;
    swapChainDesc.buffer_count = 2;
    swapChainDesc.format = Format::B8G8R8A8_UNORM;
    swapChainDesc.vsync = true;
    swapChainDesc.clear_color[0] = 0.2f;  // 어두운 파란 배경
    swapChainDesc.clear_color[1] = 0.2f;
    swapChainDesc.clear_color[2] = 0.3f;
    swapChainDesc.clear_color[3] = 1.0f;

    _device->CreateSwapChain(&swapChainDesc, (__bridge void*)contentView, &_swapChain);

    // ========== 4. Vertex Buffer 생성 ==========
    Vertex vertices[] = {
        { {  0.0f,  0.5f }, { 1.0f, 0.0f, 0.0f } },
        { { -0.5f, -0.5f }, { 0.0f, 1.0f, 0.0f } },
        { {  0.5f, -0.5f }, { 0.0f, 0.0f, 1.0f } },
    };

    GPUBufferDesc bufferDesc = {};
    bufferDesc.size = sizeof(vertices);
    bufferDesc.usage = Usage::UPLOAD;
    bufferDesc.bind_flags = BindFlag::VERTEX_BUFFER;
    _device->CreateBuffer(&bufferDesc, vertices, &_vertexBuffer);

    // ========== 5. Shader 생성 ==========
    _device->CreateShader(ShaderStage::VS, g_ShaderSource, strlen(g_ShaderSource), &_vertexShader);
    _device->CreateShader(ShaderStage::PS, g_ShaderSource, strlen(g_ShaderSource), &_pixelShader);

    // ========== 6. Input Layout 정의 ==========
    InputLayout inputLayout;
    inputLayout.elements.resize(2);

    // Position 속성
    inputLayout.elements[0].semantic_name = "POSITION";
    inputLayout.elements[0].format = Format::R32G32_FLOAT;
    inputLayout.elements[0].input_slot = 0;
    inputLayout.elements[0].aligned_byte_offset = offsetof(Vertex, position);

    // Color 속성
    inputLayout.elements[1].semantic_name = "COLOR";
    inputLayout.elements[1].format = Format::R32G32B32_FLOAT;
    inputLayout.elements[1].input_slot = 0;
    inputLayout.elements[1].aligned_byte_offset = offsetof(Vertex, color);

    // ========== 7. Pipeline State 생성 ==========
    RenderPassInfo renderPassInfo = {};
    renderPassInfo.rt_count = 1;
    renderPassInfo.rt_formats[0] = Format::B8G8R8A8_UNORM;
    renderPassInfo.sample_count = 1;

    PipelineStateDesc psoDesc = {};
    psoDesc.vs = &_vertexShader;
    psoDesc.ps = &_pixelShader;
    psoDesc.il = &inputLayout;
    psoDesc.pt = PrimitiveTopology::TRIANGLELIST;

    _device->CreatePipelineState(&psoDesc, &_pipelineState, &renderPassInfo);

    // ========== 8. 렌더 타이머 시작 (60 FPS) ==========
    self.renderTimer = [NSTimer scheduledTimerWithTimeInterval:1.0/60.0
                                                        target:self
                                                      selector:@selector(render)
                                                      userInfo:nil
                                                       repeats:YES];
}

- (void)render
{
    @autoreleasepool
    {
        // ===== 렌더링 루프 =====

        // 1. 커맨드 리스트 시작
        CommandList cmd = _device->BeginCommandList(QUEUE_GRAPHICS);

        // 2. 렌더패스 시작 (SwapChain 렌더타겟)
        _device->RenderPassBegin(&_swapChain, cmd);

        // 3. 뷰포트 설정
        Viewport viewport = {};
        viewport.width = (float)_swapChain.desc.width;
        viewport.height = (float)_swapChain.desc.height;
        viewport.min_depth = 0.0f;
        viewport.max_depth = 1.0f;
        _device->BindViewports(1, &viewport, cmd);

        // 4. 시저 사각형 설정
        Rect scissorRect = {};
        scissorRect.right = _swapChain.desc.width;
        scissorRect.bottom = _swapChain.desc.height;
        _device->BindScissorRects(1, &scissorRect, cmd);

        // 5. 파이프라인 바인딩
        _device->BindPipelineState(&_pipelineState, cmd);

        // 6. 버텍스 버퍼 바인딩
        const GPUBuffer* vertexBuffers[] = { &_vertexBuffer };
        uint32_t strides[] = { sizeof(Vertex) };
        uint64_t offsets[] = { 0 };
        _device->BindVertexBuffers(vertexBuffers, 0, 1, strides, offsets, cmd);

        // 7. 삼각형 그리기 (3 vertices, start at 0)
        _device->Draw(3, 0, cmd);

        // 8. 렌더패스 종료
        _device->RenderPassEnd(cmd);

        // 9. 커맨드 제출 및 Present
        _device->SubmitCommandLists();
    }
}

@end
```

### 11.5 빌드 및 실행

```bash
# 1. 프로젝트 디렉토리로 이동
cd Examples/MetalTriangleSample

# 2. CMake 설정
mkdir build && cd build
cmake ..

# 3. 빌드
make

# 4. 실행
./MetalTriangleSample
```

**예상 결과:**
- 800x600 윈도우가 열림
- 어두운 파란 배경
- 중앙에 RGB 그라데이션 삼각형 표시

---

## 12. 현재 구현 상태

### 구현 완료 ✅

| 카테고리 | 항목 | Metal API |
|---------|------|-----------|
| **초기화** | 디바이스 생성 | MTLCreateSystemDefaultDevice |
| | 커맨드 큐 생성 | [device newCommandQueue] |
| | 프레임 동기화 | dispatch_semaphore |
| **리소스** | SwapChain | CAMetalLayer |
| | Buffer (UPLOAD/DEFAULT) | MTLBuffer |
| | Texture (2D/3D/Cube) | MTLTexture |
| | Sampler | MTLSamplerState |
| | Shader (MSL) | MTLLibrary + MTLFunction |
| **파이프라인** | RenderPipelineState | MTLRenderPipelineState |
| | DepthStencilState | MTLDepthStencilState |
| | Input Layout | MTLVertexDescriptor |
| | Blend State | colorAttachment 설정 |
| **렌더링** | RenderPassBegin/End | MTLRenderCommandEncoder |
| | BindViewports | setViewports |
| | BindScissorRects | setScissorRects |
| | BindPipelineState | setRenderPipelineState |
| | BindVertexBuffers | setVertexBuffer |
| | BindIndexBuffer | indexBuffer 저장 |
| | BindConstantBuffer | setVertexBuffer/setFragmentBuffer |
| | BindResource (Texture) | setVertexTexture/setFragmentTexture |
| | BindSampler | setSamplerState |
| **드로우** | Draw | drawPrimitives |
| | DrawIndexed | drawIndexedPrimitives |
| | DrawInstanced | drawPrimitives (instanceCount) |
| | DrawIndexedInstanced | drawIndexedPrimitives (instanceCount) |
| **복사** | CopyBuffer | copyFromBuffer (Blit) |
| **디버그** | EventBegin/End | pushDebugGroup/popDebugGroup |
| | SetMarker | insertDebugSignpost |

### 미구현 / 스텁 ⏳

| 카테고리 | 항목 | 우선순위 |
|---------|------|----------|
| **컴퓨트** | Dispatch | 중간 |
| | DispatchIndirect | 낮음 |
| | BindComputeShader | 중간 |
| **Indirect Draw** | DrawInstancedIndirect | 낮음 |
| | DrawIndexedInstancedIndirect | 낮음 |
| | DrawIndirectCount | 낮음 |
| **쿼리** | QueryBegin/End | 낮음 |
| | QueryResolve | 낮음 |
| **기타** | Barrier | 중간 |
| | ClearUAV | 낮음 |
| | PushConstants | 중간 |
| | CopyTexture | 중간 |
| | Subresources | 중간 |

---

## 13. 빌드 방법

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
- 빌드 → GBackendMetal.dll 생성 (스텁, macOS에서만 실제 동작)

### CMakeLists.txt 핵심 내용
```cmake
# macOS에서만 Metal 프레임워크 링크
if(APPLE)
    find_library(METAL_FRAMEWORK Metal REQUIRED)
    find_library(METALKIT_FRAMEWORK MetalKit REQUIRED)
    find_library(FOUNDATION_FRAMEWORK Foundation REQUIRED)
    find_library(APPKIT_FRAMEWORK AppKit REQUIRED)
    find_library(QUARTZCORE_FRAMEWORK QuartzCore REQUIRED)

    target_link_libraries(GBackendMetal PRIVATE
        ${METAL_FRAMEWORK}
        ${METALKIT_FRAMEWORK}
        ${FOUNDATION_FRAMEWORK}
        ${APPKIT_FRAMEWORK}
        ${QUARTZCORE_FRAMEWORK}
    )

    # ARC(Automatic Reference Counting) 활성화
    target_compile_options(GBackendMetal PRIVATE -fobjc-arc)
endif()
```

---

## 14. 다음 단계

### 구현 로드맵

#### Phase 1: 기본 렌더링 ✅ 완료
- [x] SwapChain
- [x] Buffer
- [x] Texture
- [x] Shader (MSL 소스 컴파일)
- [x] Pipeline State
- [x] CommandList
- [x] Draw / DrawIndexed / DrawInstanced
- [x] 삼각형 샘플

#### Phase 2: 고급 렌더링 🔄 진행 중
- [ ] Compute Shader (MTLComputeCommandEncoder)
- [ ] Indirect Draw
- [ ] CopyTexture
- [ ] Barrier (메모리 동기화)
- [ ] Push Constants

#### Phase 3: 엔진 통합
- [ ] SPIRV-Cross로 HLSL → MSL 변환
- [ ] 엔진 렌더러와 연동 테스트
- [ ] 성능 최적화

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
| ID3D12GraphicsCommandList | id\<MTLRenderCommandEncoder\> |
| ID3D12Resource (Buffer) | id\<MTLBuffer\> |
| ID3D12Resource (Texture) | id\<MTLTexture\> |
| ID3D12PipelineState | id\<MTLRenderPipelineState\> |
| D3D12_RENDER_TARGET_VIEW | MTLRenderPassDescriptor |
| Root Signature | Argument Buffer (Metal 3) |
| Resource Barrier | MTLFence / MTLEvent |

---

## 부록: 용어 정리

| 용어 | 설명 |
|------|------|
| **Backend** | 특정 그래픽스 API(DX12, Metal 등)의 구현체 |
| **SwapChain** | 화면에 표시할 프레임 버퍼들의 집합 |
| **CommandList** | GPU에 보낼 명령들을 기록하는 버퍼 |
| **Pipeline State** | 셰이더, 블렌딩, 깊이테스트 등 렌더링 설정 묶음 |
| **RenderPass** | 렌더 타겟을 설정하고 그리기 명령을 실행하는 단위 |
| **Barrier** | GPU 리소스 접근 동기화를 위한 메모리 배리어 |
| **MTL** | Metal의 접두사 |
| **MSL** | Metal Shading Language |
| **ARC** | Automatic Reference Counting, Objective-C 메모리 관리 |
| **NDC** | Normalized Device Coordinates, -1~+1 범위의 좌표계 |
| **Drawable** | 화면에 표시될 텍스처 (CAMetalDrawable) |
| **Triple Buffering** | 3개의 프레임 버퍼를 사용한 끊김 없는 렌더링 |
| **Staging Buffer** | CPU→GPU 데이터 전송을 위한 임시 버퍼 |
