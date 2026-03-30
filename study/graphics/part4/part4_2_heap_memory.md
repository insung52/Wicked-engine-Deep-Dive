# Part 4-2: Heap Types and Memory Allocation

> Index: [Part 4 Index](README.md)
> Previous: [Part 4-1](part4_1_buffer_types.md) — Buffer Types
> Next: [Part 4-3](part4_3_descriptors.md) — Descriptors and Binding

---

## What Is a Heap?

A heap is a **contiguous region of GPU memory**. Resources (buffers, textures) live inside heaps. The heap type determines who can read and write it.

```
┌─────────────────────────────────────────────────────────┐
│                      GPU Memory                         │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Default     │  │ Upload      │  │ Readback    │     │
│  │ Heap        │  │ Heap        │  │ Heap        │     │
│  │ GPU only    │  │ CPU → GPU   │  │ GPU → CPU   │     │
│  │ Fastest     │  │ Upload use  │  │ Download    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

---

## Heap Types

### Default Heap

GPU-only memory. The fastest for GPU access. CPU cannot directly read or write.

```cpp
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_DEFAULT;
```

**Use for:** textures, static meshes (VB/IB), render targets, depth buffers.

### Upload Heap

CPU-writable, GPU-readable. Slower for the GPU than Default Heap. Write-Combined memory (optimized for sequential CPU writes).

```cpp
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_UPLOAD;

// Create buffer, then map permanently
void* mappedData;
D3D12_RANGE readRange = { 0, 0 };  // CPU will not read
uploadBuffer->Map(0, &readRange, &mappedData);
memcpy(mappedData, sourceData, dataSize);
uploadBuffer->Unmap(0, nullptr);
```

**Use for:** staging uploads to Default Heap, per-frame constant buffers, dynamic vertex data.

### Readback Heap

GPU-writable, CPU-readable. Used to pull GPU computation results back to the CPU.

```cpp
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_READBACK;

// GPU copies into it, then CPU reads after GPU finishes
commandList->CopyResource(readbackBuffer, gpuBuffer);
// ... wait for GPU ...
void* data;
readbackBuffer->Map(0, nullptr, &data);
// read data
readbackBuffer->Unmap(0, nullptr);
```

**Use for:** screenshots, GPU computation results, debugging.

### Custom / Explicit Heap

Create a heap manually, then place resources inside it at specific offsets.

```cpp
D3D12_HEAP_DESC heapDesc = {};
heapDesc.SizeInBytes                = 64 * 1024 * 1024;  // 64 MB
heapDesc.Properties.Type            = D3D12_HEAP_TYPE_DEFAULT;
heapDesc.Flags                      = D3D12_HEAP_FLAG_ALLOW_ONLY_BUFFERS;

ID3D12Heap* heap;
device->CreateHeap(&heapDesc, IID_PPV_ARGS(&heap));

// Place resource at a specific offset within the heap
device->CreatePlacedResource(
    heap,
    offsetInBytes,
    &resourceDesc,
    initialState,
    nullptr,
    IID_PPV_ARGS(&resource)
);
```

---

## Full Upload Process (CPU → GPU)

To get data into Default Heap (GPU-only), you must go through an Upload Heap staging buffer:

```cpp
// Step 1: Create destination resource in Default Heap
D3D12_HEAP_PROPERTIES defaultHeap = { D3D12_HEAP_TYPE_DEFAULT };
device->CreateCommittedResource(
    &defaultHeap, D3D12_HEAP_FLAG_NONE,
    &textureDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,   // ready to receive a copy
    nullptr, IID_PPV_ARGS(&texture)
);

// Step 2: Create Upload Heap staging buffer
UINT64 uploadSize;
device->GetCopyableFootprints(&textureDesc, 0, 1, 0, nullptr, nullptr, nullptr, &uploadSize);

D3D12_HEAP_PROPERTIES uploadHeap = { D3D12_HEAP_TYPE_UPLOAD };
device->CreateCommittedResource(
    &uploadHeap, D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(uploadSize),
    D3D12_RESOURCE_STATE_GENERIC_READ,
    nullptr, IID_PPV_ARGS(&uploadBuffer)
);

// Step 3: Write data into staging buffer
void* mapped;
uploadBuffer->Map(0, nullptr, &mapped);
memcpy(mapped, imageData, imageSize);
uploadBuffer->Unmap(0, nullptr);

