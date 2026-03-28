# Part 6-1: Static PSO vs Dynamic PSO

> Index: [Part 6 Index](README.md)
> Next: [Part 6-2](part6_2_active_pso_rootsig.md) — active_pso and Root Sig Optimizer

---

## Static PSO

A PSO that is compiled before runtime — typically at engine startup or shader build time.

```cpp
// Created at startup
ID3D12PipelineState* compiledPSO;
device->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&compiledPSO));

// Inside PipelineState_DX12:
//   resource = compiledPSO  ← not nullptr
```

- `resource != nullptr` means this is a Static PSO
- Assumes a fixed RTV format, sample count, and render pass configuration
- Fast at draw time — no runtime creation cost

---

## Dynamic PSO

A PSO where `resource == nullptr`. The actual GPU PSO object is created (or retrieved from cache) at the time of the draw call, based on the current render pass configuration.

```cpp
// Inside PipelineState_DX12:
//   resource = nullptr  ← not a Static PSO

// Just before a draw call:
void GraphicsDevice::BindPipelineState(const PipelineState* pso, CommandList cmd)
{
    auto internal_state = to_internal(pso);
    if (internal_state->resource == nullptr)  // Dynamic PSO path
    {
        // Combine current render pass's RTV format, MSAA level, etc.
        // to create or retrieve from cache the actual GPU PSO object
        auto actual_pso = GetOrCreateDynamicPSO(pso, renderpass_info);
        commandList->SetPipelineState(actual_pso.Get());
    }
}
```

- The same `PipelineState*` can produce different GPU PSO objects depending on which render pass it is used in
- Necessary because RTV format and MSAA can differ between render passes

---

## Why Both Coexist

```
Static PSO:
  Pro:  pre-compiled → zero runtime creation cost
  Con:  must know all render pass combinations in advance (often impractical)

Dynamic PSO:
  Pro:  works for any render pass
  Con:  creation cost on first use (subsequent uses hit the cache and are fast)
```

Real engines use Static PSO where the full render pass configuration is known ahead of time, and Dynamic PSO where flexibility is needed.

---

## Pipeline Hash — Duplicate Binding Prevention

Dynamic PSO uses a `(PSO pointer, render pass format)` pair as a cache key for the actual GPU PSO object. At the command list level, the engine also tracks whether the previous draw used the identical combination to avoid redundant binds.

```cpp
struct PipelineHash
{
    const PipelineState* pso;    // PSO pointer
    uint32_t renderpass_hash;    // hash of current render pass format
};

// Member of CommandList_DX12:
PipelineHash prev_pipeline_hash;
```

```
Draw A: PSO=X, renderpass=MRT_A  →  hash(X, MRT_A) — bind PSO, store hash
Draw B: PSO=X, renderpass=MRT_A  →  hash matches    — skip (no redundant bind)
Draw C: PSO=X, renderpass=MRT_B  →  hash differs    — bind new PSO, update hash
```

Static PSO skips the hash check entirely — it uses a simple pointer comparison (`active_pso == pso`) to detect duplicates, which is cheaper.
