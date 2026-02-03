# Part 3: GPU와 통신하기

Part 2에서 그래픽스 파이프라인이 어떻게 동작하는지 배웠다.
이제 **CPU에서 GPU에게 어떻게 명령을 내리는지** 알아보자.

이 부분이 DX12/Vulkan의 핵심이고, renderer.cpp를 이해하는 열쇠다.

---

## DX12 용어/약어 정리

코드를 읽기 전에 자주 나오는 접두사와 약어를 알아두자.

### 인터페이스 접두사

| 접두사 | 의미 | 예시 |
|--------|------|------|
| `ID3D12` | Direct3D 12 인터페이스 | `ID3D12Device`, `ID3D12Resource` |
| `IDXGI` | DirectX Graphics Infrastructure | `IDXGISwapChain`, `IDXGIAdapter` |
| `D3D12_` | DX12 구조체/열거형 | `D3D12_RESOURCE_DESC` |
| `DXGI_` | DXGI 구조체/열거형 | `DXGI_FORMAT_R8G8B8A8_UNORM` |

### 파이프라인 스테이지 약어

| 약어 | 전체 이름 | 역할 |
|------|-----------|------|
| `IA` | Input Assembler | 버텍스/인덱스 조립 |
| `VS` | Vertex Shader | 버텍스 변환 |
| `HS` | Hull Shader | 테셀레이션 제어 |
| `DS` | Domain Shader | 테셀레이션 결과 처리 |
| `GS` | Geometry Shader | 기하 생성/삭제 |
| `RS` | Rasterizer | 삼각형 → 픽셀 |
| `PS` | Pixel Shader | 픽셀 색상 결정 |
| `OM` | Output Merger | 깊이테스트, 블렌딩 |
| `CS` | Compute Shader | 범용 GPU 계산 |
| `SO` | Stream Output | 버텍스 데이터 출력 |

### 뷰(View) 약어

| 약어 | 전체 이름 | 용도 |
|------|-----------|------|
| `RTV` | Render Target View | 렌더 타겟에 쓰기 |
| `DSV` | Depth Stencil View | 깊이/스텐실 버퍼 |
| `SRV` | Shader Resource View | 셰이더에서 읽기 |
| `UAV` | Unordered Access View | 셰이더에서 읽기/쓰기 |
| `CBV` | Constant Buffer View | 상수 버퍼 접근 |

### 기타 중요 약어

| 약어 | 전체 이름 | 설명 |
|------|-----------|------|
| `PSO` | Pipeline State Object | 파이프라인 설정 묶음 |
| `RS` | Root Signature | 리소스 바인딩 레이아웃 |
| `GPU` | Graphics Processing Unit | 그래픽 처리 장치 |
| `VRAM` | Video RAM | GPU 전용 메모리 |
| `CB` | Constant Buffer | 상수 버퍼 |
| `VB` | Vertex Buffer | 버텍스 버퍼 |
| `IB` | Index Buffer | 인덱스 버퍼 |
| `RT` | Render Target | 렌더링 대상 버퍼 |
| `MRT` | Multiple Render Targets | 다중 렌더 타겟 |
| `MSAA` | Multi-Sample Anti-Aliasing | 다중 샘플 안티앨리어싱 |

### 동기화 관련 용어

| 용어 | 설명 |
|------|------|
| `Fence` | CPU-GPU 동기화 객체. GPU 작업 완료를 CPU가 확인 |
| `Semaphore` | GPU-GPU 동기화 객체. 큐 간 작업 순서 보장 |
| `Barrier` | 리소스 상태 전이 및 메모리 동기화 |
| `Signal` | 동기화 객체에 완료 신호 보내기 |
| `Wait` | 동기화 객체의 신호를 기다리기 |
| `Timeline Semaphore` | 카운터 값 기반 세마포어 (Vulkan 1.2+) |
| `Binary Semaphore` | 신호/대기 한 번씩 사용하는 세마포어 |

### 함수 이름 읽는 법

```cpp
commandList->IASetVertexBuffers(0, 1, &vbView);
//          IA  Set  VertexBuffers
//          │   │    └─ 무엇을: 버텍스 버퍼들
//          │   └─ 동작: 설정
//          └─ 어디에: Input Assembler

commandList->OMSetRenderTargets(1, &rtv, FALSE, &dsv);
//          OM  Set  RenderTargets
//          │   │    └─ 무엇을: 렌더 타겟들
//          │   └─ 동작: 설정
//          └─ 어디에: Output Merger

device->CreateCommittedResource(...);
//      Create  Committed  Resource
//      │       │          └─ 무엇을: 리소스
//      │       └─ 타입: Committed (자동 힙 할당)
//      └─ 동작: 생성
```

---

## 3.1 그래픽스 API 역할

### API란?

**A**pplication **P**rogramming **I**nterface
= 응용 프로그램이 GPU를 제어하기 위한 인터페이스

```
┌─────────────────────────────────────┐
│            너의 게임/엔진            │
├─────────────────────────────────────┤
│     그래픽스 API (DX12, Vulkan)     │  ← 여기서 공부할 것
├─────────────────────────────────────┤
│            GPU 드라이버             │
├─────────────────────────────────────┤
│            GPU 하드웨어             │
└─────────────────────────────────────┘
```

