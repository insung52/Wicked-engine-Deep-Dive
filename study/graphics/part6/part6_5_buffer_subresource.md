# Part 6-5: Buffer Subresource and SingleDescriptor

> Index: [Part 6 Index](README.md)
> Related: [Part 6-3](part6_3_descriptor_binder.md) — DescriptorBinder (uses these concepts in flush())

---

## Warning: "Subresource" Has Two Different Meanings

In DX12 documentation, "subresource" usually means a **mip level or array slice of a texture** (e.g., mip 0, mip 1, array slice 2).

In this engine (VizMotive/WickedEngine), "subresource" on a **buffer** means something different: a **specific byte range within one large buffer**. 

These are unrelated concepts that share the same name.

This document covers the buffer meaning only.

---

## Why Buffers Have Subresources

A single `GPUBuffer` is just a flat block of GPU memory. The engine often allocates one large buffer and uses different regions of it for different purposes — this is called **suballocation**.

```
GPUBuffer (1 MB total allocation)
┌─────────────────────────────────────────────┐
│  offset=0:      Instance data A  (512 KB)   │  ← subresource index 0
│  offset=512 KB: Instance data B  (512 KB)   │  ← subresource index 1
└─────────────────────────────────────────────┘
```

Each region is a "subresource" — it is a logical view into part of the buffer. The GPU sees one buffer; the engine tracks multiple views into it.

**Why suballocate instead of creating multiple small buffers?**
- Fewer DX12 resource objects → less driver overhead
- Better memory locality
- One allocation, multiple uses

---

## CreateSubresource() — Registering a View

To expose a region to shaders, you call `CreateSubresource()` on the buffer:

```cpp
// Register a read-only view (SRV) starting at byte offset 512KB
int subresource_index = device->CreateSubresource(
    &gpuBuffer,
    SubresourceType::SRV,
    offset,       // byte offset into the buffer = 512 * 1024
    size,         // how many bytes this view covers
    format        // element format (e.g., FLOAT)
);
// Returns an integer index — use this index to bind the view later
```

Internally this creates a `SingleDescriptor` and appends it to an array inside the buffer's internal state:

```cpp
// Inside GPUBuffer's internal state (Resource_DX12):
std::vector<SingleDescriptor> subresources_srv;   // SRV views
std::vector<SingleDescriptor> subresources_uav;   // UAV views
```

The returned `subresource_index` is the position in that array.

---

## SingleDescriptor — What It Is

`SingleDescriptor` is the engine's struct that represents **one descriptor for one buffer view**.

```cpp
struct SingleDescriptor
{
    D3D12_CPU_DESCRIPTOR_HANDLE handle = {};   // CPU handle into the descriptor heap
    int   index         = -1;                  // bindless index (for shader access without binding)
    uint64_t buffer_offset = 0;               // byte offset from buffer start (for Root Descriptor)

    bool IsValid() const { return handle.ptr != 0; }
};
```

| Field | Purpose |
|-------|---------|
| `handle` | CPU-side pointer into a descriptor heap. Used to copy into GPU-visible heap for Descriptor Table binding. |
| `index` | Bindless index — the shader can access this descriptor by number, without explicit binding. |
| `buffer_offset` | The byte offset from the buffer's start address. Required when binding via Root Descriptor (see below). |

---

## How Subresource Index Is Used in Binding

When you call `device->BindResource(&buffer, subresource_index, t0, cmd)`, the engine stores the index:

```cpp
table.SRV_index[0] = subresource_index;  // "t0 → subresource N"
```

At flush time, the engine retrieves the `SingleDescriptor` for that index and uses it:

```cpp
// Inside flush() — Root Descriptor path
int subresource = table.SRV_index[param.Descriptor.ShaderRegister];

auto address = internal_state->gpu_address;   // buffer start (byte 0)
if (subresource >= 0)
{
    // Retrieve the SingleDescriptor for this subresource
    address += internal_state->subresources_srv[subresource].buffer_offset;
    //         ↑ add the byte offset stored when CreateSubresource() was called
}
commandList->SetGraphicsRootShaderResourceView(slot, address);
```

`subresource == -1` means "use the whole buffer starting from byte 0" — no offset needed.

---

## Descriptor Table vs Root Descriptor — Where the Offset Lives

For **Descriptor Table** binding, the offset is baked into the descriptor itself at creation time:

```cpp
// Inside CreateSubresource() for Descriptor Table path:
srvDesc.Buffer.FirstElement = offset / sizeof(element);
device->CreateShaderResourceView(resource, &srvDesc, descriptor.handle);
// The descriptor now permanently encodes the start position
// → GPU reads the descriptor and automatically starts at the right byte
```

For **Root Descriptor** binding, the descriptor is not used at all — only the raw GPU address is passed:

```cpp
commandList->SetGraphicsRootShaderResourceView(slot, gpu_address + buffer_offset);
// The offset must be added manually — there is no descriptor to carry it
```

This is why `SingleDescriptor` needs the `buffer_offset` field specifically for the Root Descriptor path.

---

## Summary

```
GPUBuffer (one large allocation)
    │
    ├── CreateSubresource(offset=0,   SRV) → index 0 → subresources_srv[0]
    │       SingleDescriptor { handle=..., index=N, buffer_offset=0 }
    │
    └── CreateSubresource(offset=512K, SRV) → index 1 → subresources_srv[1]
            SingleDescriptor { handle=..., index=M, buffer_offset=524288 }

At draw time:
    BindResource(&buffer, subresource_index=1, t0)
        → table.SRV_index[0] = 1

    flush() → Root Descriptor path:
        subresource = table.SRV_index[0]           // = 1
        descriptor  = subresources_srv[1]          // buffer_offset = 524288
        address     = gpu_address + 524288         // correct position
        SetGraphicsRootShaderResourceView(slot, address)
```
