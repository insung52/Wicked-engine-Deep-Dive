# Part 6-3: DescriptorBinder and Root Descriptor Offset

> Index: [Part 6 Index](README.md)
> Previous: [Part 6-2](part6_2_active_pso_rootsig.md) — active_pso and Root Sig Optimizer
> Next: [Part 6-4](part6_4_draw_flow.md) — Full Draw Call Flow

---

## DescriptorBinder Pattern

The engine does not call `SetGraphicsRootXxx` directly at the point where you bind a resource. Instead it uses a **Record → Flush** pattern:

1. **Record**: calls like `BindResource()` and `BindConstantBuffer()` only write into an internal table
2. **Flush**: just before a draw call, `flush()` reads the table and issues all the actual DX12 bind calls at once

This separates resource registration from GPU command emission, and allows the engine to batch all binds into a single pass per draw call.

---

## Record Phase

```cpp
// Engine rendering code (e.g., renderer.cpp)
device->BindResource(&texture, 0, cmd);      // register t0
device->BindConstantBuffer(&cb, 1, cmd, 0);  // register b1, byte offset 0
device->DrawInstanced(36, 1, 0, 0, cmd);     // triggers flush internally
```

Internally, these calls only write to the `DescriptorBinder` table:

```cpp
table.SRV[0] = &texture;     // "t0 → texture" (recorded, not yet sent to GPU)
table.CBV[1] = &cb;          // "b1 → cb"
table.CBV_offset[1] = 0;     // "b1 byte offset = 0"
```

No DX12 API calls happen yet.

---

## Flush Phase

Just before the draw call, `flush()` reads the table and issues all the actual root binds:

```cpp
void DescriptorBinder::flush(bool graphics, CommandList cmd)
{
    auto& optimizer = pso_internal->rootsig_optimizer;

    for (auto& param : optimizer.root_parameters)
    {
        switch (param.ParameterType)
        {
        case D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS:
            commandList->SetGraphicsRoot32BitConstants(...);
            break;

        case D3D12_ROOT_PARAMETER_TYPE_CBV:
        {
            uint64_t offset  = table.CBV_offset[param.Descriptor.ShaderRegister];
            auto     address = internal_state->gpu_address + offset;  // ← offset added
            commandList->SetGraphicsRootConstantBufferView(slot, address);
            break;
        }

        case D3D12_ROOT_PARAMETER_TYPE_SRV:
        {
            int subresource = table.SRV_index[param.Descriptor.ShaderRegister];
            auto address    = internal_state->gpu_address;
            if (subresource >= 0)
                address += internal_state->subresources_srv[subresource].buffer_offset;
                // ↑ must manually add byte offset for Root Descriptor
            commandList->SetGraphicsRootShaderResourceView(slot, address);
            break;
        }

        case D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE:
            commandList->SetGraphicsRootDescriptorTable(slot, gpuHandle);
            break;
        }
    }
}
```

---

## Root Descriptor Byte Offset — Why It's Needed

Root Descriptor and Descriptor Table handle buffer offsets differently. See also: [Part 3-3 Root Signature — Byte Offset Warning](../part3/part3_3_root_signature.md).

For what `subresource` and `SingleDescriptor` are, see [Part 6-5: Buffer Subresource and SingleDescriptor](part6_5_buffer_subresource.md).

```
Buffer layout example (1 MB GPUBuffer):
┌────────────────────────────────────────────┐
│  offset=0:     Instance data A (512 KB)    │  ← subresource[0] SRV
│  offset=512KB: Instance data B (512 KB)    │  ← subresource[1] SRV
└────────────────────────────────────────────┘
```

**Descriptor Table SRV** — offset is baked into the descriptor at creation time:
```cpp
srvDesc.Buffer.FirstElement = 512 * 1024 / sizeof(element);  // offset inside descriptor
// GPU reads the descriptor and automatically uses the correct start position
```

**Root Descriptor SRV** — `GetGPUVirtualAddress()` always returns the buffer start address. There is no offset field. The engine must manually add it:
```cpp
// Wrong — reads from byte 0 (Data A), not Data B
commandList->SetGraphicsRootShaderResourceView(slot, buffer->GetGPUVirtualAddress());

// Correct — manually add the subresource's byte offset
auto address = buffer->GetGPUVirtualAddress() + subresource.buffer_offset;
commandList->SetGraphicsRootShaderResourceView(slot, address);
```

**Summary:**

| Method | Where offset lives | Who applies it |
|--------|--------------------|---------------|
| Descriptor Table SRV | `srvDesc.Buffer.FirstElement` | GPU (automatic) |
| Root Descriptor SRV/UAV | Not in the address — must be added | CPU (manually) |
| Root Descriptor CBV | Not in the address — must be added | CPU (manually) |

This is why the `flush()` code has `address += subresources_srv[subresource].buffer_offset` specifically in the Root Descriptor case, but not for Descriptor Table.