### 주요 API 비교

| API | 플랫폼 | 특징 |
|-----|--------|------|
| OpenGL | 크로스 플랫폼 | 오래됨, 배우기 쉬움, 드라이버 오버헤드 |
| DirectX 11 | Windows | OpenGL보다 효율적, 아직도 많이 사용 |
| **DirectX 12** | Windows, Xbox | 저수준, 명시적 제어, 최고 성능 |
| **Vulkan** | 크로스 플랫폼 | DX12와 비슷, 리눅스/안드로이드 지원 |
| Metal | Apple | Apple 전용, DX12/Vulkan과 비슷 |

Wicked Engine은 **DX12**를 기본으로 사용.
DX12와 Vulkan은 개념이 거의 같아서, 하나 배우면 다른 것도 쉽게 배울 수 있음.

### OpenGL vs DX12/Vulkan 철학 차이

**OpenGL/DX11 (암시적, Implicit)**:
```cpp
// "나 대신 알아서 해줘"
glBindTexture(GL_TEXTURE_2D, textureID);
glDrawArrays(GL_TRIANGLES, 0, 36);
// 드라이버가 동기화, 메모리 관리, 상태 관리 다 함
```

**DX12/Vulkan (명시적, Explicit)**:
```cpp
// "내가 다 알려줄게"
// 1. 메모리 직접 할당
device->CreateCommittedResource(...);

// 2. 리소스 상태 직접 전이
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;
commandList->ResourceBarrier(1, &barrier);

// 3. 커맨드 직접 기록하고 제출
commandList->DrawInstanced(36, 1, 0, 0);
commandQueue->ExecuteCommandLists(1, &commandList);

// 4. 동기화 직접 관리
commandQueue->Signal(fence, fenceValue);
fence->SetEventOnCompletion(fenceValue, event);
WaitForSingleObject(event, INFINITE);
```

**왜 더 복잡한 방식을 쓰나?**
- 드라이버 오버헤드 감소
- 멀티스레딩 지원
- 예측 가능한 성능
- 최적화 여지가 더 많음

---

## 3.2 기본 DX12 객체들

### Device

GPU를 나타내는 객체. 리소스 생성의 중심.

```cpp
// Device 생성
ID3D12Device* device;

// 어댑터(GPU) 선택
IDXGIAdapter1* adapter;
factory->EnumAdapters1(0, &adapter);  // 첫 번째 GPU

// Device 생성
D3D12CreateDevice(
    adapter,
    D3D_FEATURE_LEVEL_12_0,
    IID_PPV_ARGS(&device)
);
```

```
Device로 할 수 있는 것:
- 리소스(버퍼, 텍스처) 생성
- 파이프라인 스테이트 생성
- 커맨드 큐/리스트 생성
- 힙, 디스크립터 생성
```

### Command Queue

GPU에게 명령을 **제출**하는 큐.

```cpp
// Command Queue 생성
D3D12_COMMAND_QUEUE_DESC queueDesc = {};
queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;  // 그래픽스 + 컴퓨트
queueDesc.Priority = D3D12_COMMAND_QUEUE_PRIORITY_NORMAL;

ID3D12CommandQueue* commandQueue;
device->CreateCommandQueue(&queueDesc, IID_PPV_ARGS(&commandQueue));
```

큐 타입:
```
DIRECT:  그래픽스 + 컴퓨트 + 복사 (모든 명령)
COMPUTE: 컴퓨트 + 복사
COPY:    복사만 (데이터 전송)

여러 큐를 병렬로 사용 가능!
┌─────────────┐
│ Direct 큐   │ ─→ GPU 그래픽스 엔진
└─────────────┘
┌─────────────┐
│ Compute 큐  │ ─→ GPU 컴퓨트 엔진
└─────────────┘
┌─────────────┐
│ Copy 큐     │ ─→ GPU 복사 엔진
└─────────────┘
```

### Command Allocator

커맨드를 저장할 메모리 풀.

```cpp
// Command Allocator 생성
ID3D12CommandAllocator* commandAllocator;
device->CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    IID_PPV_ARGS(&commandAllocator)
);

// 중요: GPU가 다 쓴 후에만 Reset 가능!
// 프레임마다 새 allocator 사용하거나
// 펜스로 완료 대기 후 재사용
commandAllocator->Reset();
```

### Command List

실제 명령을 **기록**하는 객체.

```cpp
// Command List 생성
ID3D12GraphicsCommandList* commandList;
device->CreateCommandList(
    0,                           // Node mask
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    commandAllocator,            // Allocator
    nullptr,                     // Initial PSO
    IID_PPV_ARGS(&commandList)
);

// 명령 기록
commandList->SetPipelineState(pso);
commandList->SetGraphicsRootSignature(rootSig);
commandList->IASetVertexBuffers(0, 1, &vbView);
commandList->IASetIndexBuffer(&ibView);
commandList->DrawIndexedInstanced(36, 1, 0, 0, 0);

// 기록 종료
commandList->Close();

// 큐에 제출
ID3D12CommandList* lists[] = { commandList };
commandQueue->ExecuteCommandLists(1, lists);
```

### 전체 흐름

