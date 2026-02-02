# Part 4: 리소스 관리

Part 3에서 GPU에게 명령을 보내는 방법을 배웠다.
이제 GPU가 사용할 **데이터(리소스)를 어떻게 관리하는지** 알아보자.

이 부분이 DX12에서 가장 복잡하면서도 성능에 직접적인 영향을 주는 부분이다.

---

## 4.1 버퍼 종류

### 버퍼란?

GPU 메모리에 있는 **연속된 데이터 블록**.

```cpp
// DX12에서 버퍼 생성 (기본 형태)
D3D12_RESOURCE_DESC bufferDesc = {};
bufferDesc.Dimension = D3D12_RESOURCE_DIMENSION_BUFFER;
bufferDesc.Width = sizeInBytes;  // 버퍼 크기
bufferDesc.Height = 1;
bufferDesc.DepthOrArraySize = 1;
bufferDesc.MipLevels = 1;
bufferDesc.Format = DXGI_FORMAT_UNKNOWN;  // 버퍼는 포맷 없음
bufferDesc.SampleDesc.Count = 1;
bufferDesc.Layout = D3D12_TEXTURE_LAYOUT_ROW_MAJOR;

device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    initialState,
    nullptr,
    IID_PPV_ARGS(&buffer)
);
```

### Vertex Buffer (버텍스 버퍼)

버텍스 데이터를 담는 버퍼.

```cpp
// 버텍스 데이터
struct Vertex {
    float position[3];
    float normal[3];
    float texCoord[2];
};

std::vector<Vertex> vertices = { ... };
UINT bufferSize = vertices.size() * sizeof(Vertex);

// 버퍼 생성 (Default Heap에)
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_DEFAULT;

device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,  // 복사 받을 준비
    nullptr,
    IID_PPV_ARGS(&vertexBuffer)
);

// 데이터 업로드 (Upload Heap 경유)
// ... (4.2에서 설명)

// Vertex Buffer View 생성
D3D12_VERTEX_BUFFER_VIEW vbView = {};
vbView.BufferLocation = vertexBuffer->GetGPUVirtualAddress();
vbView.SizeInBytes = bufferSize;
vbView.StrideInBytes = sizeof(Vertex);

// 사용
commandList->IASetVertexBuffers(0, 1, &vbView);
```

### Index Buffer (인덱스 버퍼)

인덱스 데이터를 담는 버퍼.

```cpp
std::vector<uint32_t> indices = { 0, 1, 2, 2, 1, 3, ... };
UINT bufferSize = indices.size() * sizeof(uint32_t);

// 버퍼 생성 (동일한 방식)
// ...

// Index Buffer View
D3D12_INDEX_BUFFER_VIEW ibView = {};
ibView.BufferLocation = indexBuffer->GetGPUVirtualAddress();
ibView.SizeInBytes = bufferSize;
ibView.Format = DXGI_FORMAT_R32_UINT;  // 또는 R16_UINT

// 사용
commandList->IASetIndexBuffer(&ibView);
commandList->DrawIndexedInstanced(indexCount, 1, 0, 0, 0);
```

### Constant Buffer (상수 버퍼)

셰이더에 전달할 상수 데이터.

```cpp
// 상수 버퍼 구조체 (16바이트 정렬 필수!)
struct PerObjectConstants {
    DirectX::XMFLOAT4X4 worldMatrix;       // 64 bytes
    DirectX::XMFLOAT4X4 worldViewProj;     // 64 bytes
    DirectX::XMFLOAT4 color;               // 16 bytes
};  // 총 144 bytes

// 상수 버퍼 크기는 256바이트 정렬
UINT cbSize = (sizeof(PerObjectConstants) + 255) & ~255;

// Upload Heap에 직접 생성 (매 프레임 업데이트하므로)
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_UPLOAD;

// 버퍼 생성
device->CreateCommittedResource(...);

// 영구 맵핑
void* mappedData;
constantBuffer->Map(0, nullptr, &mappedData);
// 프로그램 끝날 때까지 Unmap 안 해도 됨

// 매 프레임 데이터 업데이트
PerObjectConstants cbData = { ... };
memcpy(mappedData, &cbData, sizeof(cbData));

// 셰이더에 바인딩
commandList->SetGraphicsRootConstantBufferView(
    0,  // Root parameter index
    constantBuffer->GetGPUVirtualAddress()
);
```

### 상수 버퍼 정렬 규칙

```hlsl
// HLSL 패킹 규칙
cbuffer PerObject : register(b0) {
    float4x4 worldMatrix;  // 64 bytes, offset 0
    float3 position;       // 12 bytes, offset 64
    // 4 bytes padding (float3 뒤에 float 오면 같은 16바이트에)
    float scale;           // 4 bytes, offset 76
    float3 color;          // 12 bytes, offset 80
    // 4 bytes padding
    float2 tiling;         // 8 bytes, offset 96
    // 8 bytes padding (다음 변수가 16바이트 경계 넘으면)
    float4 params;         // 16 bytes, offset 112
};  // 총 128 bytes
```

