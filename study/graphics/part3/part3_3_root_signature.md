# Part 3-3: Root Signature

> Index: [Part 3 Index](part3_index.md)
> Previous: [Part 3-2](part3_2_pso.md) — PSO
> Next: [Part 3-4](part3_4_command_list.md) — Command List

---

## How Shaders Receive Data

Shader code (HLSL) cannot directly reference CPU buffers or textures. Instead, it receives data through numbered **register slots**.

```hlsl
// HLSL shader code
cbuffer PerFrame      : register(b0) { float4 time; }       // b0: constant buffer
Texture2D albedo      : register(t0);                        // t0: read-only texture
RWBuffer<float> output: register(u0);                        // u0: read/write buffer
SamplerState samp     : register(s0);                        // s0: sampler
```

Register prefixes:

| Prefix | View Type | Purpose |
|--------|-----------|---------|
| `b` | CBV (Constant Buffer View) | Small constant data (matrices, colors, etc.) |
| `t` | SRV (Shader Resource View) | Read-only textures and buffers |
| `u` | UAV (Unordered Access View) | Read/write access (compute output, etc.) |
| `s` | Sampler | Texture sampling mode (filter, wrap, etc.) |

The slot number alone doesn't tell the GPU which actual buffer `b0` refers to. The CPU must connect them at draw time: "b0 = this buffer". This connection is called **binding**.

---

## What is a Root Signature?

A Root Signature is a **layout declaration** that describes which slots a shader uses and how each slot will be filled.

Before a draw call, you tell the GPU: "this shader uses b0, t0–t9, and u0, and here is the format for each."

```
Analogy:
Function declaration = Root Signature   (declares the slot layout)
void render(Texture diffuse, Texture normal, Buffer lights);

Function call = actual binding          (connects real data at draw time)
render(myDiffuse, myNormal, myLights);
```

---

## What is a Root Parameter?

A **Root Parameter** is a **single slot entry** inside a Root Signature.

A Root Signature is an array of Root Parameters. Each has an index (0, 1, 2, ...) and you connect data to that index at draw time.

```
Root Signature
├─ Parameter [0]: b0      → filled via Root Constants
├─ Parameter [1]: b1      → filled via Root Descriptor
└─ Parameter [2]: t0–t9  → filled via Descriptor Table
```

At draw time:
```cpp
commandList->SetGraphicsRoot32BitConstants(0, ...);     // connect data to parameter [0]
commandList->SetGraphicsRootConstantBufferView(1, ...); // connect data to parameter [1]
commandList->SetGraphicsRootDescriptorTable(2, ...);    // connect data to parameter [2]
```

---

## Root Parameter Types

### 1. Root Constants

Fastest option. Inlines small values directly into the command buffer.

```cpp
rootParams[0].ParameterType             = D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS;
rootParams[0].Constants.Num32BitValues  = 4;   // 16 bytes
rootParams[0].Constants.ShaderRegister  = 0;   // b0

commandList->SetGraphicsRoot32BitConstants(0, 4, &data, 0);
```

### 2. Root Descriptor

Passes a raw GPU memory address. Binds one buffer or texture quickly.

```cpp
rootParams[1].ParameterType             = D3D12_ROOT_PARAMETER_TYPE_CBV;
rootParams[1].Descriptor.ShaderRegister = 1;   // b1

commandList->SetGraphicsRootConstantBufferView(1, cbGpuAddress);
```

### 3. Descriptor Table

Binds a range of consecutive slots in a descriptor heap. Can bind many textures at once.

```cpp
D3D12_DESCRIPTOR_RANGE range = {};
range.RangeType          = D3D12_DESCRIPTOR_RANGE_TYPE_SRV;
range.NumDescriptors     = 10;  // 10 textures
range.BaseShaderRegister = 0;   // t0–t9

rootParams[2].ParameterType                       = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
rootParams[2].DescriptorTable.NumDescriptorRanges = 1;
rootParams[2].DescriptorTable.pDescriptorRanges   = &range;

commandList->SetGraphicsRootDescriptorTable(2, gpuHandle);
```

---

## Creating a Root Signature

```cpp
D3D12_ROOT_SIGNATURE_DESC rsDesc = {};
rsDesc.NumParameters     = 3;
rsDesc.pParameters       = rootParams;
rsDesc.NumStaticSamplers = 1;
rsDesc.pStaticSamplers   = &staticSampler;
rsDesc.Flags             = D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT;

// Serialize
ID3DBlob* signatureBlob;
D3D12SerializeRootSignature(&rsDesc, D3D_ROOT_SIGNATURE_VERSION_1, &signatureBlob, nullptr);

// Create
ID3D12RootSignature* rootSignature;
device->CreateRootSignature(0,
    signatureBlob->GetBufferPointer(),
    signatureBlob->GetBufferSize(),
    IID_PPV_ARGS(&rootSignature));
```

---

## HLSL Register Mapping

```hlsl
cbuffer PerFrame  : register(b0) { float4 data; };         // Root Constants or Root CBV
cbuffer PerObject : register(b1) { float4x4 worldMatrix; };// Root CBV
Texture2D textures[10] : register(t0);                      // Descriptor Table (t0–t9)
SamplerState sampler0 : register(s0);                       // Static Sampler
```

---

## Size Limit and Optimization

A Root Signature has a maximum size of **64 DWORDs (256 bytes)**.

```
Cost per parameter type:
- Root Constants:    1 DWORD per 32-bit value
- Root Descriptor:   2 DWORDs (64-bit GPU address)
- Descriptor Table:  1 DWORD

Static Samplers: no cost (separate space)
```

Placement strategy:
```
Example layout (7 DWORDs total = 28 bytes):
┌──────────────────────────────────────────────────────────┐
│ [0] Root Constants    — changes per draw call  (1 DWORD) │
│ [1] Root CBV          — changes per draw call  (2 DWORDs)│
│ [2] Descriptor Table  — changes per material   (1 DWORD) │
│ [3] Root CBV          — changes per frame      (2 DWORDs)│
│ [4] Descriptor Table  — rarely changes         (1 DWORD) │
└──────────────────────────────────────────────────────────┘

→ Place frequently-changing parameters at lower indices (better cache behavior)
→ Small values that change often → Root Constants
→ Large groups of textures → Descriptor Table
```

---

## Root Descriptor: Byte Offset Warning

When binding a specific region of a buffer via Root Descriptor, you must **manually add the byte offset** to the GPU address.

```
GPUBuffer (1 MB):
├─ offset=0:     Data A (512 KB)  ← SRV[0]
└─ offset=512KB: Data B (512 KB)  ← SRV[1]
```

**Descriptor Table SRV** — offset is embedded in the descriptor:
```cpp
srvDesc.Buffer.FirstElement = 512 * 1024 / sizeof(element);
SetGraphicsRootDescriptorTable(slot, gpuHandle);
// GPU reads the descriptor and automatically uses the correct start position
```

**Root Descriptor SRV** — offset must be added manually:
```cpp
// Wrong: reads from the start of the buffer
auto addr = buffer->GetGPUVirtualAddress();
SetGraphicsRootShaderResourceView(slot, addr);

// Correct: manually offset to Data B
auto addr = buffer->GetGPUVirtualAddress() + 512 * 1024;
SetGraphicsRootShaderResourceView(slot, addr);
```

| Method | Where offset is applied | Automatic? |
|--------|------------------------|-----------|
| Descriptor Table SRV | `srvDesc.Buffer.FirstElement` | Yes — GPU handles it |
| Root Descriptor SRV/UAV | Not included — must add manually | No |
| Root Descriptor CBV | Not included — must add manually | No |