```
1. Command Allocator에서 메모리 할당
2. Command List에 명령 기록
3. Command List 닫기 (Close)
4. Command Queue에 제출 (Execute)
5. GPU가 비동기로 실행
6. 펜스로 완료 확인
7. Allocator/List 재사용

┌───────────────┐    ┌───────────────┐
│ CPU (메인)    │    │ GPU           │
│               │    │               │
│ 기록 ─────────┼───→│               │
│ 제출 ─────────┼───→│ 실행          │
│ (다른 작업)   │    │ 실행 중...    │
│ 펜스 대기 ←───┼────│ 완료          │
└───────────────┘    └───────────────┘
```

---

## 3.3 PSO (Pipeline State Object)

### 파이프라인 상태란?

렌더링에 필요한 **모든 설정을 하나로 묶은 것**.

```cpp
// DX11 방식 (상태가 분리됨)
context->VSSetShader(vertexShader);
context->PSSetShader(pixelShader);
context->RSSetState(rasterizerState);
context->OMSetBlendState(blendState);
context->OMSetDepthStencilState(depthState);
// 매번 호출할 때마다 드라이버가 검증해야 함

// DX12 방식 (상태가 하나로)
commandList->SetPipelineState(pso);
// 미리 검증됨, 빠름
```

### Graphics PSO 구성요소

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc = {};

// 1. 루트 시그니처 (리소스 바인딩 레이아웃)
psoDesc.pRootSignature = rootSignature;

// 2. 셰이더
psoDesc.VS = { vsBlob->GetBufferPointer(), vsBlob->GetBufferSize() };
psoDesc.PS = { psBlob->GetBufferPointer(), psBlob->GetBufferSize() };
// HS, DS, GS도 있으면 설정

// 3. 블렌드 상태
psoDesc.BlendState.RenderTarget[0].BlendEnable = TRUE;
psoDesc.BlendState.RenderTarget[0].SrcBlend = D3D12_BLEND_SRC_ALPHA;
psoDesc.BlendState.RenderTarget[0].DestBlend = D3D12_BLEND_INV_SRC_ALPHA;
// ...

// 4. 래스터라이저 상태
psoDesc.RasterizerState.FillMode = D3D12_FILL_MODE_SOLID;
psoDesc.RasterizerState.CullMode = D3D12_CULL_MODE_BACK;
psoDesc.RasterizerState.FrontCounterClockwise = TRUE;
// ...

// 5. 깊이/스텐실 상태
psoDesc.DepthStencilState.DepthEnable = TRUE;
psoDesc.DepthStencilState.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ALL;
psoDesc.DepthStencilState.DepthFunc = D3D12_COMPARISON_FUNC_LESS;
// ...

// 6. 입력 레이아웃
psoDesc.InputLayout = { inputElements, numElements };

// 7. 렌더 타겟 포맷
psoDesc.NumRenderTargets = 1;
psoDesc.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM;
psoDesc.DSVFormat = DXGI_FORMAT_D32_FLOAT;

// 8. 기타
psoDesc.SampleMask = UINT_MAX;
psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
psoDesc.SampleDesc.Count = 1;

// PSO 생성
ID3D12PipelineState* pso;
device->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&pso));
```

### Compute PSO

컴퓨트 셰이더용 PSO는 더 단순함.

```cpp
D3D12_COMPUTE_PIPELINE_STATE_DESC computePsoDesc = {};
computePsoDesc.pRootSignature = computeRootSig;
computePsoDesc.CS = { csBlob->GetBufferPointer(), csBlob->GetBufferSize() };

ID3D12PipelineState* computePso;
device->CreateComputePipelineState(&computePsoDesc, IID_PPV_ARGS(&computePso));
```

### PSO 캐싱

PSO 생성은 **비용이 큼** (셰이더 컴파일 + 검증).

```cpp
// 방법 1: 미리 생성 (로딩 화면)
void LoadAllPSOs() {
    for (each shader combination) {
        CreatePSO(...);
        psoCache[hash] = pso;
    }
}

// 방법 2: PSO 캐시 직렬화
// 한 번 만든 PSO를 파일로 저장
ID3DBlob* cachedBlob;
pso->GetCachedBlob(&cachedBlob);
SaveToFile("pso_cache.bin", cachedBlob);

// 다음 실행 시 캐시에서 로드
psoDesc.CachedPSO = { cachedData, cachedSize };
device->CreateGraphicsPipelineState(&psoDesc, ...);
// 훨씬 빠름!
```

### PSO 상태 변경 비용

```
비용 높음 ←─────────────────────────→ 비용 낮음

PSO 변경 > 루트 시그니처 > 디스크립터 > 상수

같은 PSO로 여러 물체 그리기 = 빠름
물체마다 PSO 변경 = 느림

→ 같은 재질(PSO)끼리 묶어서 그리기 (Batching)
```

---

## 3.4 루트 시그니처 (Root Signature)

### 루트 시그니처란?

셰이더가 어떤 리소스(텍스처, 버퍼 등)를 어떻게 접근할지 정의하는 **계약서**.

```
비유:
함수 선언 = 루트 시그니처
void render(Texture diffuse, Texture normal, Buffer lights);

