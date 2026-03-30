# Part 4-3: Descriptors and Binding

> Index: [Part 4 Index](README.md)
> Previous: [Part 4-2](part4_2_heap_memory.md) — Heap Types and Memory Allocation
> Next: [Part 4-4](part4_4_resource_states.md) — Resource States and Barriers

---

## What Is a Descriptor?

A descriptor is a **small data structure** that tells the GPU where a resource is and how to interpret it.

```
Resource itself:  actual data in GPU memory
Descriptor:       "this resource is at address X, interpret it as a 2D texture with format Y"

Analogy:
Resource   = a book
Descriptor = a library card (location + classification info)
```

The GPU never receives a raw pointer to a resource — it always goes through a descriptor. This allows the GPU to validate and cache resource access efficiently.

---

## Descriptor Types

```cpp
// CBV — Constant Buffer View: read-only shader constants
D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc = {};
cbvDesc.BufferLocation = constantBuffer->GetGPUVirtualAddress();
cbvDesc.SizeInBytes    = 256;  // must be 256-byte aligned
device->CreateConstantBufferView(&cbvDesc, cbvHandle);

// SRV — Shader Resource View: read a texture or buffer in a shader
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Format                  = DXGI_FORMAT_R8G8B8A8_UNORM;
srvDesc.ViewDimension           = D3D12_SRV_DIMENSION_TEXTURE2D;
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Texture2D.MipLevels     = -1;  // all mips
device->CreateShaderResourceView(texture, &srvDesc, srvHandle);

// UAV — Unordered Access View: read/write in compute shaders
D3D12_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};
uavDesc.Format        = DXGI_FORMAT_R32_FLOAT;
uavDesc.ViewDimension = D3D12_UAV_DIMENSION_TEXTURE2D;
device->CreateUnorderedAccessView(texture, nullptr, &uavDesc, uavHandle);

// RTV — Render Target View: write as a render target
device->CreateRenderTargetView(renderTarget, nullptr, rtvHandle);

// DSV — Depth Stencil View: depth/stencil buffer
D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc = {};
dsvDesc.Format        = DXGI_FORMAT_D32_FLOAT;
dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2D;
device->CreateDepthStencilView(depthBuffer, &dsvDesc, dsvHandle);

// Sampler — how to sample a texture (filter mode, address mode)
D3D12_SAMPLER_DESC samplerDesc = {};
samplerDesc.Filter   = D3D12_FILTER_MIN_MAG_MIP_LINEAR;
samplerDesc.AddressU = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.AddressV = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.AddressW = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
device->CreateSampler(&samplerDesc, samplerHandle);
```

---

## Descriptor Heaps

Descriptors are stored in **descriptor heaps** — arrays of descriptors in GPU memory.

```cpp
// CBV/SRV/UAV heap — for shader-accessible descriptors
D3D12_DESCRIPTOR_HEAP_DESC heapDesc = {};
heapDesc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
heapDesc.NumDescriptors = 10000;
heapDesc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;  // GPU can access directly

ID3D12DescriptorHeap* srvHeap;
device->CreateDescriptorHeap(&heapDesc, IID_PPV_ARGS(&srvHeap));

// RTV heap — render target descriptors (not shader-visible)
heapDesc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
heapDesc.NumDescriptors = 100;
heapDesc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;

ID3D12DescriptorHeap* rtvHeap;
device->CreateDescriptorHeap(&heapDesc, IID_PPV_ARGS(&rtvHeap));
```

| Heap Type | Used For | Shader Visible |
|-----------|----------|----------------|
| `CBV_SRV_UAV` | CBV, SRV, UAV | Yes |
| `SAMPLER` | Samplers | Yes |
| `RTV` | Render targets | No |
| `DSV` | Depth/stencil | No |

RTV and DSV are only used for pipeline setup commands (`OMSetRenderTargets`), not shader register access — so they do not need to be shader-visible.

---

## Descriptor Handles

```cpp
// CPU handle — used by the CPU to create/copy descriptors
D3D12_CPU_DESCRIPTOR_HANDLE cpuHandle = heap->GetCPUDescriptorHandleForHeapStart();

// GPU handle — used by the GPU to read descriptors
D3D12_GPU_DESCRIPTOR_HANDLE gpuHandle = heap->GetGPUDescriptorHandleForHeapStart();

// Descriptor size varies by heap type (hardware-dependent)
UINT descriptorSize = device->GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV
);

// Get handle for the Nth descriptor
cpuHandle.ptr += N * descriptorSize;
gpuHandle.ptr += N * descriptorSize;
```

---

## Binding Descriptors to Shaders

```cpp
// Step 1: Set the descriptor heaps (once per frame)
ID3D12DescriptorHeap* heaps[] = { srvHeap, samplerHeap };
commandList->SetDescriptorHeaps(2, heaps);

// Step 2: Set root signature
commandList->SetGraphicsRootSignature(rootSig);

// Step 3a: Descriptor Table — GPU reads a range of descriptors from the heap
commandList->SetGraphicsRootDescriptorTable(0, srvGpuHandle);
commandList->SetGraphicsRootDescriptorTable(1, samplerGpuHandle);

// Step 3b: Root Descriptor — pass GPU address directly, no descriptor heap
commandList->SetGraphicsRootConstantBufferView(2, cbGpuAddress);
```

For the difference between Descriptor Table and Root Descriptor binding, see [Part 3-3: Root Signature](../part3/part3_3_root_signature.md).

---

## Bindless Rendering

Instead of binding different descriptor tables for each draw call, register all resources in one giant heap and access them by index in the shader.

```cpp
// One huge heap holding all textures
heapDesc.NumDescriptors = 1000000;

// Register every texture at creation time
for (int i = 0; i < textureCount; i++) {
    device->CreateShaderResourceView(textures[i], &srvDesc, GetHandle(i));
}
```

```hlsl
// Shader: access any texture by index
Texture2D textures[] : register(t0, space0);  // unbounded array

float4 main(PSInput input) : SV_TARGET {
    uint texIdx = input.materialID;
    return textures[texIdx].Sample(sampler, input.uv);
}
```

**Advantages:**
- No descriptor table changes between draw calls
- Minimal binding overhead per draw call
- Required for GPU-Driven Rendering (where the GPU generates draw arguments)

The engine stores the heap index for each resource in a `SingleDescriptor::index` field. See [Part 6-5: Buffer Subresource and SingleDescriptor](../part6/part6_5_buffer_subresource.md) for how this integrates with the engine's binding system.