```cpp
// C++ 구조체도 맞춰야 함
struct alignas(16) PerObjectConstants {
    XMFLOAT4X4 worldMatrix;  // 64
    XMFLOAT3 position;       // 12
    float scale;             // 4
    XMFLOAT3 color;          // 12
    float _pad1;             // 4 (명시적 패딩)
    XMFLOAT2 tiling;         // 8
    XMFLOAT2 _pad2;          // 8
    XMFLOAT4 params;         // 16
};
```

### Structured Buffer (구조화 버퍼)

구조체 배열을 담는 버퍼. 인스턴싱, 스키닝 등에 사용.

```cpp
// 인스턴스 데이터
struct InstanceData {
    DirectX::XMFLOAT4X4 world;
    DirectX::XMFLOAT4 color;
};

std::vector<InstanceData> instances(1000);
UINT bufferSize = instances.size() * sizeof(InstanceData);

// SRV (Shader Resource View) 생성
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Format = DXGI_FORMAT_UNKNOWN;  // 구조화 버퍼
srvDesc.ViewDimension = D3D12_SRV_DIMENSION_BUFFER;
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Buffer.FirstElement = 0;
srvDesc.Buffer.NumElements = instances.size();
srvDesc.Buffer.StructureByteStride = sizeof(InstanceData);
srvDesc.Buffer.Flags = D3D12_BUFFER_SRV_FLAG_NONE;

device->CreateShaderResourceView(buffer, &srvDesc, srvHandle);
```

```hlsl
// 셰이더에서 사용
struct InstanceData {
    float4x4 world;
    float4 color;
};

StructuredBuffer<InstanceData> instanceBuffer : register(t0);

VSOutput main(VSInput input, uint instanceID : SV_InstanceID) {
    InstanceData inst = instanceBuffer[instanceID];
    float4 worldPos = mul(float4(input.position, 1), inst.world);
    // ...
}
```

### ByteAddress Buffer (바이트 주소 버퍼)

바이트 단위로 접근하는 버퍼. 가장 유연함.

```hlsl
// HLSL
ByteAddressBuffer rawBuffer : register(t0);

uint value = rawBuffer.Load(byteOffset);
uint4 values = rawBuffer.Load4(byteOffset);  // 4개 동시 로드
```

### RWBuffer / RWStructuredBuffer (읽기-쓰기 버퍼)

컴퓨트 셰이더에서 쓰기 가능한 버퍼.

```cpp
// UAV (Unordered Access View) 생성
D3D12_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};
uavDesc.Format = DXGI_FORMAT_UNKNOWN;
uavDesc.ViewDimension = D3D12_UAV_DIMENSION_BUFFER;
uavDesc.Buffer.FirstElement = 0;
uavDesc.Buffer.NumElements = elementCount;
uavDesc.Buffer.StructureByteStride = sizeof(Element);

device->CreateUnorderedAccessView(buffer, nullptr, &uavDesc, uavHandle);
```

```hlsl
// 셰이더
RWStructuredBuffer<Particle> particles : register(u0);

[numthreads(256, 1, 1)]
void CSMain(uint id : SV_DispatchThreadID) {
    // 읽기
    Particle p = particles[id];

    // 수정
    p.position += p.velocity * deltaTime;

    // 쓰기
    particles[id] = p;
}
```

---

## 4.2 힙 종류 (Heap Types)

### 힙이란?

GPU 메모리의 연속된 영역. 리소스가 실제로 저장되는 곳.

```
┌─────────────────────────────────────────────────────────┐
│                      GPU 메모리                         │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Default Heap│  │ Upload Heap │  │ Readback    │     │
│  │             │  │             │  │ Heap        │     │
│  │ GPU 전용    │  │ CPU→GPU     │  │ GPU→CPU     │     │
│  │ 가장 빠름   │  │ 업로드용    │  │ 다운로드용   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Default Heap

GPU가 가장 빠르게 접근할 수 있는 메모리.

```cpp
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_DEFAULT;

// 특징:
// - GPU 읽기/쓰기 빠름
// - CPU 직접 접근 불가
// - 텍스처, 정적 버퍼에 사용
```

**용도**:
- 텍스처
- 정적 메시 (버텍스/인덱스 버퍼)
- 렌더 타겟
- 깊이 버퍼

### Upload Heap

CPU에서 GPU로 데이터 전송용.

```cpp
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_UPLOAD;