함수 호출 = 실제 바인딩
render(myDiffuse, myNormal, myLights);
```

### 루트 파라미터 종류

```cpp
// 1. Root Constants (32비트 상수)
// 가장 빠름, 작은 데이터용
rootParams[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS;
rootParams[0].Constants.Num32BitValues = 4;  // 16바이트
rootParams[0].Constants.ShaderRegister = 0;  // b0
rootParams[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_VERTEX;

// 사용:
commandList->SetGraphicsRoot32BitConstants(0, 4, &data, 0);

// 2. Root Descriptor (직접 GPU 주소)
// 빠름, CBV/SRV/UAV 하나
rootParams[1].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
rootParams[1].Descriptor.ShaderRegister = 1;  // b1

// 사용:
commandList->SetGraphicsRootConstantBufferView(1, cbGpuAddress);

// 3. Descriptor Table (디스크립터 힙 참조)
// 유연함, 여러 리소스 가능
D3D12_DESCRIPTOR_RANGE range = {};
range.RangeType = D3D12_DESCRIPTOR_RANGE_TYPE_SRV;
range.NumDescriptors = 10;  // 텍스처 10개
range.BaseShaderRegister = 0;  // t0~t9

rootParams[2].ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
rootParams[2].DescriptorTable.NumDescriptorRanges = 1;
rootParams[2].DescriptorTable.pDescriptorRanges = &range;

// 사용:
commandList->SetGraphicsRootDescriptorTable(2, gpuHandle);
```

### 루트 시그니처 생성

```cpp
D3D12_ROOT_SIGNATURE_DESC rsDesc = {};
rsDesc.NumParameters = 3;
rsDesc.pParameters = rootParams;
rsDesc.NumStaticSamplers = 1;
rsDesc.pStaticSamplers = &staticSampler;
rsDesc.Flags = D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT;

// 직렬화
ID3DBlob* signatureBlob;
D3D12SerializeRootSignature(&rsDesc, D3D_ROOT_SIGNATURE_VERSION_1,
                            &signatureBlob, nullptr);

// 생성
ID3D12RootSignature* rootSignature;
device->CreateRootSignature(0,
                           signatureBlob->GetBufferPointer(),
                           signatureBlob->GetBufferSize(),
                           IID_PPV_ARGS(&rootSignature));
```

### 셰이더에서 사용

```hlsl
// HLSL
cbuffer PerFrame : register(b0) {    // Root Constant 또는 CBV
    float4 data;
};

cbuffer PerObject : register(b1) {   // Root CBV
    float4x4 worldMatrix;
};

Texture2D textures[10] : register(t0);  // Descriptor Table
SamplerState sampler0 : register(s0);   // Static Sampler
```

### 루트 시그니처 최적화

```
루트 시그니처 공간 제한: 64 DWORD (256 bytes)

각 파라미터 비용:
- Root Constant:      1 DWORD per 32bit
- Root Descriptor:    2 DWORD (64bit GPU 주소)
- Descriptor Table:   1 DWORD

전략:
1. 자주 바뀌는 작은 데이터 → Root Constants
2. 자주 바뀌는 버퍼 하나 → Root Descriptor
3. 많은 텍스처들 → Descriptor Table
4. Static Sampler는 공간 안 차지함
```

```cpp
// 좋은 예: 바인딩 빈도에 따른 배치
struct RootSignatureLayout {
    // Slot 0: 매 드로우마다 바뀜 (가장 빠른 접근)
    // 32bit constants - 물체별 인덱스
    uint objectIndex;  // 1 DWORD

    // Slot 1: 매 드로우마다 바뀜
    // Root CBV - 물체별 변환 행렬
    D3D12_GPU_VIRTUAL_ADDRESS perObjectCB;  // 2 DWORD

    // Slot 2: 재질마다 바뀜
    // Descriptor Table - 재질 텍스처들
    D3D12_GPU_DESCRIPTOR_HANDLE materialTextures;  // 1 DWORD

    // Slot 3: 프레임마다 바뀜
    // Root CBV - 카메라, 조명
    D3D12_GPU_VIRTUAL_ADDRESS perFrameCB;  // 2 DWORD

    // Slot 4: 거의 안 바뀜
    // Descriptor Table - 글로벌 텍스처 (섀도우맵 등)
    D3D12_GPU_DESCRIPTOR_HANDLE globalTextures;  // 1 DWORD
};
// 총 7 DWORD = 28 bytes (64 DWORD 중)
```

---

## 3.5 커맨드 버퍼 / 커맨드 리스트

### 왜 커맨드 버퍼를 쓰나?

```
직접 호출 (OpenGL 스타일):
CPU: "삼각형 그려" → GPU 바로 실행
CPU: "다음 삼각형" → GPU 바로 실행
→ CPU-GPU 왕복이 많음, 느림

커맨드 버퍼 (DX12 스타일):
CPU: "삼각형 그려" → 버퍼에 기록
CPU: "다음 삼각형" → 버퍼에 기록
CPU: "다 됐어, 실행해" → GPU가 버퍼 통째로 실행
→ 왕복 1번, 빠름
```

### 커맨드 리스트 기록 예시

```cpp
// 1. 리스트 시작 (allocator 리셋 후)
commandAllocator->Reset();
commandList->Reset(commandAllocator, pso);

// 2. 렌더 타겟 설정
D3D12_CPU_DESCRIPTOR_HANDLE rtv = rtvHeap->GetCPUDescriptorHandleForHeapStart();
D3D12_CPU_DESCRIPTOR_HANDLE dsv = dsvHeap->GetCPUDescriptorHandleForHeapStart();
commandList->OMSetRenderTargets(1, &rtv, FALSE, &dsv);

// 3. 뷰포트/시저
commandList->RSSetViewports(1, &viewport);
commandList->RSSetScissorRects(1, &scissorRect);

// 4. 리소스 상태 전이
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource = renderTarget;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_PRESENT;
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_RENDER_TARGET;
commandList->ResourceBarrier(1, &barrier);

// 5. 클리어
float clearColor[] = { 0.0f, 0.0f, 0.0f, 1.0f };
commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);
commandList->ClearDepthStencilView(dsv, D3D12_CLEAR_FLAG_DEPTH, 1.0f, 0, 0, nullptr);

// 6. 파이프라인 설정
commandList->SetPipelineState(pso);
commandList->SetGraphicsRootSignature(rootSig);

// 7. 리소스 바인딩
commandList->SetGraphicsRootConstantBufferView(0, cbGpuAddress);
ID3D12DescriptorHeap* heaps[] = { srvHeap };
commandList->SetDescriptorHeaps(1, heaps);
commandList->SetGraphicsRootDescriptorTable(1, srvGpuHandle);

// 8. 버텍스/인덱스 버퍼
commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
commandList->IASetVertexBuffers(0, 1, &vbView);
commandList->IASetIndexBuffer(&ibView);

// 9. 드로우
commandList->DrawIndexedInstanced(indexCount, 1, 0, 0, 0);

// 10. Present 준비
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PRESENT;
commandList->ResourceBarrier(1, &barrier);

// 11. 리스트 종료
commandList->Close();
```

### 멀티스레드 기록

DX12의 큰 장점: 여러 스레드에서 **동시에** 커맨드 기록 가능.

```cpp
// 각 스레드가 자신만의 allocator와 list 사용
struct ThreadContext {
    ID3D12CommandAllocator* allocator;
    ID3D12GraphicsCommandList* commandList;
};

// 메인 스레드
void RenderFrame() {
    // 여러 스레드에서 병렬로 기록
    parallel_for(0, numThreads, [](int threadIdx) {
        auto& ctx = threadContexts[threadIdx];
        ctx.allocator->Reset();
        ctx.commandList->Reset(ctx.allocator, pso);

        // 이 스레드 담당 물체들 기록
        for (auto& obj : objectsForThread[threadIdx]) {
            RecordDrawCommands(ctx.commandList, obj);
        }

        ctx.commandList->Close();
    });

    // 모든 리스트 모아서 한 번에 제출
    std::vector<ID3D12CommandList*> lists;
    for (auto& ctx : threadContexts) {
        lists.push_back(ctx.commandList);
    }
    commandQueue->ExecuteCommandLists(lists.size(), lists.data());
}
```

### 번들 (Bundle)

자주 반복되는 명령 시퀀스를 미리 기록해두고 재사용.

```cpp
// 번들 생성 (한 번만)
ID3D12GraphicsCommandList* bundle;
device->CreateCommandList(0, D3D12_COMMAND_LIST_TYPE_BUNDLE,
                         bundleAllocator, pso, IID_PPV_ARGS(&bundle));

bundle->SetGraphicsRootSignature(rootSig);
bundle->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
bundle->IASetVertexBuffers(0, 1, &vbView);
bundle->IASetIndexBuffer(&ibView);
bundle->DrawIndexedInstanced(indexCount, 1, 0, 0, 0);
bundle->Close();

// 매 프레임 사용
commandList->ExecuteBundle(bundle);  // 빠른 재실행
```

번들 제한:
- Render Target 변경 불가
- 리소스 배리어 불가
- Clear 불가

---

## 3.6 큐와 제출 (Queues and Submission)

### 큐 타입별 역할

```
┌─────────────────────────────────────────────────────────────┐
│                        GPU                                  │
│  ┌─────────────────┐ ┌─────────────────┐ ┌───────────────┐ │
│  │ Graphics Engine │ │ Compute Engine  │ │ Copy Engine   │ │
│  │                 │ │                 │ │               │ │
│  │  렌더링 작업      │ │  컴퓨트 작업     │ │  복사 작업      │ │
│  │  + 컴퓨트        │ │  + 복사         │ │                │ │
│  │  + 복사          │ │                 │ │               │ │
│  └────────┬────────┘ └────────┬────────┘ └───────┬───────┘ │
└───────────│─────────────────────│──────────────────│────────┘
            ↑                     ↑                  ↑
     Direct Queue          Compute Queue        Copy Queue
```

### 멀티 큐 활용

```cpp
// 시나리오: 텍스처 스트리밍 + 렌더링 동시에

// Copy Queue: 백그라운드에서 텍스처 로드
copyQueue->ExecuteCommandLists(1, &copyList);
copyQueue->Signal(copyFence, copyFenceValue);

// Direct Queue: 렌더링 계속 진행
directQueue->ExecuteCommandLists(1, &renderList);

// 나중에 텍스처 로드 완료 확인
if (copyFence->GetCompletedValue() >= copyFenceValue) {
    // 새 텍스처 사용 가능
}
```

### Async Compute

렌더링과 컴퓨트를 병렬로.

```
시간 →
Direct Queue:  [렌더링 A][렌더링 B][렌더링 C]
Compute Queue:           [컴퓨트 1][컴퓨트 2]

GPU 활용률 ↑
```

```cpp
// 프레임 구조 예시
void RenderFrame() {
    // 1. 이전 프레임 컴퓨트 결과 대기 (필요시)
    directQueue->Wait(computeFence, lastComputeFenceValue);

    // 2. 그래픽스 렌더링 시작
    directQueue->ExecuteCommandLists(1, &shadowList);
    directQueue->ExecuteCommandLists(1, &gBufferList);

    // 3. 이 시점에 컴퓨트 작업 시작 (병렬)
    computeQueue->ExecuteCommandLists(1, &computeList);
    computeQueue->Signal(computeFence, ++computeFenceValue);

    // 4. 그래픽스 계속
    directQueue->ExecuteCommandLists(1, &lightingList);

    // 5. 컴퓨트 결과 필요한 시점
    directQueue->Wait(computeFence, computeFenceValue);
    directQueue->ExecuteCommandLists(1, &postProcessList);

    // 6. Present
    swapChain->Present(1, 0);
}
```

---

## 3.7 동기화 (Synchronization)

### 왜 동기화가 필요한가?

CPU와 GPU는 **비동기**로 동작한다.

```
문제 상황:
CPU: 버퍼에 데이터 쓰기
CPU: GPU에게 버퍼 그리라고 명령
CPU: 같은 버퍼에 다른 데이터 쓰기  ← 문제!
GPU: (아직 첫 번째 데이터로 그리는 중)

결과: 데이터 충돌, 렌더링 깨짐
```

### Fence (펜스)

GPU 작업 완료를 CPU에서 확인하는 동기화 객체.

```cpp
// Fence 생성
ID3D12Fence* fence;
UINT64 fenceValue = 0;
device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&fence));

