# Part 3: GPU Communication — Index

How the CPU sends commands to the GPU. Covers core DX12 structures and the WickedEngine/VizMotive abstraction layer.

---

## Files

| File | Content |
|------|---------|
| [part3_1_dx12_overview.md](part3_1_dx12_overview.md) | DX12 abbreviations, graphics API overview, core DX12 objects (Device, Queue, Allocator, CommandList) |
| [part3_2_pso.md](part3_2_pso.md) | PSO (Pipeline State Object) — what it is, creation, caching, cost |
| [part3_3_root_signature.md](part3_3_root_signature.md) | Root Signature — shader slot layout, root parameter types, binding |
| [part3_4_command_list.md](part3_4_command_list.md) | Command Buffer/List — recording pattern, multithreading, bundles |
| [part3_5_queues_sync.md](part3_5_queues_sync.md) | Queues (Direct/Compute/Copy), Fence, Semaphore, sync patterns |
| [part3_6_swapchain.md](part3_6_swapchain.md) | SwapChain, Present, VSync, HDR, DXGI_USAGE flags, intermediate RT pattern |
| [part3_7_engine_abstraction.md](part3_7_engine_abstraction.md) | WickedEngine/VizMotive abstraction layer (CommandList handle, SubmitCommandLists) |
| [part3_8_resource_states.md](part3_8_resource_states.md) | Resource states and barriers — why they exist, state types, barrier cost and batching |

---

## Reading Order

If you're new to this:
1. `part3_1` — Get the overall picture of DX12 objects
2. `part3_2` — What a PSO is
3. `part3_3` — How to pass data to shaders (Root Signature)
4. `part3_4` — How commands are recorded and submitted
5. `part3_5` — Queues and synchronization
6. `part3_6` — Screen output (SwapChain)
7. `part3_8` — Resource states and barriers (important!)
8. `part3_7` — VizMotive abstraction (read when you're ready to read engine code)

---

## Quick Reference

**How to bind a texture to a shader?** → [part3_3](part3_3_root_signature.md) Root Signature
**Multithreaded rendering?** → [part3_4](part3_4_command_list.md) Multithreaded recording
**How does the CPU know when the GPU is done?** → [part3_5](part3_5_queues_sync.md) Fence
**Why do we need barriers?** → [part3_8](part3_8_resource_states.md)
**What is `BeginCommandList` in VizMotive code?** → [part3_7](part3_7_engine_abstraction.md)