// 특징:
// - CPU 쓰기 가능 (Map)
// - GPU 읽기 가능 (느림)
// - Write-Combined 메모리 (쓰기 최적화)
```

**용도**:
- 데이터 업로드 스테이징
- 매 프레임 업데이트되는 상수 버퍼
- 동적 버텍스 버퍼

```cpp
// Upload Heap 버퍼 생성
device->CreateCommittedResource(
    &uploadHeapProps,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    D3D12_RESOURCE_STATE_GENERIC_READ,  // Upload는 항상 이 상태
    nullptr,
    IID_PPV_ARGS(&uploadBuffer)
);

// Map하여 데이터 쓰기
void* mappedData;
D3D12_RANGE readRange = { 0, 0 };  // CPU에서 읽지 않음
uploadBuffer->Map(0, &readRange, &mappedData);
memcpy(mappedData, sourceData, dataSize);
uploadBuffer->Unmap(0, nullptr);
```

### Readback Heap

GPU에서 CPU로 데이터 읽기용.

```cpp
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_READBACK;

// 특징:
// - GPU 쓰기 가능
// - CPU 읽기 가능
// - 느림 (디버깅/특수 목적)
```

**용도**:
- 스크린샷 캡처
- GPU 계산 결과 읽기
- 디버깅

```cpp
// Readback 버퍼 생성
device->CreateCommittedResource(
    &readbackHeapProps,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,
    nullptr,
    IID_PPV_ARGS(&readbackBuffer)
);

// GPU에서 복사
commandList->CopyResource(readbackBuffer, gpuBuffer);

// GPU 작업 완료 후 CPU에서 읽기
WaitForGPU();
void* data;
readbackBuffer->Map(0, nullptr, &data);
// data 사용
readbackBuffer->Unmap(0, nullptr);
```

### 데이터 업로드 전체 과정

```cpp
// Default Heap에 텍스처 업로드 예시

// 1. Default Heap에 최종 텍스처 생성
D3D12_HEAP_PROPERTIES defaultHeap = { D3D12_HEAP_TYPE_DEFAULT };
device->CreateCommittedResource(
    &defaultHeap, ...,
    D3D12_RESOURCE_STATE_COPY_DEST,  // 복사 받을 준비
    ..., &texture
);

// 2. Upload Heap에 스테이징 버퍼 생성
UINT64 uploadSize;
device->GetCopyableFootprints(&textureDesc, 0, 1, 0, nullptr, nullptr, nullptr, &uploadSize);

D3D12_HEAP_PROPERTIES uploadHeap = { D3D12_HEAP_TYPE_UPLOAD };
device->CreateCommittedResource(
    &uploadHeap, ...,
    D3D12_RESOURCE_STATE_GENERIC_READ,
    ..., &uploadBuffer
);

// 3. 스테이징 버퍼에 데이터 쓰기
void* mapped;
uploadBuffer->Map(0, nullptr, &mapped);
memcpy(mapped, imageData, imageSize);
uploadBuffer->Unmap(0, nullptr);

// 4. GPU 복사 명령
D3D12_TEXTURE_COPY_LOCATION src = {};
src.pResource = uploadBuffer;
src.Type = D3D12_TEXTURE_COPY_TYPE_PLACED_FOOTPRINT;
src.PlacedFootprint = footprint;

D3D12_TEXTURE_COPY_LOCATION dst = {};
dst.pResource = texture;
dst.Type = D3D12_TEXTURE_COPY_TYPE_SUBRESOURCE_INDEX;
dst.SubresourceIndex = 0;

commandList->CopyTextureRegion(&dst, 0, 0, 0, &src, nullptr);

// 5. 상태 전이
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource = texture;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_COPY_DEST;
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;
commandList->ResourceBarrier(1, &barrier);

// 6. 커맨드 실행 후 업로드 버퍼 해제 가능
```

### Custom Heap

직접 힙을 생성하고 리소스를 배치.

```cpp
// 힙 생성
D3D12_HEAP_DESC heapDesc = {};
heapDesc.SizeInBytes = 64 * 1024 * 1024;  // 64MB
heapDesc.Properties.Type = D3D12_HEAP_TYPE_DEFAULT;
heapDesc.Flags = D3D12_HEAP_FLAG_ALLOW_ONLY_BUFFERS;

ID3D12Heap* heap;
device->CreateHeap(&heapDesc, IID_PPV_ARGS(&heap));