// 이벤트 핸들
HANDLE fenceEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
```

```cpp
// 사용 패턴 1: GPU 작업 완료 대기

// 커맨드 제출 후 펜스 시그널 요청
commandQueue->ExecuteCommandLists(1, &cmdList);
commandQueue->Signal(fence, ++fenceValue);

// GPU 완료 대기
if (fence->GetCompletedValue() < fenceValue) {
    fence->SetEventOnCompletion(fenceValue, fenceEvent);
    WaitForSingleObject(fenceEvent, INFINITE);
}

// 이제 GPU 작업 완료됨
```

```cpp
// 사용 패턴 2: 프레임 동기화 (더블/트리플 버퍼링)

const int FRAME_COUNT = 2;
UINT64 frameFenceValues[FRAME_COUNT] = {};
int currentFrame = 0;

void RenderFrame() {
    // 이 프레임의 이전 사용이 완료될 때까지 대기
    if (fence->GetCompletedValue() < frameFenceValues[currentFrame]) {
        fence->SetEventOnCompletion(frameFenceValues[currentFrame], fenceEvent);
        WaitForSingleObject(fenceEvent, INFINITE);
    }

    // 렌더링...
    commandQueue->ExecuteCommandLists(1, &cmdList);

    // 이 프레임 완료 시 펜스 값 업데이트 요청
    frameFenceValues[currentFrame] = ++fenceValue;
    commandQueue->Signal(fence, fenceValue);

    // Present
    swapChain->Present(1, 0);

    // 다음 프레임
    currentFrame = (currentFrame + 1) % FRAME_COUNT;
}
```

### Semaphore (세마포어)

**GPU-GPU 동기화**를 위한 객체. Fence가 CPU-GPU 동기화라면, Semaphore는 GPU 작업들 간의 순서를 보장한다.

```
┌──────────────────────────────────────────────────────────────┐
│                    동기화 객체 비교                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Fence (펜스)              Semaphore (세마포어)               │
│   ─────────────             ──────────────────               │
│   CPU ←→ GPU 동기화         GPU ←→ GPU 동기화                  │
│                                                              │
│   ┌─────┐    Signal    ┌─────┐                               │
│   │ GPU │────────────▶│ CPU │     (GPU 완료를 CPU가 대기)     │
│   └─────┘              └─────┘                               │
│                                                              │
│   ┌───────┐  Signal   ┌───────┐                              │
│   │Queue A│─────────▶│Queue B│    (큐 간 작업 순서 보장)       │
│   └───────┘  Wait     └───────┘                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### DX12에서의 Semaphore

