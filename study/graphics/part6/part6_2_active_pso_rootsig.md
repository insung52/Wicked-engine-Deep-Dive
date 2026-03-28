# Part 6-2: active_pso and Root Sig Optimizer

> Index: [Part 6 Index](README.md)
> Previous: [Part 6-1](part6_1_pso_static_dynamic.md) — Static vs Dynamic PSO
> Next: [Part 6-3](part6_3_descriptor_binder.md) — DescriptorBinder

---

## active_pso

```cpp
// Members of CommandList_DX12:
const PipelineState* active_pso;  // currently bound PSO
const Shader*        active_cs;   // currently bound Compute Shader
```

`active_pso` is not just a simple optimization variable to avoid redundant `SetPipelineState` calls. Its primary role is to **give `DescriptorBinder::flush()` access to the Root Sig Optimizer of the currently bound PSO**.

```cpp
void DescriptorBinder::flush(bool graphics, CommandList cmd)
{
    auto& commandlist = device->GetCommandList(cmd);

    // Get rootsig_optimizer from active_pso
    auto pso_internal = graphics
        ? to_internal(commandlist.active_pso)   // ← here
        : to_internal(commandlist.active_cs);

    auto& rootsig_optimizer = pso_internal->rootsig_optimizer;  // ← used for binding
    // rootsig_optimizer determines which root parameter index to use for each register
}
```

If `active_pso` does not match the PSO that is actually bound to the GPU, `flush()` uses the **wrong `rootsig_optimizer`** — which means it puts descriptors into the wrong root parameter slots, causing incorrect rendering or a crash.

---

## Root Sig Optimizer

The Root Sig Optimizer parses the Root Signature structure and precomputes a lookup table: **"which shader register (b0, t1, u2, ...) maps to which root parameter index, and how?"**

```
Root Signature:
  Slot 0: Root Constants         (b0)
  Slot 1: Root CBV               (b1)
  Slot 2: Root SRV               (t0)
  Slot 3: Descriptor Table       (t1–t10)

rootsig_optimizer lookup table:
  b0  → slot 0, type = Constants
  b1  → slot 1, type = Root CBV
  t0  → slot 2, type = Root SRV (Root Descriptor)
  t1–t10 → slot 3, type = Descriptor Table
```

At flush time, this table tells the engine exactly which `SetGraphicsRootXxx` call to make for each register:

```cpp
// t1 → slot 3 (Descriptor Table)
commandList->SetGraphicsRootDescriptorTable(3, handle);

// t0 → slot 2 (Root Descriptor)
commandList->SetGraphicsRootShaderResourceView(2, addr);
```

---

## Why active_pso Must Be Correct

When a PSO changes, the Root Signature can change too. The same register (`t0`) might be at a different slot index or bound with a different method (Descriptor Table vs Root Descriptor) in the new PSO.

If `active_pso` is stale:
- `flush()` reads the old PSO's `rootsig_optimizer`
- It writes descriptors to the wrong root parameter slots
- The GPU shader reads from the wrong memory → corrupted output or crash

This is why `BindPipelineState` always updates `active_pso` after changing the GPU PSO, even for Dynamic PSOs where the pointer is the same but the hash differs.
