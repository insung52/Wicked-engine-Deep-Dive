# Part 4-6: Indirect Draw and ExecuteIndirect

> Index: [Part 4 Index](README.md)
> Previous: [Part 4-5](part4_5_textures.md) — Textures

---

## What Is Indirect Draw?

In a normal draw call, the CPU specifies the arguments:

```cpp
commandList->DrawInstanced(36, 1, 0, 0);   // CPU decides: 36 vertices, 1 instance
```

In an **indirect draw**, the CPU says "read the draw arguments from this GPU buffer":

```cpp
commandList->ExecuteIndirect(commandSignature, maxCount, argBuffer, argOffset, countBuffer, countOffset);
// GPU reads DrawInstanced arguments from argBuffer at runtime
```

This lets the **GPU decide** what and how much to draw — after GPU-side culling, LOD selection, or other compute operations that determine which objects are visible.

---

## Why It Matters

### GPU-Driven Rendering

```
Traditional (CPU-driven):
  CPU: culls objects → calls DrawInstanced() for each visible object
  Problem: CPU must wait for GPU, many draw calls = driver overhead

GPU-Driven:
  GPU compute: culls objects → writes DrawInstanced args into a buffer
  CPU: one ExecuteIndirect() call → GPU reads args and draws everything
  Benefit: GPU stays busy, no CPU/GPU synchronization per object
```

### Use Cases
- Frustum culling on the GPU
- GPU-side LOD selection
- Particle systems (GPU decides how many particles to emit)
- Any case where the visible set is determined by a compute pass

---

## CommandSignature

`ExecuteIndirect` requires a **CommandSignature** that describes what kind of commands are in the argument buffer and in what order.

```cpp
// Define the indirect command layout
D3D12_INDIRECT_ARGUMENT_DESC argDescs[1] = {};
argDescs[0].Type = D3D12_INDIRECT_ARGUMENT_TYPE_DRAW;  // DrawInstanced

D3D12_COMMAND_SIGNATURE_DESC sigDesc = {};
sigDesc.ByteStride       = sizeof(D3D12_DRAW_ARGUMENTS);  // stride between entries
sigDesc.NumArgumentDescs = 1;
sigDesc.pArgumentDescs   = argDescs;

ID3D12CommandSignature* commandSignature;
device->CreateCommandSignature(&sigDesc, nullptr, IID_PPV_ARGS(&commandSignature));
```

For indexed draw, use `D3D12_INDIRECT_ARGUMENT_TYPE_DRAW_INDEXED`:

```cpp
argDescs[0].Type = D3D12_INDIRECT_ARGUMENT_TYPE_DRAW_INDEXED;
sigDesc.ByteStride = sizeof(D3D12_DRAW_INDEXED_ARGUMENTS);
```

---

## Argument Buffer Layout

The argument buffer is a GPU buffer containing an array of draw argument structs:

```cpp
// For DrawInstanced:
struct D3D12_DRAW_ARGUMENTS {
    UINT VertexCountPerInstance;
    UINT InstanceCount;
    UINT StartVertexLocation;
    UINT StartInstanceLocation;
};

// For DrawIndexedInstanced:
struct D3D12_DRAW_INDEXED_ARGUMENTS {
    UINT IndexCountPerInstance;
    UINT InstanceCount;
    UINT StartIndexLocation;
    INT  BaseVertexLocation;
    UINT StartInstanceLocation;
};
```

The compute shader that generates draw commands writes into this buffer:

```hlsl
// Compute shader: one thread per object
RWStructuredBuffer<DrawArgs> drawArgBuffer : register(u0);
RWBuffer<uint>               drawCount     : register(u1);

[numthreads(64, 1, 1)]
void CullCS(uint id : SV_DispatchThreadID) {
    if (IsVisible(objects[id])) {
        uint slot;
        InterlockedAdd(drawCount[0], 1, slot);  // atomic counter
        drawArgBuffer[slot].VertexCountPerInstance = objects[id].vertexCount;
        drawArgBuffer[slot].InstanceCount          = 1;
        drawArgBuffer[slot].StartVertexLocation    = objects[id].startVertex;
        drawArgBuffer[slot].StartInstanceLocation  = 0;
    }
}
```

---

## Executing Indirect Commands

```cpp
// Barrier: UNORDERED_ACCESS → INDIRECT_ARGUMENT
// (compute write → indirect argument read)
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type                   = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource   = argBuffer;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_UNORDERED_ACCESS;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT;
commandList->ResourceBarrier(1, &barrier);

// Execute: GPU reads from argBuffer and issues up to maxDrawCount draws
commandList->ExecuteIndirect(
    commandSignature,
    maxDrawCount,       // maximum possible draw calls
    argBuffer,          // argument buffer
    0,                  // byte offset into argBuffer
    countBuffer,        // optional: buffer holding actual count (can be nullptr)
    0                   // byte offset into countBuffer
);
```

---

## Initialization Requirement (Fix #2 Context)

The argument buffer **must be zero-initialized** before first use.

**Why:** On some hardware (Xbox, and some AMD/NVIDIA configurations), if the GPU reads uninitialized memory as draw arguments, it may interpret garbage values as very large vertex or instance counts, causing the hardware to hang or crash.

Zero-initialization ensures that unused argument slots read as "draw 0 vertices / 0 instances" — a no-op rather than a hang.

```cpp
// Option 1: Create with ClearUAV
commandList->ClearUnorderedAccessViewUint(gpuHandle, cpuHandle, argBuffer, zeros, 0, nullptr);

// Option 2: Use a zero-filled Upload Heap copy at creation time
// Copy zeros from an Upload Heap buffer into the arg buffer before first use

// Option 3: Initialize in the compute shader (set all entries to 0 first)
// then fill in the visible ones — but this requires an extra pass
```

**Resource state for clearing:**
```
Initial: COPY_DEST → clear/copy zeros into it
Then: UNORDERED_ACCESS → compute shader writes real args
Then: INDIRECT_ARGUMENT → ExecuteIndirect reads from it
```

This was the bug fixed in WickedEngine commit `bf...` (Fix #2 in apply_bug_fixes.md) — Xbox would hang because the indirect argument buffer wasn't guaranteed to be zeroed before ExecuteIndirect was called.

---

## Summary

```
GPU compute pass:
  Cull → write DrawInstanced args into argBuffer (UAV write)

Barrier: UAV → INDIRECT_ARGUMENT

Graphics pass:
  ExecuteIndirect(commandSignature, maxCount, argBuffer)
  → GPU reads args, issues DrawInstanced for each entry
  → Only visible objects are drawn
  → CPU issued exactly ONE command
```

| Concept | Role |
|---------|------|
| `CommandSignature` | Declares what argument types are in the buffer |
| Argument buffer | GPU buffer holding draw arguments (written by compute) |
| Count buffer | Optional: actual draw count (avoids padding with zero-args) |
| `ExecuteIndirect` | Issues up to N commands from the argument buffer |
| Zero-init requirement | Prevents hardware hang on uninitialized reads |