DX12는 별도의 Semaphore 객체가 없다. 대신 **Fence를 GPU 간 동기화에도 사용**한다.

```cpp
// 큐 간 동기화 (Copy → Graphics)
// Copy Queue에서 작업 완료 후 Signal
copyQueue->ExecuteCommandLists(1, &copyCommandList);
copyQueue->Signal(syncFence, ++syncFenceValue);  // "복사 끝났어"

// Graphics Queue에서 복사 완료 대기 후 렌더링
graphicsQueue->Wait(syncFence, syncFenceValue);  // "복사 끝날 때까지 대기"
graphicsQueue->ExecuteCommandLists(1, &graphicsCommandList);
```

#### Vulkan에서의 Semaphore

Vulkan은 Semaphore를 명시적으로 제공한다. 두 종류가 있다:

```cpp
// 1. Binary Semaphore - 신호/대기 한 번씩
VkSemaphore imageAvailable;   // 스왑체인 이미지 획득 완료
VkSemaphore renderFinished;   // 렌더링 완료

// 이미지 획득 시 세마포어 신호
vkAcquireNextImageKHR(device, swapchain, ..., imageAvailable, ...);

// 제출 시: imageAvailable 대기, renderFinished 신호
VkSubmitInfo submitInfo = {};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = &imageAvailable;     // 이거 기다려
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = &renderFinished;   // 끝나면 이거 신호
vkQueueSubmit(graphicsQueue, 1, &submitInfo, fence);

// Present 시: renderFinished 대기
VkPresentInfoKHR presentInfo = {};
presentInfo.waitSemaphoreCount = 1;
presentInfo.pWaitSemaphores = &renderFinished;    // 렌더링 끝날 때까지 대기
vkQueuePresentKHR(presentQueue, &presentInfo);
```