// 힙 내에 리소스 배치
device->CreatePlacedResource(
    heap,
    offsetInBytes,  // 힙 내 위치
    &resourceDesc,
    initialState,
    nullptr,
    IID_PPV_ARGS(&resource)
);
```

**장점**:
- 메모리 재사용 (Aliasing)
- 메모리 단편화 감소
- 세밀한 제어

---

## 4.3 GPU 메모리 할당

### 할당 전략

```
┌──────────────────────────────────────────────────────────┐
│                    메모리 할당 전략                       │
│                                                          │
│  1. Committed Resource (가장 단순)                       │
│     - 리소스마다 별도 힙 자동 생성                        │
│     - 메모리 낭비 가능                                   │
│                                                          │
│  2. Placed Resource (효율적)                             │
│     - 큰 힙 만들고 그 안에 리소스 배치                    │
│     - 메모리 재사용 가능                                 │
│                                                          │
│  3. Reserved Resource (가상 메모리)                      │
│     - 가상 주소만 예약, 필요 시 물리 메모리 할당          │
│     - 스트리밍에 사용                                    │
└──────────────────────────────────────────────────────────┘
```

### Committed Resource

가장 단순한 방식. 리소스마다 힙이 자동 생성됨.

```cpp
device->CreateCommittedResource(
    &heapProperties,
    D3D12_HEAP_FLAG_NONE,
    &resourceDesc,
    initialState,
    nullptr,
    IID_PPV_ARGS(&resource)
);

// 내부적으로:
// 1. 힙 생성
// 2. 리소스를 힙에 배치
// 모두 자동
```

### Placed Resource

직접 만든 힙에 리소스 배치.

```cpp
// 1. 큰 힙 생성 (한 번)
D3D12_HEAP_DESC heapDesc = {};
heapDesc.SizeInBytes = 256 * 1024 * 1024;  // 256MB
heapDesc.Properties.Type = D3D12_HEAP_TYPE_DEFAULT;
heapDesc.Flags = D3D12_HEAP_FLAG_ALLOW_ALL_BUFFERS_AND_TEXTURES;

device->CreateHeap(&heapDesc, IID_PPV_ARGS(&heap));

// 2. 힙 내에 리소스들 배치
UINT64 offset = 0;

device->CreatePlacedResource(
    heap, offset,
    &texture1Desc, state, nullptr,
    IID_PPV_ARGS(&texture1)
);
offset += AlignUp(GetResourceSize(texture1Desc), 64 * 1024);

device->CreatePlacedResource(
    heap, offset,
    &texture2Desc, state, nullptr,
    IID_PPV_ARGS(&texture2)
);
offset += AlignUp(GetResourceSize(texture2Desc), 64 * 1024);
```

### 메모리 앨리어싱 (Memory Aliasing)

같은 메모리를 다른 리소스가 번갈아 사용.

```cpp
// 같은 힙 위치에 두 리소스 생성
device->CreatePlacedResource(heap, 0, &tempDesc, ..., &tempResource);
device->CreatePlacedResource(heap, 0, &finalDesc, ..., &finalResource);
// 동시에 사용 불가! 하나씩만

// 사용 전환 시 Aliasing Barrier
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_ALIASING;
barrier.Aliasing.pResourceBefore = tempResource;
barrier.Aliasing.pResourceAfter = finalResource;
commandList->ResourceBarrier(1, &barrier);
```

**용도**:
- 렌더 패스 간 임시 버퍼 재사용
- 메모리 절약

### Ring Buffer (링 버퍼)

동적 데이터 업로드에 자주 쓰는 패턴.

```cpp
class UploadRingBuffer {
    ID3D12Resource* buffer;
    UINT8* mappedData;
    UINT64 size;
    UINT64 writeOffset;
    UINT64 fenceValues[FRAME_COUNT];  // 각 프레임의 끝 위치

public:
    UploadRingBuffer(UINT64 size) : size(size), writeOffset(0) {
        // Upload Heap 버퍼 생성
        // 영구 Map
    }

    struct Allocation {
        D3D12_GPU_VIRTUAL_ADDRESS gpuAddress;
        void* cpuAddress;
    };

    Allocation Allocate(UINT64 sizeInBytes, UINT64 alignment) {
        // 정렬
        UINT64 alignedOffset = AlignUp(writeOffset, alignment);

        // 공간 부족하면 처음으로
        if (alignedOffset + sizeInBytes > size) {
            alignedOffset = 0;
        }

        // 이전 프레임이 이 영역 쓰고 있으면 대기
        WaitForFenceIfNeeded(alignedOffset, sizeInBytes);

        Allocation alloc;
        alloc.gpuAddress = buffer->GetGPUVirtualAddress() + alignedOffset;
        alloc.cpuAddress = mappedData + alignedOffset;

        writeOffset = alignedOffset + sizeInBytes;
        return alloc;
    }

