# Part 3-6: SwapChain and Present

> Index: [Part 3 Index](part3_index.md)
> Previous: [Part 3-5](part3_5_queues_sync.md) — Queues & Synchronization
> Next: [Part 3-7](part3_7_engine_abstraction.md) — Engine Abstraction

---

## What is a SwapChain?

A SwapChain is a collection of buffers used to display output on screen. The GPU writes to a back buffer while the monitor displays the front buffer. After rendering, the two are swapped.

```
┌───────────────────────────────────────┐
│              Swap Chain                │
│  ┌─────────────┐  ┌─────────────┐     │
│  │  Buffer 0   │  │  Buffer 1   │ ... │
│  │  (front)    │  │  (back)     │     │
│  └──────┬──────┘  └─────────────┘     │
└─────────│─────────────────────────────┘
          │
          ▼
       Monitor
```

---

## Creating a SwapChain

```cpp
DXGI_SWAP_CHAIN_DESC1 swapChainDesc = {};
swapChainDesc.Width       = 1920;
swapChainDesc.Height      = 1080;
swapChainDesc.Format      = DXGI_FORMAT_R8G8B8A8_UNORM;
swapChainDesc.SampleDesc.Count = 1;
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
swapChainDesc.BufferCount = 2;                              // double buffering
swapChainDesc.SwapEffect  = DXGI_SWAP_EFFECT_FLIP_DISCARD;
swapChainDesc.Flags       = DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING;

IDXGISwapChain1* swapChain;
factory->CreateSwapChainForHwnd(
    commandQueue,  // must be a Direct Queue
    hwnd,
    &swapChainDesc,
    nullptr, nullptr,
    &swapChain
);
```

---

## Getting the Back Buffers and Creating RTVs

```cpp
ID3D12Resource* renderTargets[2];

for (int i = 0; i < 2; i++) {
    swapChain->GetBuffer(i, IID_PPV_ARGS(&renderTargets[i]));

    D3D12_CPU_DESCRIPTOR_HANDLE rtvHandle = rtvHeap->GetCPUDescriptorHandleForHeapStart();
    rtvHandle.ptr += i * rtvDescriptorSize;
    device->CreateRenderTargetView(renderTargets[i], nullptr, rtvHandle);
}
```

---

## Present

After rendering, transition the back buffer to `PRESENT` state and call `Present()`.

```cpp
// Transition to PRESENT
commandList->ResourceBarrier(1, &presentBarrier);  // RENDER_TARGET → PRESENT
commandList->Close();
commandQueue->ExecuteCommandLists(1, &cmdList);

// Present
UINT syncInterval = 1;  // VSync: 0=off, 1=60Hz, 2=30Hz
swapChain->Present(syncInterval, presentFlags);

// Update current back buffer index
currentBackBuffer = swapChain->GetCurrentBackBufferIndex();
```

---

## VSync and Tearing

```cpp
// VSync off — lowest latency, allows screen tearing
if (tearingSupported) {
    swapChain->Present(0, DXGI_PRESENT_ALLOW_TEARING);
} else {
    swapChain->Present(1, 0);  // VSync on
}
```

---

## HDR Support

```cpp
// HDR swap chain formats
swapChainDesc.Format = DXGI_FORMAT_R16G16B16A16_FLOAT;  // full HDR
// or
swapChainDesc.Format = DXGI_FORMAT_R10G10B10A2_UNORM;   // HDR10 (10-bit)

// Set color space
swapChain->SetColorSpace1(DXGI_COLOR_SPACE_RGB_FULL_G2084_NONE_P2020);
```

---

## DXGI_USAGE Flags — Declaring Buffer Usage

The `BufferUsage` field in the swap chain descriptor tells DXGI how the buffers will be used.

| Flag | Meaning | Default |
|------|---------|---------|
| `DXGI_USAGE_RENDER_TARGET_OUTPUT` | Use as RTV — GPU can **write** to it | Always required |
| `DXGI_USAGE_SHADER_INPUT` | Use as SRV — shaders can **read** from it | Not set by default |
| `DXGI_USAGE_UNORDERED_ACCESS` | Use as UAV — compute shader read/write | Not set by default |

