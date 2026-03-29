# Part 6: PSO Management in an Engine — Index

Part 3 covered the concepts of PSO and Root Signature. This part covers **how a real engine manages them at runtime** — caching, binding tracking, and the full draw call flow.

---

## Files

| File | Content |
|------|---------|
| [part6_1_pso_static_dynamic.md](part6_1_pso_static_dynamic.md) | Static vs Dynamic PSO, Pipeline Hash, duplicate binding prevention |
| [part6_2_active_pso_rootsig.md](part6_2_active_pso_rootsig.md) | `active_pso` tracking, Root Sig Optimizer — how flush() finds the right slot |
| [part6_3_descriptor_binder.md](part6_3_descriptor_binder.md) | DescriptorBinder record→flush pattern, Root Descriptor byte offset |
| [part6_4_draw_flow.md](part6_4_draw_flow.md) | Full draw call flow CPU→GPU, VizMotive structure summary |
| [part6_5_buffer_subresource.md](part6_5_buffer_subresource.md) | Buffer suballocation, subresource index, SingleDescriptor struct |

---

## Reading Order

1. `part6_1` — understand what Static and Dynamic PSO are and why both exist
2. `part6_2` — understand what `active_pso` is used for and why it must be correct
3. `part6_3` — understand how resources get bound to shaders (record → flush)
4. `part6_4` — put it all together as a complete draw call sequence

---

## Prerequisites

- [Part 3-2: PSO](../part3/part3_2_pso.md) — what a PSO is
- [Part 3-3: Root Signature](../part3/part3_3_root_signature.md) — register slots and binding types
