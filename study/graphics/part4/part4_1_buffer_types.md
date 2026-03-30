# Part 4-1: Buffer Types and GPU Alignment

> Index: [Part 4 Index](README.md)
> Next: [Part 4-2](part4_2_heap_memory.md) — Heap Types and Memory Allocation

---

## What Is a Buffer?

A buffer is a **contiguous block of GPU memory**. It has no format — just raw bytes. You describe how to interpret those bytes when you create a view (SRV, UAV, etc.).

```cpp
D3D12_RESOURCE_DESC bufferDesc = {};
bufferDesc.Dimension = D3D12_RESOURCE_DIMENSION_BUFFER;
bufferDesc.Width     = sizeInBytes;   // total byte count
bufferDesc.Height    = 1;
bufferDesc.DepthOrArraySize = 1;
bufferDesc.MipLevels = 1;
bufferDesc.Format    = DXGI_FORMAT_UNKNOWN;  // buffers have no format
bufferDesc.SampleDesc.Count = 1;
bufferDesc.Layout    = D3D12_TEXTURE_LAYOUT_ROW_MAJOR;
```

---

## Buffer Types

### Vertex Buffer (VB)

Holds per-vertex data. Bound via `IASetVertexBuffers()`.

```cpp
struct Vertex {
    float position[3];
    float normal[3];
    float texCoord[2];
};

D3D12_VERTEX_BUFFER_VIEW vbView = {};
vbView.BufferLocation = vertexBuffer->GetGPUVirtualAddress();
vbView.SizeInBytes    = vertexCount * sizeof(Vertex);
vbView.StrideInBytes  = sizeof(Vertex);

commandList->IASetVertexBuffers(0, 1, &vbView);
```

### Index Buffer (IB)

Holds indices into the vertex buffer. Reduces vertex duplication.

```cpp
D3D12_INDEX_BUFFER_VIEW ibView = {};
ibView.BufferLocation = indexBuffer->GetGPUVirtualAddress();
ibView.SizeInBytes    = indexCount * sizeof(uint32_t);
ibView.Format         = DXGI_FORMAT_R32_UINT;  // or R16_UINT

commandList->IASetIndexBuffer(&ibView);
commandList->DrawIndexedInstanced(indexCount, 1, 0, 0, 0);
```

### Constant Buffer (CB)

Holds small, frequently-updated shader constants (transforms, material params, etc.).

- Must be placed in an **Upload Heap** (CPU writes every frame)
- Size must be **256-byte aligned** (hardware requirement)
- Permanently mapped — no need to call `Unmap` between frames

```cpp
// 256-byte align
UINT cbSize = (sizeof(PerObjectConstants) + 255) & ~255;

// Create in Upload Heap, permanently mapped
void* mappedData;
constantBuffer->Map(0, nullptr, &mappedData);

// Each frame: write new data
PerObjectConstants cbData = { ... };
memcpy(mappedData, &cbData, sizeof(cbData));

// Bind
commandList->SetGraphicsRootConstantBufferView(
    0,
    constantBuffer->GetGPUVirtualAddress()
);
```

**HLSL/C++ alignment rules:**

HLSL packs struct members into 16-byte rows. A member that would straddle a 16-byte boundary is moved to the next row. Always match your C++ struct layout to the HLSL layout.

```cpp
// C++ struct (must match HLSL cbuffer layout)
struct alignas(16) PerObjectConstants {
    XMFLOAT4X4 worldMatrix;  // 64 bytes
    XMFLOAT3   position;     // 12 bytes
    float      scale;        // 4 bytes  ← fits in same row
    XMFLOAT3   color;        // 12 bytes
    float      _pad1;        // 4 bytes  ← explicit padding
    XMFLOAT2   tiling;       // 8 bytes
    XMFLOAT2   _pad2;        // 8 bytes
    XMFLOAT4   params;       // 16 bytes
};
```

### Structured Buffer

An array of structs. Read-only in shaders as an SRV; read-write as a UAV. Used for instancing, skinning, particle data.