```cpp
// Typical swap chain (write only)
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;

// Allow shader reading (e.g., post-processing that reads from backbuffer)
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT | DXGI_USAGE_SHADER_INPUT;
```

**Why is RTV-only the default?**
Swap chain buffers are special — DXGI flips them directly to the display. Adding `SHADER_INPUT` requires the driver to do extra work (aliasing prevention, compression handling), which adds a small cost. Only add it when actually needed.

> **Note**: `DXGI_SWAP_EFFECT_FLIP_DISCARD` (the modern DX12 standard) supports `SHADER_INPUT`. The older `SEQUENTIAL` mode may have restrictions depending on platform/driver.

For a detailed analysis of the performance impact and why VizMotive skips this flag, see [appendix_rt_compression.md](../../wicked-follow/appendix/appendix_rt_compression.md).

---

## RTV vs SRV — Two Views of the Same Memory

The same GPU memory can be viewed in different ways depending on how you intend to use it.

```
Back buffer GPU memory (R8G8B8A8_UNORM, 1920×1080)
        │
        ├── RTV (Render Target View)
        │     → GPU writes rendering output here
        │     → Used in RenderPassBegin()
        │
        └── SRV (Shader Resource View)
              → Shader reads this as a texture
              → Used via GetDescriptorIndex() → bindless index
```

**Important**: RTV and SRV cannot be active on the same resource at the same time (Resource State conflict). A barrier must be issued to transition between them.

```
[Render]  RTV state     → GPU writes to back buffer
    ↓ Barrier: RENDER_TARGET → SHADER_RESOURCE
[Read]    SRV state     → shader reads from back buffer (post-processing)
    ↓ Barrier: SHADER_RESOURCE → PRESENT
[Present]
```

---

## Intermediate Render Target Pattern

HDR rendering results typically cannot be written directly to the swap chain. Reasons:

1. **Format mismatch**: rendering uses `R11G11B10_FLOAT` (high precision HDR), swap chain uses `R8G8B8A8_UNORM` (8-bit LDR)
2. **Compute shader needs UAV**: tonemappers often run as compute shaders, which require UAV output — swap chain buffers don't support UAV
3. **Post-processing chaining**: TAA → Bloom → Tonemap → FXAA chain requires intermediate buffers for ping-pong

```
rtMain (R11G11B10_FLOAT, HDR)          ← rendering output
    ↓  compute tonemap (needs UAV → can't write to swap chain directly)
rtPostprocess (R8G8B8A8_UNORM, LDR)   ← intermediate RT
    ↓  image::Draw / blit (pixel shader + RTV)
swapchain (R8G8B8A8_UNORM)            ← final output
```

**Can the intermediate RT be skipped?**
- Yes, if the last pass is a pixel shader that writes directly to the swap chain as an RTV.
- Compute-shader-based passes always need an intermediate RT because they cannot UAV-write to a swap chain buffer.

---

## Resize Handling

```cpp
void OnResize(UINT width, UINT height) {
    // 1. Wait for GPU to finish
    WaitForGPU();

    // 2. Release existing references
    for (int i = 0; i < BUFFER_COUNT; i++) {
        renderTargets[i]->Release();
    }

    // 3. Resize swap chain
    swapChain->ResizeBuffers(
        BUFFER_COUNT,
        width, height,
        DXGI_FORMAT_R8G8B8A8_UNORM,
        DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING
    );

    // 4. Get new buffers and recreate RTVs
    for (int i = 0; i < BUFFER_COUNT; i++) {
        swapChain->GetBuffer(i, IID_PPV_ARGS(&renderTargets[i]));
        // recreate RTVs...
    }

    // 5. Recreate depth buffer
    RecreateDepthBuffer(width, height);

    currentBackBuffer = swapChain->GetCurrentBackBufferIndex();
}
```

**Why release before ResizeBuffers?**
`ResizeBuffers` fails if any references to the existing buffers are still alive. All `ComPtr`s holding buffer references must be released first.