    void MarkFrameEnd(UINT64 fenceValue, int frameIndex) {
        fenceValues[frameIndex] = writeOffset;
        // 펜스와 연결
    }
};
```

```
Ring Buffer 동작:
┌──────────────────────────────────────────┐
│ Frame0 │ Frame1 │ Frame2 │ (비어있음)    │
│  사용  │  GPU   │  CPU   │              │
│  완료  │  처리중│  기록중│              │
└──────────────────────────────────────────┘
         ↑        ↑        ↑
      완료 영역  처리중   기록중
      (재사용가)
```

---

## 4.4 디스크립터와 바인딩

### 디스크립터란?

GPU에게 리소스의 위치와 사용 방법을 알려주는 **작은 데이터 구조**.

```
리소스 자체: GPU 메모리의 실제 데이터
디스크립터: "이 리소스는 여기 있고, 이렇게 해석해"

비유:
리소스 = 책
디스크립터 = 도서관 카드 (책 위치, 분류 정보)
```

### 디스크립터 종류

```cpp
// CBV (Constant Buffer View)
// 상수 버퍼를 셰이더에서 읽기
D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc = {};
cbvDesc.BufferLocation = constantBuffer->GetGPUVirtualAddress();
cbvDesc.SizeInBytes = 256;  // 256 정렬

device->CreateConstantBufferView(&cbvDesc, cbvHandle);

// SRV (Shader Resource View)
// 텍스처/버퍼를 셰이더에서 읽기
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Texture2D.MipLevels = -1;  // 모든 밉

device->CreateShaderResourceView(texture, &srvDesc, srvHandle);

// UAV (Unordered Access View)
// 컴퓨트 셰이더에서 읽기/쓰기
D3D12_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};
uavDesc.Format = DXGI_FORMAT_R32_FLOAT;
uavDesc.ViewDimension = D3D12_UAV_DIMENSION_TEXTURE2D;

device->CreateUnorderedAccessView(texture, nullptr, &uavDesc, uavHandle);

// RTV (Render Target View)
// 렌더 타겟으로 쓰기
device->CreateRenderTargetView(renderTarget, nullptr, rtvHandle);

// DSV (Depth Stencil View)
// 깊이/스텐실 버퍼
D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc = {};
dsvDesc.Format = DXGI_FORMAT_D32_FLOAT;
dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2D;

device->CreateDepthStencilView(depthBuffer, &dsvDesc, dsvHandle);

// Sampler
// 텍스처 샘플링 방법
D3D12_SAMPLER_DESC samplerDesc = {};
samplerDesc.Filter = D3D12_FILTER_MIN_MAG_MIP_LINEAR;
samplerDesc.AddressU = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.AddressV = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.AddressW = D3D12_TEXTURE_ADDRESS_MODE_WRAP;

device->CreateSampler(&samplerDesc, samplerHandle);
```

### 디스크립터 힙 (Descriptor Heap)

디스크립터들을 저장하는 배열.

```cpp
// CBV/SRV/UAV 힙 (셰이더에서 사용)
D3D12_DESCRIPTOR_HEAP_DESC heapDesc = {};
heapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
heapDesc.NumDescriptors = 10000;  // 최대 개수
heapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;  // 셰이더 접근 가능!

ID3D12DescriptorHeap* srvHeap;
device->CreateDescriptorHeap(&heapDesc, IID_PPV_ARGS(&srvHeap));

// RTV 힙 (셰이더 접근 불필요)
heapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
heapDesc.NumDescriptors = 100;
heapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;  // Shader Visible 아님

ID3D12DescriptorHeap* rtvHeap;
device->CreateDescriptorHeap(&heapDesc, IID_PPV_ARGS(&rtvHeap));
```

```
디스크립터 힙 종류:
┌─────────────────────────────────────────────────────────┐
│ CBV_SRV_UAV    │ 셰이더에서 사용   │ Shader Visible 가능 │
│ SAMPLER        │ 샘플러           │ Shader Visible 가능 │
│ RTV            │ 렌더 타겟        │ Shader Visible 불가  │
│ DSV            │ 깊이 스텐실      │ Shader Visible 불가  │
└─────────────────────────────────────────────────────────┘

Shader Visible = GPU 셰이더에서 직접 접근 가능
RTV/DSV는 파이프라인 설정에만 사용, 셰이더에서 직접 접근 안 함
```

### 디스크립터 핸들

```cpp
// CPU 핸들: CPU에서 디스크립터 생성/수정에 사용
D3D12_CPU_DESCRIPTOR_HANDLE cpuHandle = heap->GetCPUDescriptorHandleForHeapStart();

// GPU 핸들: GPU에서 디스크립터 접근에 사용
D3D12_GPU_DESCRIPTOR_HANDLE gpuHandle = heap->GetGPUDescriptorHandleForHeapStart();

// 디스크립터 크기 (힙 타입마다 다름)
UINT descriptorSize = device->GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV
);