```cpp
// 2. Timeline Semaphore - 카운터 값으로 동기화 (Vulkan 1.2+)
// DX12의 Fence와 개념적으로 동일!
VkSemaphore timelineSemaphore;
uint64_t timelineValue = 0;

// Signal: 값 증가
VkTimelineSemaphoreSubmitInfo timelineInfo = {};
timelineInfo.signalSemaphoreValueCount = 1;
timelineInfo.pSignalSemaphoreValues = &(++timelineValue);

// Wait: 특정 값 이상이 될 때까지 대기
VkSemaphoreWaitInfo waitInfo = {};
waitInfo.semaphoreCount = 1;
waitInfo.pSemaphores = &timelineSemaphore;
waitInfo.pValues = &timelineValue;
vkWaitSemaphores(device, &waitInfo, UINT64_MAX);
```

#### 동기화 흐름 예시

```
프레임 N 렌더링 파이프라인:

┌─────────────┐   imageAvailable   ┌─────────────┐   renderFinished   ┌─────────────┐
│ 이미지 획득   │──────────────────▶│  렌더링     │───────────────────▶│  Present    │
│ (Swapchain) │    세마포어         │ (Graphics)  │     세마포어        │ (Display)   │
└─────────────┘                    └─────────────┘                    └─────────────┘
      │                                   │                                  │
      │                                   │                                  │
   GPU 작업                            GPU 작업                           GPU 작업
   (매우 빠름)                         (렌더링)                          (디스플레이)

세마포어가 없으면?
→ 아직 이미지 획득 안 됐는데 렌더링 시작
→ 아직 렌더링 안 끝났는데 화면에 표시
→ 찢어진 화면, 깨진 이미지!
```

#### 동기화 객체 정리

| 특성 | DX12 Fence | Vulkan Fence | Vulkan Binary Semaphore | Vulkan Timeline Semaphore |
|------|------------|--------------|------------------------|---------------------------|
| CPU 대기 | ✅ | ✅ | ❌ | ✅ |
| GPU 대기 | ✅ | ❌ | ✅ | ✅ |
| 값 기반 | ✅ (uint64) | ❌ (bool) | ❌ (bool) | ✅ (uint64) |
| 주 용도 | CPU↔GPU, GPU↔GPU | CPU↔GPU | GPU↔GPU | 둘 다 |

### Resource Barrier (리소스 배리어)

리소스 상태 전이와 GPU 내부 동기화.

```cpp
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource = texture;
barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;

commandList->ResourceBarrier(1, &barrier);
```

배리어 타입:
```cpp
// 1. Transition Barrier - 상태 전이
// 읽기/쓰기 용도 변경 시
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;

// 2. UAV Barrier - UAV 동기화
// 같은 UAV에 연속 쓰기 시
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_UAV;
barrier.UAV.pResource = uavResource;

// 3. Aliasing Barrier - 메모리 재사용
// 같은 힙 영역을 다른 리소스가 사용할 때
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_ALIASING;
barrier.Aliasing.pResourceBefore = oldResource;
barrier.Aliasing.pResourceAfter = newResource;
```

### 일반적인 리소스 상태

```cpp
D3D12_RESOURCE_STATE_COMMON                     // 초기/범용
D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER // VB/CB로 읽기
D3D12_RESOURCE_STATE_INDEX_BUFFER               // IB로 읽기
D3D12_RESOURCE_STATE_RENDER_TARGET              // RT로 쓰기
D3D12_RESOURCE_STATE_UNORDERED_ACCESS           // UAV 읽기/쓰기
D3D12_RESOURCE_STATE_DEPTH_WRITE                // 깊이 쓰기
D3D12_RESOURCE_STATE_DEPTH_READ                 // 깊이 읽기
D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE      // PS에서 SRV로 읽기
D3D12_RESOURCE_STATE_COPY_DEST                  // 복사 대상
D3D12_RESOURCE_STATE_COPY_SOURCE                // 복사 소스
D3D12_RESOURCE_STATE_PRESENT                    // Present용
```

### 배리어 배칭

```cpp
// 나쁜 예: 배리어 하나씩
commandList->ResourceBarrier(1, &barrier1);
commandList->ResourceBarrier(1, &barrier2);
commandList->ResourceBarrier(1, &barrier3);

// 좋은 예: 한 번에
D3D12_RESOURCE_BARRIER barriers[] = { barrier1, barrier2, barrier3 };
commandList->ResourceBarrier(3, barriers);
```

