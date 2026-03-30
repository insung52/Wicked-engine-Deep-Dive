# Part 4: Resource Management — Index

Part 3 covered how to send commands to the GPU.
This part covers **how GPU data (resources) is managed** — buffers, heaps, descriptors, textures, and synchronization.

---

## Files

| File | Content |
|------|---------|
| [part4_1_buffer_types.md](part4_1_buffer_types.md) | Buffer types (VB, IB, CB, Structured, RW), GPU alignment |
| [part4_2_heap_memory.md](part4_2_heap_memory.md) | Heap types, upload process, Committed/Placed/Aliasing, Ring Buffer |
| [part4_3_descriptors.md](part4_3_descriptors.md) | Descriptor types, descriptor heaps, CPU/GPU handles, Bindless rendering |
| [part4_4_resource_states.md](part4_4_resource_states.md) | Resource states, barriers, state tracker, Enhanced Barrier |
| [part4_5_textures.md](part4_5_textures.md) | Texture types, mipmaps, compression formats, Render Target, Depth buffer |
| [part4_6_indirect_draw.md](part4_6_indirect_draw.md) | ExecuteIndirect, CommandSignature, argument buffer, initialization requirement |

---

## Reading Order

1. `part4_1` — understand what kinds of buffers exist and how they are used
2. `part4_2` — understand where GPU memory lives and how data gets uploaded
3. `part4_3` — understand how shaders access resources (descriptors, bindless)
4. `part4_4` — understand why resource state transitions (barriers) are required
5. `part4_5` — texture-specific details (mip levels, compression, render targets)
6. `part4_6` — indirect draw: letting the GPU decide draw arguments at runtime

---

## Prerequisites

- [Part 3: Commands and Synchronization](../part3/part3_index.md)
- [Part 3-3: Root Signature](../part3/part3_3_root_signature.md) — how resources are bound to shaders