// N번째 디스크립터 핸들
cpuHandle.ptr += n * descriptorSize;
gpuHandle.ptr += n * descriptorSize;
```

### 바인딩 과정

```cpp
// 1. 디스크립터 힙 설정 (프레임당 한 번)
ID3D12DescriptorHeap* heaps[] = { srvHeap, samplerHeap };
commandList->SetDescriptorHeaps(2, heaps);

// 2. 루트 시그니처 설정
commandList->SetGraphicsRootSignature(rootSig);

// 3. 디스크립터 테이블 바인딩
commandList->SetGraphicsRootDescriptorTable(0, srvGpuHandle);
commandList->SetGraphicsRootDescriptorTable(1, samplerGpuHandle);

// 4. 또는 직접 디스크립터 바인딩 (루트 디스크립터)
commandList->SetGraphicsRootConstantBufferView(2, cbGpuAddress);
```

### Bindless 렌더링

모든 리소스를 하나의 큰 디스크립터 배열에 넣고, 인덱스로 접근.

```cpp
// 거대한 디스크립터 힙
heapDesc.NumDescriptors = 1000000;  // 백만 개

// 모든 텍스처를 힙에 등록
for (int i = 0; i < textureCount; i++) {
    device->CreateShaderResourceView(textures[i], &srvDesc, GetHandle(i));
}
```

```hlsl
// 셰이더에서 인덱스로 접근
Texture2D textures[] : register(t0, space0);  // Unbounded array

float4 main(PSInput input) : SV_TARGET {
    uint textureIndex = input.materialID;  // 머티리얼이 가진 텍스처 인덱스
    return textures[textureIndex].Sample(sampler, input.uv);
}
```

**장점**:
- 디스크립터 테이블 변경 없이 다른 텍스처 사용
- 드로우 콜 간 바인딩 변경 최소화
- GPU Driven Rendering에 필수

---

## 4.5 리소스 상태와 전이

### 리소스 상태란?

리소스가 현재 어떤 용도로 사용되고 있는지.

```
GPU는 리소스 접근을 최적화하기 위해 상태를 알아야 함:
- 읽기 전용? → 캐시 적극 활용
- 쓰기 중? → 다른 읽기 차단
- 압축됨? → 압축 해제 필요

상태가 맞지 않으면 → 데이터 손상, 크래시
```

### 주요 리소스 상태

```cpp
// 초기/범용 상태
D3D12_RESOURCE_STATE_COMMON

// 읽기 전용 상태들
D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER  // VB, CB로 읽기
D3D12_RESOURCE_STATE_INDEX_BUFFER                // IB로 읽기
D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE   // VS, GS 등에서 SRV
D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE       // PS에서 SRV
D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT           // 간접 드로우 인자
D3D12_RESOURCE_STATE_COPY_SOURCE                 // 복사 소스
D3D12_RESOURCE_STATE_DEPTH_READ                  // 깊이 읽기만

// 쓰기 상태들
D3D12_RESOURCE_STATE_RENDER_TARGET               // RT로 쓰기
D3D12_RESOURCE_STATE_UNORDERED_ACCESS            // UAV로 읽기/쓰기
D3D12_RESOURCE_STATE_DEPTH_WRITE                 // 깊이 쓰기
D3D12_RESOURCE_STATE_COPY_DEST                   // 복사 대상
D3D12_RESOURCE_STATE_STREAM_OUT                  // 스트림 출력

// 특수 상태
D3D12_RESOURCE_STATE_PRESENT                     // 화면 표시
D3D12_RESOURCE_STATE_GENERIC_READ                // Upload 힙 전용

// 조합 가능
D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE |
D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE   // 모든 셰이더에서 읽기
```

### 상태 전이 (Transition)

```cpp
// 상태 전이 배리어
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_NONE;
barrier.Transition.pResource = texture;
barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;

commandList->ResourceBarrier(1, &barrier);
```

### 왜 배리어가 필요한가?

```
시나리오: 렌더 타겟 → 텍스처로 읽기

Pass 1: 씬을 RT에 렌더링
        GPU가 RT에 쓰기 중...
        (쓰기 캐시에 데이터 있을 수 있음)

[배리어 없이 바로 읽기 시도]
        GPU가 RT를 텍스처로 읽기
        → 캐시 플러시 안 됨
        → 쓰기 완료 안 기다림
        → 잘못된 데이터 읽음!

[배리어 삽입]
        GPU: "쓰기 다 끝내고 캐시 플러시해"
        GPU: "이제 읽기 시작해도 됨"
        → 올바른 데이터
```

### 일반적인 전이 패턴

```cpp
// 렌더 타겟으로 그리기 → 셰이더에서 읽기
RT -> SRV

// 셰이더에서 읽기 → 다시 그리기
SRV -> RT

