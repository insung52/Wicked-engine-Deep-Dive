# Part 3-7: Engine Command List Abstraction Layer

> Index: [Part 3 Index](part3_index.md)
> Previous: [Part 3-6](part3_6_swapchain.md) — SwapChain
> Next: [Part 3-8](part3_8_resource_states.md) — Resource States

---

## Overview

WickedEngine (and VizMotive) does not call DX12 APIs directly in rendering code. Instead, it wraps them in an abstraction layer. This document explains that layer.

```
[Top — Engine rendering code]
  CommandList (= integer ID handle)       ← obtained from BeginCommandList()
  │  internally: CommandList_DX12 = {
  │      allocator[BUFFERCOUNT][QUEUE_COUNT]
  │      commandList[QUEUE_COUNT]
  │      DescriptorBinder, ...
  │  }
  │
  ├─ WaitCommandList(cmd_A, cmd_B)        ← enforce GPU execution order
  └─ SubmitCommandLists()                 ← submit all queues at end of frame

[Middle — Engine DX12 internals]
  CommandQueue::submit()                  ← collects submit_cmds, calls ExecuteCommandLists()

[Bottom — DX12 API]
  ID3D12CommandQueue                      ← real GPU execution pipeline
  ID3D12GraphicsCommandList               ← command buffer sent to GPU
  ID3D12CommandQueue::ExecuteCommandLists ← direct GPU submission
```

---

## Key Functions

| Function | Level | Role |
|----------|-------|------|
| `ID3D12CommandQueue::ExecuteCommandLists()` | DX12 API (direct) | Submits command lists to the GPU queue |
| `device->BeginCommandList()` | Engine backend API (public) | Starts command recording on a CPU thread, returns a `CommandList` handle |
| `device->WaitCommandList(A, B)` | Engine backend API (public) | **GPU-side ordering**: makes A wait for B to complete before A executes |
| `GraphicsDevice_DX12::SubmitCommandLists()` | Engine backend API (public) | Submits all queues for the frame + fence signal + present + **CPU wait** |
| `CommandQueue::submit()` | Engine backend API (internal) | Collects queued `submit_cmds` and calls `ExecuteCommandLists()` in one batch |

---

## Call Chain

```
SubmitCommandLists()                    ← called once per frame (includes CPU-side wait)
  └─ queue.submit()                     ← called per queue type
       └─ queue->ExecuteCommandLists()  ← DX12 API
```

---

## CommandList Handle

In engine code, `CommandList` is an **integer ID**, not a DX12 object. It is a lightweight handle that maps internally to a `CommandList_DX12`.

```cpp
// Engine rendering code
CommandList cmd = device->BeginCommandList(QUEUE_TYPE_GRAPHICS);

device->RenderPassBegin(&renderPass, cmd);
device->DrawInstanced(vertexCount, 1, 0, cmd);
device->RenderPassEnd(cmd);

// At end of frame:
device->SubmitCommandLists();
```

The handle is intentionally cheap — passing it around doesn't copy any DX12 objects.

---

## WaitCommandList — GPU-Side Ordering

`WaitCommandList(cmd_A, cmd_B)` ensures cmd_A **does not start on the GPU** until cmd_B has completed.

This is different from a CPU wait — the CPU continues, but the GPU internally stalls cmd_A's queue until cmd_B's queue signals completion.

```cpp
// Example: shadow map must be complete before lighting reads it
CommandList shadowCmd   = device->BeginCommandList(QUEUE_TYPE_GRAPHICS);
CommandList lightingCmd = device->BeginCommandList(QUEUE_TYPE_GRAPHICS);

// Record shadow pass into shadowCmd
// Record lighting pass into lightingCmd

// Enforce: lightingCmd waits for shadowCmd on the GPU
device->WaitCommandList(lightingCmd, shadowCmd);

device->SubmitCommandLists();
```

---

## SubmitCommandLists — End of Frame

Called once at the end of each frame. It:
1. Flushes all recorded command lists to their respective queues (`queue.submit()`)
2. Signals fences for each queue
3. Calls `Present()` on the swap chain
4. CPU-waits for the oldest frame to complete (double/triple buffer rotation)

This single call replaces the sequence of explicit `ExecuteCommandLists` + `Signal` + `Present` + `WaitForSingleObject` that you would write in raw DX12.