### Split Barriers (분할 배리어)

긴 배리어를 분할해서 GPU 유휴 시간 줄이기.

```cpp
// 배리어 시작
barrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_BEGIN_ONLY;
commandList->ResourceBarrier(1, &barrier);

// 다른 작업 수행 (GPU가 배리어 처리하는 동안)
commandList->DrawIndexedInstanced(...);

// 배리어 종료
barrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_END_ONLY;
commandList->ResourceBarrier(1, &barrier);
```

---

## 3.8 스왑체인과 프레젠트 (Swap Chain & Present)

### 스왑체인이란?

화면에 표시할 버퍼들의 집합.

```
┌─────────────────────────────────────────┐
│              Swap Chain                  │
│  ┌───────────┐  ┌───────────┐           │
│  │ Buffer 0  │  │ Buffer 1  │  (Buffer 2) │
│  │ (Front)   │  │ (Back)    │           │
│  └─────┬─────┘  └───────────┘           │
└────────│────────────────────────────────┘
         │
         ▼
      모니터
```

### 스왑체인 생성

```cpp
DXGI_SWAP_CHAIN_DESC1 swapChainDesc = {};
swapChainDesc.Width = 1920;
swapChainDesc.Height = 1080;
swapChainDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
swapChainDesc.SampleDesc.Count = 1;
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
swapChainDesc.BufferCount = 2;  // 더블 버퍼링
swapChainDesc.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
swapChainDesc.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING;

IDXGISwapChain1* swapChain1;
factory->CreateSwapChainForHwnd(
    commandQueue,  // Direct Queue 연결
    hwnd,
    &swapChainDesc,
    nullptr,
    nullptr,
    &swapChain1
);
```

### 버퍼 가져오기

```cpp
ID3D12Resource* renderTargets[2];

for (int i = 0; i < 2; i++) {
    swapChain->GetBuffer(i, IID_PPV_ARGS(&renderTargets[i]));

    // RTV 생성
    D3D12_CPU_DESCRIPTOR_HANDLE rtvHandle = rtvHeap->GetCPUDescriptorHandleForHeapStart();
    rtvHandle.ptr += i * rtvDescriptorSize;
    device->CreateRenderTargetView(renderTargets[i], nullptr, rtvHandle);
}
```

### Present

렌더링 완료 후 화면에 표시.

```cpp
// 렌더링 완료
commandList->ResourceBarrier(1, &presentBarrier);  // RT → PRESENT
commandList->Close();
commandQueue->ExecuteCommandLists(1, &cmdList);

// Present
UINT syncInterval = 1;  // VSync: 0=off, 1=60Hz, 2=30Hz
UINT presentFlags = 0;
swapChain->Present(syncInterval, presentFlags);

// 현재 백버퍼 인덱스 업데이트
currentBackBuffer = swapChain->GetCurrentBackBufferIndex();
```

### Tearing과 VSync

```cpp
// VSync 끄고 Tearing 허용 (최저 지연)
if (tearingSupported) {
    swapChain->Present(0, DXGI_PRESENT_ALLOW_TEARING);
} else {
    swapChain->Present(1, 0);  // VSync
}
```

### HDR 지원

```cpp
// HDR 스왑체인
swapChainDesc.Format = DXGI_FORMAT_R16G16B16A16_FLOAT;  // HDR 포맷

// 또는
swapChainDesc.Format = DXGI_FORMAT_R10G10B10A2_UNORM;   // 10비트

// 컬러 스페이스 설정
swapChain->SetColorSpace1(DXGI_COLOR_SPACE_RGB_FULL_G2084_NONE_P2020);
```

### 리사이즈 처리

```cpp
void OnResize(UINT width, UINT height) {
    // 1. GPU 작업 완료 대기
    WaitForGPU();

    // 2. 기존 참조 해제
    for (int i = 0; i < BUFFER_COUNT; i++) {
        renderTargets[i]->Release();
    }

    // 3. 스왑체인 리사이즈
    swapChain->ResizeBuffers(
        BUFFER_COUNT,
        width, height,
        DXGI_FORMAT_R8G8B8A8_UNORM,
        DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING
    );

    // 4. 새 버퍼 가져오기
    for (int i = 0; i < BUFFER_COUNT; i++) {
        swapChain->GetBuffer(i, IID_PPV_ARGS(&renderTargets[i]));
        // RTV 재생성...
    }

    // 5. 깊이 버퍼도 리사이즈
    RecreateDepthBuffer(width, height);

    currentBackBuffer = swapChain->GetCurrentBackBufferIndex();
}
```

---

## 요약

| 개념 | 역할 |
|------|------|
| Device | GPU 대표, 리소스 생성 |
| Command Queue | GPU에 명령 제출 |
| Command List | 명령 기록 |
| PSO | 렌더링 상태 묶음 |
| Root Signature | 리소스 바인딩 레이아웃 |
| Fence | CPU-GPU 동기화 |
| Barrier | 리소스 상태 전이 |
| Swap Chain | 화면 출력 버퍼 |

---

## 다음 단계

Part 3을 이해했다면 Part 4(리소스 관리)로 넘어가자.
질문 있으면 언제든 물어봐.