// 렌더링 완료 → Present
RT -> PRESENT

// Present 후 → 다시 렌더링
PRESENT -> RT

// 업로드 → 사용
COPY_DEST -> SRV (또는 VB 등)

// UAV 컴퓨트 → 읽기
UAV -> SRV

// UAV 연속 쓰기 (UAV Barrier 필요)
UAV -> UAV
```

### 리소스 상태 추적

엔진에서 상태를 추적해야 올바른 배리어 삽입 가능.

```cpp
class ResourceStateTracker {
    struct ResourceState {
        D3D12_RESOURCE_STATES currentState;
        D3D12_RESOURCE_STATES pendingState;  // 배리어 대기 중
    };

    std::unordered_map<ID3D12Resource*, ResourceState> states;

public:
    void TransitionResource(ID3D12Resource* resource,
                           D3D12_RESOURCE_STATES newState,
                           std::vector<D3D12_RESOURCE_BARRIER>& barriers) {
        auto& state = states[resource];

        if (state.currentState != newState) {
            D3D12_RESOURCE_BARRIER barrier = {};
            barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
            barrier.Transition.pResource = resource;
            barrier.Transition.StateBefore = state.currentState;
            barrier.Transition.StateAfter = newState;
            barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;

            barriers.push_back(barrier);
            state.currentState = newState;
        }
    }

    void FlushBarriers(ID3D12GraphicsCommandList* cmdList) {
        if (!pendingBarriers.empty()) {
            cmdList->ResourceBarrier(pendingBarriers.size(), pendingBarriers.data());
            pendingBarriers.clear();
        }
    }
};
```

### Enhanced Barrier (DX12 Ultimate)

더 세밀한 동기화 제어.

```cpp
// 레거시 배리어
D3D12_RESOURCE_BARRIER barrier;
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;

// Enhanced 배리어 (DX12 Agility SDK)
D3D12_BARRIER_GROUP barrierGroup = {};
D3D12_TEXTURE_BARRIER textureBarrier = {};
textureBarrier.SyncBefore = D3D12_BARRIER_SYNC_RENDER_TARGET;
textureBarrier.SyncAfter = D3D12_BARRIER_SYNC_PIXEL_SHADING;
textureBarrier.AccessBefore = D3D12_BARRIER_ACCESS_RENDER_TARGET;
textureBarrier.AccessAfter = D3D12_BARRIER_ACCESS_SHADER_RESOURCE;
textureBarrier.LayoutBefore = D3D12_BARRIER_LAYOUT_RENDER_TARGET;
textureBarrier.LayoutAfter = D3D12_BARRIER_LAYOUT_SHADER_RESOURCE;
textureBarrier.pResource = texture;

// 더 정확한 동기화 지점 지정 가능
// → 불필요한 대기 줄임
```

---

## 4.6 텍스처

### 텍스처 생성

```cpp
D3D12_RESOURCE_DESC textureDesc = {};
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
textureDesc.Width = 1024;
textureDesc.Height = 1024;
textureDesc.DepthOrArraySize = 1;
textureDesc.MipLevels = 0;  // 0 = 자동으로 모든 밉 레벨
textureDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
textureDesc.SampleDesc.Count = 1;
textureDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;  // GPU 최적화 레이아웃
textureDesc.Flags = D3D12_RESOURCE_FLAG_NONE;

D3D12_HEAP_PROPERTIES heapProps = { D3D12_HEAP_TYPE_DEFAULT };

device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &textureDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,
    nullptr,
    IID_PPV_ARGS(&texture)
);
```

### 텍스처 타입

```cpp
// 1D 텍스처 (그라디언트 룩업 등)
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE1D;
textureDesc.Height = 1;

// 2D 텍스처 (일반)
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;

// 3D 텍스처 (볼륨, 노이즈)
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE3D;
textureDesc.DepthOrArraySize = 64;  // 깊이

// 2D 텍스처 배열 (스프라이트 시트)
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
textureDesc.DepthOrArraySize = 10;  // 배열 크기

// 큐브맵 (환경맵)
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
textureDesc.DepthOrArraySize = 6;  // 6면
// SRV에서 TEXTURECUBE로 지정

// 큐브맵 배열
textureDesc.DepthOrArraySize = 6 * numCubemaps;
```

### 밉맵

```cpp
// 밉맵 레벨 수 계산
UINT CalculateMipLevels(UINT width, UINT height) {
    UINT levels = 1;
    while (width > 1 || height > 1) {
        width = max(1, width / 2);
        height = max(1, height / 2);
        levels++;
    }
    return levels;
}

