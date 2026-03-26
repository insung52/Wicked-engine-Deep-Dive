# Part 3-2: PSO (Pipeline State Object)

> Index: [Part 3 Index](part3_index.md)
> Previous: [Part 3-1](part3_1_dx12_overview.md) — DX12 Core Objects
> Next: [Part 3-3](part3_3_root_signature.md) — Root Signature

---

## What is a PSO?

A PSO is an **immutable object that bundles all pipeline configuration needed for rendering**.

DX11 set pipeline state piece by piece:
```cpp
// DX11 — state is set separately
context->VSSetShader(vertexShader);
context->PSSetShader(pixelShader);
context->RSSetState(rasterizerState);
context->OMSetBlendState(blendState);
context->OMSetDepthStencilState(depthState);
// The driver validates state consistency on every draw call → overhead
```

DX12 compiles all of this into **one PSO object** upfront:
```cpp
// DX12 — all state bundled as one object
commandList->SetPipelineState(pso);
// Already validated at creation time → fast at draw time
```

---

## Graphics PSO Components

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc = {};

// 1. Root Signature (shader resource binding layout)
psoDesc.pRootSignature = rootSignature;

// 2. Shaders (compiled bytecode)
psoDesc.VS = { vsBlob->GetBufferPointer(), vsBlob->GetBufferSize() };
psoDesc.PS = { psBlob->GetBufferPointer(), psBlob->GetBufferSize() };

// 3. Blend state (alpha blending, etc.)
psoDesc.BlendState.RenderTarget[0].BlendEnable = TRUE;
psoDesc.BlendState.RenderTarget[0].SrcBlend    = D3D12_BLEND_SRC_ALPHA;
psoDesc.BlendState.RenderTarget[0].DestBlend   = D3D12_BLEND_INV_SRC_ALPHA;

// 4. Rasterizer state (culling, fill mode, etc.)
psoDesc.RasterizerState.FillMode = D3D12_FILL_MODE_SOLID;
psoDesc.RasterizerState.CullMode = D3D12_CULL_MODE_BACK;

// 5. Depth/stencil state
psoDesc.DepthStencilState.DepthEnable    = TRUE;
psoDesc.DepthStencilState.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ALL;
psoDesc.DepthStencilState.DepthFunc      = D3D12_COMPARISON_FUNC_LESS;

// 6. Input layout (vertex buffer structure)
psoDesc.InputLayout = { inputElements, numElements };

// 7. Render target formats
psoDesc.NumRenderTargets = 1;
psoDesc.RTVFormats[0]    = DXGI_FORMAT_R8G8B8A8_UNORM;
psoDesc.DSVFormat        = DXGI_FORMAT_D32_FLOAT;

// 8. Misc
psoDesc.SampleMask            = UINT_MAX;
psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
psoDesc.SampleDesc.Count      = 1;

// Create the PSO
ID3D12PipelineState* pso;
device->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&pso));
```

---

## Compute PSO

A compute PSO is much simpler — no rasterizer, blend, or depth state needed.

```cpp
D3D12_COMPUTE_PIPELINE_STATE_DESC computePsoDesc = {};
computePsoDesc.pRootSignature = computeRootSig;
computePsoDesc.CS = { csBlob->GetBufferPointer(), csBlob->GetBufferSize() };

ID3D12PipelineState* computePso;
device->CreateComputePipelineState(&computePsoDesc, IID_PPV_ARGS(&computePso));
```

---

## PSO Caching

PSO creation is expensive — it involves shader compilation and full pipeline validation. Creating one at runtime causes a visible freeze.

```cpp
// Option 1: Create all PSOs during a loading screen
void LoadAllPSOs() {
    for (auto& combo : allShaderCombinations) {
        psoCache[combo.hash] = CreatePSO(combo);
    }
}

// Option 2: Binary PSO cache (fast on subsequent runs)
ID3DBlob* cachedBlob;
pso->GetCachedBlob(&cachedBlob);
SaveToFile("pso_cache.bin", cachedBlob);

// Load cache on next run (skips shader compilation → much faster)
psoDesc.CachedPSO = { cachedData, cachedSize };
device->CreateGraphicsPipelineState(&psoDesc, ...);
```

---

## PSO Switch Cost

```
High cost  ←────────────────────────→  Low cost

PSO change  >  Root Signature  >  Descriptor  >  Constant update

→ Group objects that share the same PSO and draw them together (batching)
→ Switching PSO per object is a major performance hit
```

WickedEngine groups objects with the same shader/material into a single render pass to minimize PSO switches. Reducing PSO change frequency is one of the core renderer optimizations.
