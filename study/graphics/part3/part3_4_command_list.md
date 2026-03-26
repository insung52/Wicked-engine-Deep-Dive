# Part 3-4: Command Buffer / Command List

> Index: [Part 3 Index](part3_index.md)
> Previous: [Part 3-3](part3_3_root_signature.md) — Root Signature
> Next: [Part 3-5](part3_5_queues_sync.md) — Queues & Synchronization

---

## Why Command Buffers?

```
Direct calls (OpenGL style):
CPU: "draw triangle" → GPU executes immediately
CPU: "draw next"    → GPU executes immediately
→ Many CPU-GPU round trips, high synchronization overhead

Command buffer (DX12 style):
CPU: "draw triangle" → writes to buffer
CPU: "draw next"     → writes to buffer
CPU: "done, execute" → GPU runs the entire buffer at once
→ One round trip, minimal CPU-GPU overhead
```

Additional benefits:
- Commands can be recorded on multiple threads simultaneously
- CPU can reorder/optimize before submission
- Recording and submission timing are decoupled

---

## Command List Recording — Full Example

```cpp
// 1. Reset allocator (only after GPU has finished using it)
commandAllocator->Reset();
commandList->Reset(commandAllocator, pso);

// 2. Set render targets
D3D12_CPU_DESCRIPTOR_HANDLE rtv = rtvHeap->GetCPUDescriptorHandleForHeapStart();
D3D12_CPU_DESCRIPTOR_HANDLE dsv = dsvHeap->GetCPUDescriptorHandleForHeapStart();
commandList->OMSetRenderTargets(1, &rtv, FALSE, &dsv);

// 3. Set viewport and scissor
commandList->RSSetViewports(1, &viewport);
commandList->RSSetScissorRects(1, &scissorRect);

// 4. Transition resource state (PRESENT → RENDER_TARGET)
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type                   = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource   = renderTarget;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_PRESENT;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_RENDER_TARGET;
commandList->ResourceBarrier(1, &barrier);

// 5. Clear
float clearColor[] = { 0.0f, 0.0f, 0.0f, 1.0f };
commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);
commandList->ClearDepthStencilView(dsv, D3D12_CLEAR_FLAG_DEPTH, 1.0f, 0, 0, nullptr);

// 6. Set pipeline and root signature
commandList->SetPipelineState(pso);
commandList->SetGraphicsRootSignature(rootSig);

// 7. Bind resources
commandList->SetGraphicsRootConstantBufferView(0, cbGpuAddress);
ID3D12DescriptorHeap* heaps[] = { srvHeap };
commandList->SetDescriptorHeaps(1, heaps);
commandList->SetGraphicsRootDescriptorTable(1, srvGpuHandle);

// 8. Set vertex and index buffers
commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
commandList->IASetVertexBuffers(0, 1, &vbView);
commandList->IASetIndexBuffer(&ibView);

// 9. Draw
commandList->DrawIndexedInstanced(indexCount, 1, 0, 0, 0);

// 10. Prepare for Present (RENDER_TARGET → PRESENT)
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_PRESENT;
commandList->ResourceBarrier(1, &barrier);

// 11. End recording
commandList->Close();
```

---

## Multithreaded Recording

One of DX12's key advantages: multiple threads can record commands **simultaneously**.

Requirements: each thread must have its own Command Allocator and Command List. Sharing one allocator/list across threads is not allowed.

```cpp
struct ThreadContext {
    ID3D12CommandAllocator*    allocator;
    ID3D12GraphicsCommandList* commandList;
};

void RenderFrame() {
    // Record in parallel across threads
    parallel_for(0, numThreads, [](int threadIdx) {
        auto& ctx = threadContexts[threadIdx];
        ctx.allocator->Reset();
        ctx.commandList->Reset(ctx.allocator, pso);

        // Each thread records its assigned objects
        for (auto& obj : objectsForThread[threadIdx]) {
            RecordDrawCommands(ctx.commandList, obj);
        }

        ctx.commandList->Close();
    });

    // Collect all lists and submit together
    std::vector<ID3D12CommandList*> lists;
    for (auto& ctx : threadContexts)
        lists.push_back(ctx.commandList);

    commandQueue->ExecuteCommandLists(lists.size(), lists.data());
}
```

**Note**: the GPU executes the lists in the order they appear in `ExecuteCommandLists`. The order in which threads finish recording does not matter — what matters is the final submission order.

---

## Bundles

A bundle lets you **pre-record a sequence of commands once** and reuse it multiple times. Useful for a draw sequence that repeats identically across many objects.

```cpp
// Create the bundle (done once)
ID3D12GraphicsCommandList* bundle;
device->CreateCommandList(0, D3D12_COMMAND_LIST_TYPE_BUNDLE,
                          bundleAllocator, pso, IID_PPV_ARGS(&bundle));

bundle->SetGraphicsRootSignature(rootSig);
bundle->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
bundle->IASetVertexBuffers(0, 1, &vbView);
bundle->IASetIndexBuffer(&ibView);
bundle->DrawIndexedInstanced(indexCount, 1, 0, 0, 0);
bundle->Close();

// Use it every frame
commandList->ExecuteBundle(bundle);
```

Bundle restrictions:
- Cannot change Render Targets
- Cannot insert resource barriers
- Cannot call Clear

---

## Allocator Reset Rules

A Command Allocator can only be Reset **after the GPU has finished using it**.

```
Frame 0: Allocator[0] recorded → submitted → GPU executing
Frame 1: Allocator[1] recorded → submitted → GPU executing
Frame 2: Fence confirms Frame 0 complete → Allocator[0] can be Reset

→ Keep one allocator per frame (same count as double/triple buffering)
```

Resetting an allocator without confirming GPU completion means overwriting memory the GPU is still reading.
