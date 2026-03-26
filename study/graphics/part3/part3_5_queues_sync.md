# Part 3-5: Queues and Synchronization

> Index: [Part 3 Index](part3_index.md)
> Previous: [Part 3-4](part3_4_command_list.md) — Command List
> Next: [Part 3-6](part3_6_swapchain.md) — SwapChain

---

## Queue Types

```
┌──────────────────────────────────────────────────────────┐
│                         GPU                              │
│  ┌───────────────────┐  ┌──────────────────┐  ┌───────┐  │
│  │  Graphics Engine  │  │  Compute Engine  │  │ Copy  │  │
│  │  - rendering      │  │  - compute       │  │ Engine│  │
│  │  - compute        │  │  - copy          │  │       │  │
│  │  - copy           │  │                  │  │       │  │
│  └────────┬──────────┘  └────────┬─────────┘  └───┬───┘  │
└───────────│─────────────────────│────────────────│──────┘
            ↑                     ↑                ↑
      Direct Queue          Compute Queue      Copy Queue
```

| Queue Type | Available Commands | Main Use |
|------------|-------------------|---------|
| `DIRECT` | graphics + compute + copy | Main rendering |
| `COMPUTE` | compute + copy | Async compute |
| `COPY` | copy only | Texture streaming, uploads |

---

## Multi-Queue: Async Compute

Running rendering and compute simultaneously on the GPU increases overall GPU utilization.

```
Time →
Direct Queue:  [Render A][Render B][Render C]
Compute Queue:           [Compute 1][Compute 2]
                          ←  parallel execution  →
```

```cpp
void RenderFrame() {
    // 1. Start shadow + GBuffer rendering
    directQueue->ExecuteCommandLists(1, &shadowList);
    directQueue->ExecuteCommandLists(1, &gBufferList);

    // 2. Start compute work in parallel (SSAO, particles, etc.)
    computeQueue->ExecuteCommandLists(1, &computeList);
    computeQueue->Signal(computeFence, ++computeFenceValue);

    // 3. Continue graphics work
    directQueue->ExecuteCommandLists(1, &lightingList);

    // 4. When compute result is needed — GPU waits for compute to finish
    directQueue->Wait(computeFence, computeFenceValue);
    directQueue->ExecuteCommandLists(1, &postProcessList);

    // 5. Present
    swapChain->Present(1, 0);
}
```

---

## Why Synchronization is Necessary

The CPU and GPU run **asynchronously**. Without synchronization, data races happen.

```
Problem scenario:
CPU: write Data A to buffer
CPU: submit "draw this buffer" to GPU
CPU: overwrite the same buffer with Data B  ← problem!
GPU: still rendering with Data A

Result: GPU reads partially corrupted data → broken rendering
```

---

## Fence — CPU-GPU Synchronization

A counter-based object that lets the CPU check whether the GPU has reached a specific point.

```cpp
ID3D12Fence* fence;
UINT64 fenceValue = 0;
device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&fence));
HANDLE fenceEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
```

### Pattern 1: Simple Completion Wait

```cpp
// Submit commands, then ask GPU to signal when done
commandQueue->ExecuteCommandLists(1, &cmdList);
commandQueue->Signal(fence, ++fenceValue);  // GPU sets fence to fenceValue when it reaches this point

// CPU waits
if (fence->GetCompletedValue() < fenceValue) {
    fence->SetEventOnCompletion(fenceValue, fenceEvent);
    WaitForSingleObject(fenceEvent, INFINITE);
}
// GPU work is guaranteed complete here
```

### Pattern 2: Frame Synchronization (Double Buffering)

```cpp
const int FRAME_COUNT = 2;
UINT64 frameFenceValues[FRAME_COUNT] = {};
int currentFrame = 0;

void RenderFrame() {
    // Wait for this slot's previous use to complete
    if (fence->GetCompletedValue() < frameFenceValues[currentFrame]) {
        fence->SetEventOnCompletion(frameFenceValues[currentFrame], fenceEvent);
        WaitForSingleObject(fenceEvent, INFINITE);
    }

    // Render...
    commandQueue->ExecuteCommandLists(1, &cmdList);

    // Register a signal for when this frame completes
    frameFenceValues[currentFrame] = ++fenceValue;
    commandQueue->Signal(fence, fenceValue);

    swapChain->Present(1, 0);
    currentFrame = (currentFrame + 1) % FRAME_COUNT;
}
```

With this pattern, the CPU can run up to 2 frames ahead. If the GPU falls behind, the CPU stalls at the fence wait.

---

## Semaphore — GPU-GPU Synchronization

While a Fence is for CPU↔GPU sync, a Semaphore enforces **ordering between GPU queues**.

```
Fence:     CPU ←→ GPU  (CPU waits for GPU to complete)
Semaphore: GPU ←→ GPU  (Queue A must finish before Queue B starts)
```

### Queue-to-Queue Synchronization in DX12

DX12 has no separate Semaphore type. It uses **Fence for inter-queue synchronization too**.

```cpp
// Copy Queue signals when copy is done
copyQueue->ExecuteCommandLists(1, &copyCommandList);
copyQueue->Signal(syncFence, ++syncFenceValue);   // "copy complete"

// Graphics Queue waits on the GPU side (does NOT block the CPU)
graphicsQueue->Wait(syncFence, syncFenceValue);   // GPU-side wait
graphicsQueue->ExecuteCommandLists(1, &renderList);
```

`Queue->Wait()` is a GPU-side wait — it does not block the CPU thread.

### Semaphores in Vulkan

Vulkan provides Semaphore as an explicit type:

```cpp
// Binary Semaphore — one signal, one wait
VkSemaphore imageAvailable;  // swapchain image acquisition complete
VkSemaphore renderFinished;  // rendering complete

// Acquire image — signals imageAvailable
vkAcquireNextImageKHR(device, swapchain, ..., imageAvailable, ...);

// Submit: wait for imageAvailable, signal renderFinished when done
VkSubmitInfo submitInfo = {};
submitInfo.pWaitSemaphores   = &imageAvailable;
submitInfo.pSignalSemaphores = &renderFinished;

// Present: wait for renderFinished
VkPresentInfoKHR presentInfo = {};
presentInfo.pWaitSemaphores = &renderFinished;
vkQueuePresentKHR(presentQueue, &presentInfo);
```

Timeline Semaphore (Vulkan 1.2+) is conceptually identical to DX12 Fence — counter-based, usable for both CPU and GPU waiting.

---

## Synchronization Object Comparison

| Property | DX12 Fence | Vulkan Fence | Vulkan Binary Semaphore | Vulkan Timeline Semaphore |
|----------|------------|--------------|------------------------|---------------------------|
| CPU wait | ✅ | ✅ | ❌ | ✅ |
| GPU wait | ✅ | ❌ | ✅ | ✅ |
| Counter-based | ✅ (uint64) | ❌ (bool) | ❌ (bool) | ✅ (uint64) |
| Primary use | CPU↔GPU, GPU↔GPU | CPU↔GPU | GPU↔GPU | Both |

---

## Full Frame Sync Flow (Vulkan example)

```
Acquire image      Render           Present
┌─────────────┐  imageAvailable  ┌─────────────┐  renderFinished  ┌─────────────┐
│  Swapchain  │────────────────▶│  Graphics   │────────────────▶│   Display   │
│  (acquire)  │   semaphore      │    Queue    │    semaphore     │   (flip)    │
└─────────────┘                  └─────────────┘                  └─────────────┘

Without semaphores:
→ Rendering starts before image acquisition is ready
→ Presenting starts before rendering is complete
→ Torn/corrupted frames
```