```cpp
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Format                         = DXGI_FORMAT_UNKNOWN;
srvDesc.ViewDimension                  = D3D12_SRV_DIMENSION_BUFFER;
srvDesc.Shader4ComponentMapping        = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Buffer.NumElements             = instanceCount;
srvDesc.Buffer.StructureByteStride     = sizeof(InstanceData);

device->CreateShaderResourceView(buffer, &srvDesc, srvHandle);
```

```hlsl
// HLSL
StructuredBuffer<InstanceData> instanceBuffer : register(t0);

VSOutput main(VSInput input, uint id : SV_InstanceID) {
    InstanceData inst = instanceBuffer[id];
    // ...
}
```

### ByteAddress Buffer

Raw bytes accessed by byte offset. The most flexible buffer type — no stride, no element type.

```hlsl
ByteAddressBuffer rawBuffer : register(t0);

uint  value  = rawBuffer.Load(byteOffset);
uint4 values = rawBuffer.Load4(byteOffset);  // load 4 uints at once
```

### RWBuffer / RWStructuredBuffer (UAV)

Read-write buffers for compute shaders.

```cpp
D3D12_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};
uavDesc.Format                       = DXGI_FORMAT_UNKNOWN;
uavDesc.ViewDimension                = D3D12_UAV_DIMENSION_BUFFER;
uavDesc.Buffer.NumElements           = elementCount;
uavDesc.Buffer.StructureByteStride   = sizeof(Element);

device->CreateUnorderedAccessView(buffer, nullptr, &uavDesc, uavHandle);
```

```hlsl
RWStructuredBuffer<Particle> particles : register(u0);

[numthreads(256, 1, 1)]
void CSMain(uint id : SV_DispatchThreadID) {
    Particle p = particles[id];
    p.position += p.velocity * deltaTime;
    particles[id] = p;
}
```

---

## GPU Buffer Alignment

### SIZE vs ALIGNMENT

These are independent concepts:

- **SIZE** — how many bytes the data actually occupies
- **ALIGNMENT** — the VRAM address where the buffer must start must be a multiple of this value

A 100-byte constant buffer still requires a **256-byte aligned** start address. The "wasted" bytes between 100 and 256 are padding.

### DX12 Alignment Requirements

| Buffer / Resource Type | Required Alignment |
|------------------------|--------------------|
| General buffer (VB, IB, Structured, etc.) | 4 bytes |
| Constant Buffer (CBV) | **256 bytes** (`D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT`) |
| Placed resource (textures, etc.) | **4 KB** (`D3D12_DEFAULT_RESOURCE_PLACEMENT_ALIGNMENT`) |
| Small texture (< 64 KB) | **4 KB** (`D3D12_SMALL_RESOURCE_PLACEMENT_ALIGNMENT`) |
| Normal texture (Placed) | **64 KB** (`D3D12_DEFAULT_RESOURCE_PLACEMENT_ALIGNMENT`) |
| MSAA texture | **4 MB** (`D3D12_DEFAULT_MSAA_RESOURCE_PLACEMENT_ALIGNMENT`) |

Maximum alignment value is 4 MB, so a `uint32_t` field is sufficient to store it (uint32_t max ≈ 4 GB).

### `GPUBufferDesc::alignment` Field

```cpp
// GBackend.h
struct GPUBufferDesc {
    uint64_t size      = 0;  // buffer data size in bytes
    uint32_t alignment = 0;  // required VRAM start-address alignment (0 = default 4 bytes)
    // ...
};
```

The `alignment` field stores the result of `GetMinOffsetAlignment()` (a DX12 driver query). If 0, the driver default of 4 bytes is used.

```
CreateBuffer(desc)
  → query driver: GetMinOffsetAlignment()
  → store result → desc.alignment
  → CreatePlacedResource(..., alignment, ...)
```

### AlignUp Utility

Rounds a value up to the next multiple of `alignment`:

```cpp
constexpr uint64_t AlignUp(uint64_t value, uint64_t alignment) {
    return (value + alignment - 1) & ~(alignment - 1);
}

// Examples:
AlignUp(100, 256) = 256
AlignUp(257, 256) = 512
AlignUp(256, 256) = 256  // already aligned
```
