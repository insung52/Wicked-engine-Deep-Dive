# Part 4-4: Resource States and Barriers

> Index: [Part 4 Index](README.md)
> Previous: [Part 4-3](part4_3_descriptors.md) — Descriptors and Binding
> Next: [Part 4-5](part4_5_textures.md) — Textures

---

## What Is a Resource State?

A resource state describes **what the GPU is currently doing with a resource** — reading it as a texture, writing it as a render target, copying it, etc.

The GPU optimizes memory access differently depending on the state:
- **Read-only state** → GPU can cache aggressively
- **Write state** → GPU must flush caches before others read
- **Compressed state** → GPU must decompress before reading

If you use a resource in a state it wasn't prepared for, you get corrupted data or a crash.

---

## Common Resource States

```cpp
// General / initial state
D3D12_RESOURCE_STATE_COMMON

// Read-only states (can be OR'd together)
D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER  // VB or CB
D3D12_RESOURCE_STATE_INDEX_BUFFER                // IB
D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE   // SRV in VS/GS/CS
D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE       // SRV in PS
D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT           // argument buffer for ExecuteIndirect
D3D12_RESOURCE_STATE_COPY_SOURCE                 // copy source
D3D12_RESOURCE_STATE_DEPTH_READ                  // depth read-only

// Write states
D3D12_RESOURCE_STATE_RENDER_TARGET               // render target write
D3D12_RESOURCE_STATE_UNORDERED_ACCESS            // UAV read/write
D3D12_RESOURCE_STATE_DEPTH_WRITE                 // depth write
D3D12_RESOURCE_STATE_COPY_DEST                   // copy destination
D3D12_RESOURCE_STATE_STREAM_OUT                  // stream output

// Special states
D3D12_RESOURCE_STATE_PRESENT                     // ready to display (SwapChain)
D3D12_RESOURCE_STATE_GENERIC_READ                // Upload Heap resources

// Combined example
D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE |
D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE   // readable in all shader stages
```

---

## Why Barriers Are Needed

The GPU executes commands out-of-order and caches memory aggressively. Without an explicit signal, the GPU doesn't know when it's safe to switch from writing to reading:

```
Scenario: render to RT → read as texture in next pass

Pass 1: Render scene into RT
        GPU writes to RT...
        (data may still be in write cache, not flushed to memory)

[No barrier — immediately try to read:]
        GPU reads RT as SRV
        → write cache not flushed
        → read happens before write completes
        → corrupted / stale data

[With barrier:]
        GPU: "flush all writes, wait for RT write to complete"
        GPU: "now mark it as SRV — reads can proceed"
        → correct data
```

---

## Transition Barrier

```cpp
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type                   = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Flags                  = D3D12_RESOURCE_BARRIER_FLAG_NONE;
barrier.Transition.pResource   = texture;
barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;

commandList->ResourceBarrier(1, &barrier);
```

### Common Transition Patterns

```
RT  → SRV      render pass done, now read as texture
SRV → RT       re-render into a previously-read texture
RT  → PRESENT  frame done, hand to SwapChain
PRESENT → RT   new frame, reclaim from SwapChain

COPY_DEST → SRV   upload finished, now use as shader resource
UAV       → SRV   compute write done, now read in graphics pass
UAV       → UAV   consecutive compute writes (needs UAV barrier, see below)
```

---

## UAV Barrier

When the same resource is written by one compute dispatch and then read or written again by another, use a UAV Barrier to guarantee ordering:

```cpp
D3D12_RESOURCE_BARRIER uavBarrier = {};
uavBarrier.Type     = D3D12_RESOURCE_BARRIER_TYPE_UAV;
uavBarrier.UAV.pResource = rwBuffer;
commandList->ResourceBarrier(1, &uavBarrier);
```

---

## Aliasing Barrier

Required when switching between two resources that alias the same heap memory:

```cpp
D3D12_RESOURCE_BARRIER aliasingBarrier = {};
aliasingBarrier.Type                       = D3D12_RESOURCE_BARRIER_TYPE_ALIASING;
aliasingBarrier.Aliasing.pResourceBefore   = tempResource;
aliasingBarrier.Aliasing.pResourceAfter    = finalResource;
commandList->ResourceBarrier(1, &aliasingBarrier);
```

---

## Resource State Tracker

An engine typically tracks current resource states in a map so it can automatically insert the correct barriers:

```cpp
class ResourceStateTracker {
    std::unordered_map<ID3D12Resource*, D3D12_RESOURCE_STATES> states;

public:
    void Transition(ID3D12Resource* resource,
                    D3D12_RESOURCE_STATES newState,
                    std::vector<D3D12_RESOURCE_BARRIER>& barriers)
    {
        auto& current = states[resource];
        if (current != newState) {
            D3D12_RESOURCE_BARRIER b = {};
            b.Type                        = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
            b.Transition.pResource        = resource;
            b.Transition.StateBefore      = current;
            b.Transition.StateAfter       = newState;
            b.Transition.Subresource      = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
            barriers.push_back(b);
            current = newState;
        }
    }

    void Flush(ID3D12GraphicsCommandList* cmdList) {
        if (!pendingBarriers.empty()) {
            cmdList->ResourceBarrier((UINT)pendingBarriers.size(), pendingBarriers.data());
            pendingBarriers.clear();
        }
    }
};
```

Batch multiple barriers into a single `ResourceBarrier()` call — this is cheaper than calling it once per barrier.

---

## Enhanced Barrier (DX12 Agility SDK)

The legacy `D3D12_RESOURCE_BARRIER_TYPE_TRANSITION` specifies only resource state, not exact GPU pipeline sync points. The **Enhanced Barrier** API provides finer control:

```cpp
D3D12_TEXTURE_BARRIER textureBarrier = {};
textureBarrier.SyncBefore   = D3D12_BARRIER_SYNC_RENDER_TARGET;   // wait for RT writes
textureBarrier.SyncAfter    = D3D12_BARRIER_SYNC_PIXEL_SHADING;   // unblock at PS reads
textureBarrier.AccessBefore = D3D12_BARRIER_ACCESS_RENDER_TARGET;
textureBarrier.AccessAfter  = D3D12_BARRIER_ACCESS_SHADER_RESOURCE;
textureBarrier.LayoutBefore = D3D12_BARRIER_LAYOUT_RENDER_TARGET;
textureBarrier.LayoutAfter  = D3D12_BARRIER_LAYOUT_SHADER_RESOURCE;
textureBarrier.pResource    = texture;
```

**Benefit:** you specify exactly which pipeline stage needs to wait and which can proceed — eliminating unnecessary stalls that legacy barriers can introduce.
