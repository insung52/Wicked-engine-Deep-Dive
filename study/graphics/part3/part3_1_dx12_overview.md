# Part 3-1: DX12 Overview and Core Objects

> Previous: [Part 2](../part2/part2_graphics_pipeline.md) — Graphics Pipeline
> Index: [Part 3 Index](part3_index.md)
> Next: [Part 3-2](part3_2_pso.md) — PSO

---

## DX12 Abbreviations

These prefixes and abbreviations appear constantly in DX12 code.

### Interface Prefixes

| Prefix | Meaning | Example |
|--------|---------|---------|
| `ID3D12` | Direct3D 12 interface | `ID3D12Device`, `ID3D12Resource` |
| `IDXGI` | DirectX Graphics Infrastructure | `IDXGISwapChain`, `IDXGIAdapter` |
| `D3D12_` | DX12 struct or enum | `D3D12_RESOURCE_DESC` |
| `DXGI_` | DXGI struct or enum | `DXGI_FORMAT_R8G8B8A8_UNORM` |

### Pipeline Stage Abbreviations

| Abbreviation | Full Name | Role |
|------|-----------|------|
| `IA` | Input Assembler | Assembles vertex/index data |
| `VS` | Vertex Shader | Transforms vertices |
| `HS` | Hull Shader | Tessellation control |
| `DS` | Domain Shader | Processes tessellation output |
| `GS` | Geometry Shader | Generates/discards geometry |
| `RS` | Rasterizer | Converts triangles to pixels |
| `PS` | Pixel Shader | Determines pixel color |
| `OM` | Output Merger | Depth test, blending |
| `CS` | Compute Shader | General-purpose GPU compute |

### View Abbreviations

| Abbreviation | Full Name | Purpose |
|------|-----------|---------|
| `RTV` | Render Target View | Write to a render target |
| `DSV` | Depth Stencil View | Depth/stencil buffer |
| `SRV` | Shader Resource View | Read from a shader |
| `UAV` | Unordered Access View | Read/write from a shader |
| `CBV` | Constant Buffer View | Access a constant buffer |

### Other Important Abbreviations

| Abbreviation | Full Name | Description |
|------|-----------|-------------|
| `PSO` | Pipeline State Object | Bundled pipeline configuration |
| `RS` | Root Signature | Resource binding layout |
| `CB` | Constant Buffer | Small constant data for shaders |
| `VB` | Vertex Buffer | Per-vertex geometry data |
| `IB` | Index Buffer | Index data for indexed drawing |
| `RT` | Render Target | Output buffer for rendering |
| `MRT` | Multiple Render Targets | Writing to multiple RTs at once |

### Synchronization Terms

| Term | Description |
|------|-------------|
| `Fence` | CPU-GPU sync object. CPU waits for GPU completion |
| `Semaphore` | GPU-GPU sync object. Enforces ordering between queues |
| `Barrier` | Resource state transition + memory sync command |
| `Signal` | Mark a sync object as complete |
| `Wait` | Block until a sync object is signaled |

### How to Read DX12 Function Names

```cpp
commandList->IASetVertexBuffers(0, 1, &vbView);
//          IA  Set  VertexBuffers
//          │   │    └─ what: vertex buffers
//          │   └─ action: set
//          └─ where: Input Assembler stage

commandList->OMSetRenderTargets(1, &rtv, FALSE, &dsv);
//          OM  Set  RenderTargets
//          └─ Output Merger stage

device->CreateCommittedResource(...);
//      Create  Committed  Resource
//      │       │          └─ what: a resource
//      │       └─ type: Committed (auto heap allocation)
//      └─ action: create
```

---

## 3.1 What Graphics APIs Do

```
┌─────────────────────────────────────┐
│         Your game / engine          │
├─────────────────────────────────────┤
│   Graphics API (DX12 / Vulkan)      │  ← what we're studying
├─────────────────────────────────────┤
│             GPU driver              │
├─────────────────────────────────────┤
│            GPU hardware             │
└─────────────────────────────────────┘
```

### API Comparison

| API | Platform | Characteristics |
|-----|----------|----------------|
| OpenGL | Cross-platform | Old, easy to learn, high driver overhead |
| DirectX 11 | Windows | More efficient than OpenGL, still widely used |
| **DirectX 12** | Windows, Xbox | Low-level, explicit control, highest performance |
| **Vulkan** | Cross-platform | Same philosophy as DX12, supports Linux/Android |
| Metal | Apple | Apple-only |

WickedEngine uses **DX12** as its primary backend. DX12 and Vulkan share nearly identical concepts — learn one and the other becomes easy.