// 각 밉 레벨 업로드
for (UINT mip = 0; mip < mipLevels; mip++) {
    D3D12_TEXTURE_COPY_LOCATION src = {};
    src.pResource = uploadBuffer;
    src.Type = D3D12_TEXTURE_COPY_TYPE_PLACED_FOOTPRINT;
    src.PlacedFootprint = footprints[mip];

    D3D12_TEXTURE_COPY_LOCATION dst = {};
    dst.pResource = texture;
    dst.Type = D3D12_TEXTURE_COPY_TYPE_SUBRESOURCE_INDEX;
    dst.SubresourceIndex = mip;

    commandList->CopyTextureRegion(&dst, 0, 0, 0, &src, nullptr);
}
```

### 압축 포맷

```cpp
// BC1 (DXT1) - 6:1 압축, 1비트 알파
DXGI_FORMAT_BC1_UNORM

// BC3 (DXT5) - 4:1 압축, 부드러운 알파
DXGI_FORMAT_BC3_UNORM

// BC4 - 단일 채널 (높이맵, AO)
DXGI_FORMAT_BC4_UNORM

// BC5 - 2채널 (노말맵의 XY)
DXGI_FORMAT_BC5_UNORM

// BC6H - HDR 압축
DXGI_FORMAT_BC6H_UF16

// BC7 - 고품질 압축 (가장 좋음)
DXGI_FORMAT_BC7_UNORM
```

### 렌더 타겟 텍스처

```cpp
// 렌더 타겟으로 사용할 텍스처
D3D12_RESOURCE_DESC rtDesc = {};
rtDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
rtDesc.Width = 1920;
rtDesc.Height = 1080;
rtDesc.Format = DXGI_FORMAT_R16G16B16A16_FLOAT;  // HDR
rtDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET;  // 핵심!

D3D12_CLEAR_VALUE clearValue = {};
clearValue.Format = rtDesc.Format;
clearValue.Color[0] = 0.0f;  // 클리어 색상 최적화 힌트

device->CreateCommittedResource(
    &defaultHeapProps,
    D3D12_HEAP_FLAG_NONE,
    &rtDesc,
    D3D12_RESOURCE_STATE_RENDER_TARGET,
    &clearValue,  // 클리어 값 힌트
    IID_PPV_ARGS(&renderTargetTexture)
);

// RTV 생성
device->CreateRenderTargetView(renderTargetTexture, nullptr, rtvHandle);

// SRV도 생성 (나중에 읽기 위해)
device->CreateShaderResourceView(renderTargetTexture, nullptr, srvHandle);
```

### 깊이 버퍼 텍스처

```cpp
D3D12_RESOURCE_DESC depthDesc = {};
depthDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
depthDesc.Width = 1920;
depthDesc.Height = 1080;
depthDesc.Format = DXGI_FORMAT_R32_TYPELESS;  // Typeless로 생성
depthDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;

D3D12_CLEAR_VALUE depthClear = {};
depthClear.Format = DXGI_FORMAT_D32_FLOAT;
depthClear.DepthStencil.Depth = 1.0f;

device->CreateCommittedResource(..., &depthBuffer);

// DSV (깊이 쓰기용)
D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc = {};
dsvDesc.Format = DXGI_FORMAT_D32_FLOAT;
dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2D;
device->CreateDepthStencilView(depthBuffer, &dsvDesc, dsvHandle);

// SRV (셰이더에서 읽기용)
D3D12_SHADER_RESOURCE_VIEW_DESC depthSrvDesc = {};
depthSrvDesc.Format = DXGI_FORMAT_R32_FLOAT;  // 읽기용 포맷
depthSrvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
depthSrvDesc.Texture2D.MipLevels = 1;
device->CreateShaderResourceView(depthBuffer, &depthSrvDesc, depthSrvHandle);
```

---

## 요약

| 개념 | 핵심 |
|------|------|
| 버퍼 종류 | Vertex, Index, Constant, Structured, RW |
| 힙 종류 | Default(GPU), Upload(CPU→GPU), Readback(GPU→CPU) |
| 메모리 할당 | Committed(자동), Placed(수동), Aliasing(재사용) |
| 디스크립터 | 리소스 접근 방법 정의 (CBV, SRV, UAV, RTV, DSV) |
| 디스크립터 힙 | 디스크립터 저장소, Shader Visible 여부 |
| 리소스 상태 | 현재 용도, 배리어로 전이 필요 |
| 텍스처 | 1D/2D/3D/Cube, Mipmap, 압축 포맷 |

---

## 전체 로드맵 완료!

이제 Part 0~4를 다 익혔다면:
1. renderer.cpp의 구조가 보이기 시작할 것
2. "왜 이렇게 복잡하게 하지?"가 "아, 이래서 이렇게 하는구나"로 바뀔 것
3. Wicked Engine 커밋 분석도 훨씬 수월해질 것

질문 있으면 언제든 물어봐!
