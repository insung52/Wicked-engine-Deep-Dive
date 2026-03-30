# Part 3-8: Resource States and Barriers

> Index: [Part 3 Index](part3_index.md)
> Previous: [Part 3-7](part3_7_engine_abstraction.md) — Engine Abstraction
> Next: [Part 4](../part4/README.md) — Resource Management

---

## What is a Resource State?

Every GPU resource (texture, buffer) has a **state** that describes how it is currently being used. The same texture behaves differently when it is a render target versus when a shader is sampling it — the GPU optimizes its internal caches and memory layout differently for each case.

In DX12, the developer must **explicitly track and transition states**. DX11 had the driver do this automatically, but that added unpredictable overhead.

---

## Common Resource States

| State | Meaning | Typical Use |
|-------|---------|------------|
| `RENDER_TARGET` | Graphics pipeline writes pixels here | `RenderPassBegin`, draw call output |
| `SHADER_RESOURCE` | Read-only access from a shader | SRV binding, `texture.Sample()` |
| `UNORDERED_ACCESS` | Read/write access from a shader | UAV binding, compute shader output |
| `COPY_SRC` | Source of a copy command | `CopyResource`, `CopyTextureRegion` |
| `COPY_DST` | Destination of a copy command | `CopyResource`, post-upload |
| `DEPTH_WRITE` | Depth/stencil write | DSV, depth test + write |
| `DEPTH_READ` | Depth buffer read-only | Shadow map sampling |
| `PRESENT` | Ready for `Present()` | Must be in this state before swap |
| `COMMON` | General initial state | Right after creation, cross-queue sharing |

---

## Why State Transitions are Necessary

The GPU keeps different caches active for different states:

```
RENDER_TARGET state:
  Data may be in the ROP write cache (not yet flushed to memory)
  Memory layout optimized for fast pixel writes

SHADER_RESOURCE state:
  Data expected in the texture cache (L1/L2)
  Memory layout optimized for sampling
```

If you write to a resource in `RENDER_TARGET` state and then immediately read it in a shader without a barrier:
- The write cache may not have flushed yet
- The shader reads stale data from a different cache
- Result: **undefined behavior** — the shader sees old or garbage values

---

## What is a Barrier?

A barrier is a command that tells the GPU: "I am about to use this resource differently — flush your caches, complete any pending work, and switch the resource to the new state."

```cpp
// Raw DX12
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type                   = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource   = texture.Get();
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;
commandList->ResourceBarrier(1, &barrier);

// VizMotive abstraction
device->Barrier(GPUBarrier::Image(&texture,
    ResourceState::RENDER_TARGET,
    ResourceState::SHADER_RESOURCE), cmd);
```

---

## Barrier Cost

Barriers are not free. When transitioning state, the GPU must:

| Work | Description |
|------|-------------|
| Pipeline stall | Wait for all in-flight work on this resource to complete |
| Cache flush | Write dirty cache lines back to memory |
| Cache invalidate | Mark read caches as stale |
| Layout conversion | On some hardware, physically reformat memory layout |

Minimizing unnecessary barriers is a significant part of GPU performance optimization.

---

## Barrier Types

```
Transition Barrier  — state change (most common)
  RENDER_TARGET → SHADER_RESOURCE
  RENDER_TARGET → PRESENT
  COPY_SRC      → RENDER_TARGET
  etc.

UAV Barrier         — prevents read/write hazard on the same UAV resource
  Compute A writes UAV → Compute B reads the same UAV
  (without this, B may start before A's writes are visible)

Aliasing Barrier    — when two resources share the same heap memory
  Advanced optimization, not commonly needed
```

---

## SwapChain Buffer State Transitions

The back buffer transitions each frame:

```
Frame start:  PRESENT → RENDER_TARGET   (before blit writes to it)
After blit:   RENDER_TARGET → PRESENT   (before calling Present())
```

If `DXGI_USAGE_SHADER_INPUT` is enabled, additional transitions are possible:
```
RENDER_TARGET → SHADER_RESOURCE    (shader reads from it — e.g., post-process)
SHADER_RESOURCE → COPY_SRC         (before CopyResource)
```

---

## Barrier Batching

Multiple barriers submitted in one call allow the GPU to handle them in parallel where possible.

```cpp
// Inefficient — one barrier at a time
commandList->ResourceBarrier(1, &barrier1);
commandList->ResourceBarrier(1, &barrier2);

// Efficient — batch together
D3D12_RESOURCE_BARRIER barriers[2] = { barrier1, barrier2 };
commandList->ResourceBarrier(2, barriers);
```

VizMotive implements this with a `Barrier()` / internal flush pattern:

```cpp
// Accumulate barriers
device->Barrier(GPUBarrier::Image(&texA, stateA1, stateA2), cmd);
device->Barrier(GPUBarrier::Image(&texB, stateB1, stateB2), cmd);
// Internally flushed as a single ResourceBarrier call at the right point
```

---

## Split Barriers

A split barrier separates the "begin" and "end" of a long transition, allowing the GPU to do other work in between. This reduces pipeline stalls.

```cpp
// Begin transition
barrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_BEGIN_ONLY;
commandList->ResourceBarrier(1, &barrier);

// GPU can do other work while the transition is in progress
commandList->DrawIndexedInstanced(...);

// End transition — resource is now in the new state
barrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_END_ONLY;
commandList->ResourceBarrier(1, &barrier);
```

Use this when a transition is expensive (e.g., decompression) and there is other independent work to fill the gap.