### Implicit vs Explicit

**OpenGL / DX11 (implicit)**:
```cpp
glBindTexture(GL_TEXTURE_2D, textureID);
glDrawArrays(GL_TRIANGLES, 0, 36);
// The driver handles synchronization, memory management, and state tracking
```

**DX12 / Vulkan (explicit)**:
```cpp
// Manually allocate memory
device->CreateCommittedResource(...);

// Manually transition resource state
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_RENDER_TARGET;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;
commandList->ResourceBarrier(1, &barrier);

// Manually record and submit commands
commandList->DrawInstanced(36, 1, 0, 0);
commandQueue->ExecuteCommandLists(1, &commandList);

// Manually manage synchronization
commandQueue->Signal(fence, fenceValue);
fence->SetEventOnCompletion(fenceValue, event);
WaitForSingleObject(event, INFINITE);
```

**Why use the more complex approach?**
- Reduced driver overhead
- Native multithreading support (record commands on multiple threads simultaneously)
- Predictable, consistent performance
- More room for manual optimization

---

## 3.2 Core DX12 Objects

### Device

Represents the GPU. The central factory for creating all other resources and objects.

```cpp
// Select an adapter (GPU)
IDXGIAdapter1* adapter;
factory->EnumAdapters1(0, &adapter);  // first GPU

// Create the device
ID3D12Device* device;
D3D12CreateDevice(adapter, D3D_FEATURE_LEVEL_12_0, IID_PPV_ARGS(&device));
```

What you can create from a Device:
- Resources (buffers, textures)
- PSOs, Root Signatures
- Command Queues, Command Lists, Command Allocators
- Heaps, Descriptors

### Command Queue

The pipe through which commands are **submitted** to the GPU. The GPU executes only what goes through a queue.

```cpp
D3D12_COMMAND_QUEUE_DESC queueDesc = {};
queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;  // graphics + compute + copy

ID3D12CommandQueue* commandQueue;
device->CreateCommandQueue(&queueDesc, IID_PPV_ARGS(&commandQueue));
```

Queue types:
```
DIRECT:  graphics + compute + copy (all commands)
COMPUTE: compute + copy
COPY:    copy only (data transfer)

Multiple queues can run on the GPU simultaneously:
┌──────────────┐
│  Direct Queue │ → GPU graphics engine
├──────────────┤
│ Compute Queue │ → GPU compute engine
├──────────────┤
│   Copy Queue  │ → GPU copy engine
└──────────────┘
```

### Command Allocator

The memory pool that stores recorded commands. Cannot be Reset while the GPU is still using it.

```cpp
ID3D12CommandAllocator* commandAllocator;
device->CreateCommandAllocator(D3D12_COMMAND_LIST_TYPE_DIRECT, IID_PPV_ARGS(&commandAllocator));

// IMPORTANT: can only Reset after the GPU has finished using it
// Use one allocator per frame, or confirm completion via Fence before reusing
commandAllocator->Reset();
```

### Command List

The object where you **record** commands. After calling `Close()`, submit it to a queue.

```cpp
ID3D12GraphicsCommandList* commandList;
device->CreateCommandList(0, D3D12_COMMAND_LIST_TYPE_DIRECT, commandAllocator, nullptr, IID_PPV_ARGS(&commandList));

// Record commands
commandList->SetPipelineState(pso);
commandList->SetGraphicsRootSignature(rootSig);
commandList->IASetVertexBuffers(0, 1, &vbView);
commandList->IASetIndexBuffer(&ibView);
commandList->DrawIndexedInstanced(36, 1, 0, 0, 0);

// End recording
commandList->Close();

// Submit to the queue
ID3D12CommandList* lists[] = { commandList };
commandQueue->ExecuteCommandLists(1, lists);
```

### Overall Flow

```
1. Reset Command Allocator (only after GPU is done)
2. Record commands into Command List
3. Close the Command List
4. Submit to Command Queue (ExecuteCommandLists)
5. GPU executes asynchronously
6. Check completion with Fence
7. Reuse Allocator and List

┌───────────────┐    ┌───────────────┐
│  CPU (main)   │    │     GPU       │
│               │    │               │
│ Record ───────┼──→ │               │
│ Submit ───────┼──→ │ Executing...  │
│ (other work)  │    │               │
│ Fence wait ←──┼─── │ Done (Signal) │
└───────────────┘    └───────────────┘
```

After submitting, the CPU continues doing other work. The GPU runs independently.
The CPU only waits at the Fence when it actually needs the result.
