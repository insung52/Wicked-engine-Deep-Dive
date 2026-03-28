# Part 6-4: Full Draw Call Flow and Structure Summary

> Index: [Part 6 Index](README.md)
> Previous: [Part 6-3](part6_3_descriptor_binder.md) — DescriptorBinder

---

## Full Draw Call Flow

This is what happens from engine code to GPU execution for a single draw call.

```
CPU (engine code)                                  GPU
─────────────────────────────────────────────────────────────────

BindPipelineState(pso)
  → Static PSO path:
      if active_pso == pso → return          [skip: already bound]
      SetPipelineState(pso->resource)
      active_pso = pso

  → Dynamic PSO path:
      hash = (pso, renderpass_hash)
      if prev_hash == hash:
          active_pso = pso → return          [skip: same combination]
      actual_pso = cache[hash] or create new
      SetPipelineState(actual_pso)
      active_pso = pso
      prev_hash = hash

BindResource(&texture, t0, cmd)
  → table.SRV[0] = &texture               [record only, no DX12 call]

BindConstantBuffer(&cb, b1, cmd, offset)
  → table.CBV[1] = &cb                    [record only]
  → table.CBV_offset[1] = offset

DrawInstanced(...)
  → flush() called internally:
      optimizer = active_pso->rootsig_optimizer
      for each root parameter:
          CBV b1:  gpu_address + CBV_offset[1]           → SetRootCBV(slot, addr)
          SRV t0:  gpu_address + subresource.buffer_offset → SetRootSRV(slot, addr)
                                                                    ↓
                                                         (recorded into command list)
                                                                    ↓
  commandQueue->ExecuteCommandLists(...)                (submitted to GPU at frame end)
                                                                    ↓
                                                              GPU executes draw
```

---

## VizMotive Structure Summary

```
PipelineState
├── resource (ID3D12PipelineState*)   — nullptr means Dynamic PSO
├── rootSignature (ID3D12RootSignature*)
├── rootsig_deserializer              — owns the rootsig_desc memory
├── rootsig_desc                      — pointer into deserializer memory
└── rootsig_optimizer                 — precomputed register→slot mapping, used by flush()

CommandList_DX12
├── active_pso          — currently bound PSO (gives flush() the right optimizer)
├── active_cs           — currently bound Compute Shader
├── prev_pipeline_hash  — prevents redundant Dynamic PSO binds
└── binder              — DescriptorBinder (resource table + flush logic)

SingleDescriptor  (for buffer SRV / UAV subresources)
├── handle          — CPU descriptor heap handle
├── index           — bindless index
└── buffer_offset   — byte offset added to gpu_address when binding via Root Descriptor
```

---

## Key Invariants

These must hold true at every draw call for correct rendering:

1. **`active_pso` matches the GPU-bound PSO** — if they diverge, `flush()` uses the wrong `rootsig_optimizer` and binds to wrong slots
2. **All recorded resources are compatible with the active Root Signature** — binding a resource to a register that the current Root Signature does not declare is silently ignored or causes a validation error
3. **Root Descriptor binds have the correct `buffer_offset` applied** — forgetting the offset causes the shader to read from the wrong position in the buffer