// Step 4: Issue GPU copy command
commandList->CopyTextureRegion(&dst, 0, 0, 0, &src, nullptr);

// Step 5: Transition to shader-readable state
// COPY_DEST → PIXEL_SHADER_RESOURCE
commandList->ResourceBarrier(1, &barrier);

// Step 6: After GPU executes, the upload buffer can be released
```

---

## Memory Allocation Strategies

### Committed Resource

Simplest approach. DX12 automatically creates a dedicated heap for each resource.

```cpp
device->CreateCommittedResource(
    &heapProperties,
    D3D12_HEAP_FLAG_NONE,
    &resourceDesc,
    initialState,
    nullptr,
    IID_PPV_ARGS(&resource)
);
// Internally: creates heap + places resource — fully automatic
```

**Drawback:** one heap per resource → many small allocations → memory fragmentation.

### Placed Resource

Create one large heap, then place multiple resources inside it at manual offsets. More efficient.

```cpp
// Create one large heap once
D3D12_HEAP_DESC heapDesc = {};
heapDesc.SizeInBytes = 256 * 1024 * 1024;  // 256 MB
heapDesc.Properties.Type = D3D12_HEAP_TYPE_DEFAULT;
heapDesc.Flags = D3D12_HEAP_FLAG_ALLOW_ALL_BUFFERS_AND_TEXTURES;
device->CreateHeap(&heapDesc, IID_PPV_ARGS(&heap));

// Place resources at sequential offsets
UINT64 offset = 0;

device->CreatePlacedResource(heap, offset, &tex1Desc, state, nullptr, IID_PPV_ARGS(&tex1));
offset += AlignUp(GetResourceSize(tex1Desc), 64 * 1024);

device->CreatePlacedResource(heap, offset, &tex2Desc, state, nullptr, IID_PPV_ARGS(&tex2));
```

### Memory Aliasing

Two different resources share the **same memory region**. Only one can be in use at a time.

```cpp
// Both placed at offset 0 — they alias
device->CreatePlacedResource(heap, 0, &tempDesc,  ..., IID_PPV_ARGS(&tempResource));
device->CreatePlacedResource(heap, 0, &finalDesc, ..., IID_PPV_ARGS(&finalResource));

// When switching between them, insert an Aliasing Barrier
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_ALIASING;
barrier.Aliasing.pResourceBefore = tempResource;
barrier.Aliasing.pResourceAfter  = finalResource;
commandList->ResourceBarrier(1, &barrier);
```

**Use for:** temporary render targets that are only needed within one render pass — share the memory, save VRAM.

---

## Ring Buffer

A common pattern for per-frame dynamic uploads. Avoids allocating a new staging buffer every frame.

```
┌──────────────────────────────────────────┐
│ Frame0 │ Frame1 │ Frame2 │ (free)        │
│ done   │ GPU    │ CPU    │               │
│        │ active │ writes │               │
└──────────────────────────────────────────┘
          ↑        ↑        ↑
       reusable  in-use   writing now
```

One large Upload Heap buffer is permanently mapped. Each frame allocates a chunk from it. When the write pointer reaches the end, it wraps around. Before overwriting an old frame's region, the CPU waits for the fence value associated with that region.

```cpp
class UploadRingBuffer {
    ID3D12Resource* buffer;
    UINT8*  mappedData;
    UINT64  size;
    UINT64  writeOffset;
    UINT64  fenceValues[FRAME_COUNT];  // fence value per frame end position

public:
    struct Allocation {
        D3D12_GPU_VIRTUAL_ADDRESS gpuAddress;
        void* cpuAddress;
    };

    Allocation Allocate(UINT64 sizeInBytes, UINT64 alignment) {
        UINT64 alignedOffset = AlignUp(writeOffset, alignment);

        if (alignedOffset + sizeInBytes > size)
            alignedOffset = 0;  // wrap around

        WaitForFenceIfNeeded(alignedOffset, sizeInBytes);  // don't overwrite in-flight data

        Allocation alloc;
        alloc.gpuAddress = buffer->GetGPUVirtualAddress() + alignedOffset;
        alloc.cpuAddress = mappedData + alignedOffset;

        writeOffset = alignedOffset + sizeInBytes;
        return alloc;
    }
};
```
