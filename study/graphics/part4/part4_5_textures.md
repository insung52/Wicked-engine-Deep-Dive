# Part 4-5: Textures

> Index: [Part 4 Index](README.md)
> Previous: [Part 4-4](part4_4_resource_states.md) — Resource States and Barriers
> Next: [Part 4-6](part4_6_indirect_draw.md) — Indirect Draw

---

## Texture vs Buffer

A buffer is a flat array of bytes — no spatial layout. A texture has **dimensions** (width, height, depth/array size, mip levels) and the GPU understands its layout for cache-efficient 2D/3D access and hardware filtering.

---

## Creating a Texture

```cpp
D3D12_RESOURCE_DESC textureDesc = {};
textureDesc.Dimension          = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
textureDesc.Width               = 1024;
textureDesc.Height              = 1024;
textureDesc.DepthOrArraySize    = 1;
textureDesc.MipLevels           = 0;    // 0 = auto-compute all mip levels
textureDesc.Format              = DXGI_FORMAT_R8G8B8A8_UNORM;
textureDesc.SampleDesc.Count    = 1;
textureDesc.Layout              = D3D12_TEXTURE_LAYOUT_UNKNOWN;  // GPU-optimal layout
textureDesc.Flags               = D3D12_RESOURCE_FLAG_NONE;

D3D12_HEAP_PROPERTIES heapProps = { D3D12_HEAP_TYPE_DEFAULT };

device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &textureDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,   // ready to receive upload
    nullptr,
    IID_PPV_ARGS(&texture)
);
```

---

## Texture Types

```cpp
// 1D texture — gradient lookups, tone curves
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE1D;
textureDesc.Height    = 1;

// 2D texture — standard (most common)
textureDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;

// 3D texture — volumetric effects, 3D noise
textureDesc.Dimension        = D3D12_RESOURCE_DIMENSION_TEXTURE3D;
textureDesc.DepthOrArraySize = 64;   // depth slices

// 2D texture array — sprite sheets, terrain tiles
textureDesc.Dimension        = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
textureDesc.DepthOrArraySize = 10;   // array size

// Cube map — environment map, reflections
textureDesc.Dimension        = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
textureDesc.DepthOrArraySize = 6;    // 6 faces
// SRV: use D3D12_SRV_DIMENSION_TEXTURECUBE

// Cube map array
textureDesc.DepthOrArraySize = 6 * numCubemaps;
```

---

## Mipmaps

Mipmaps are pre-downsampled versions of a texture (mip 0 = full size, mip 1 = half size, etc.). The GPU selects the appropriate mip level based on screen-space footprint, reducing aliasing and improving cache efficiency.

```cpp
// Calculate mip count
UINT CalculateMipLevels(UINT width, UINT height) {
    UINT levels = 1;
    while (width > 1 || height > 1) {
        width  = max(1u, width / 2);
        height = max(1u, height / 2);
        levels++;
    }
    return levels;
}

// Upload each mip level separately
for (UINT mip = 0; mip < mipLevels; mip++) {
    D3D12_TEXTURE_COPY_LOCATION src = {};
    src.pResource     = uploadBuffer;
    src.Type          = D3D12_TEXTURE_COPY_TYPE_PLACED_FOOTPRINT;
    src.PlacedFootprint = footprints[mip];   // layout for this mip

    D3D12_TEXTURE_COPY_LOCATION dst = {};
    dst.pResource        = texture;
    dst.Type             = D3D12_TEXTURE_COPY_TYPE_SUBRESOURCE_INDEX;
    dst.SubresourceIndex = mip;    // ← this is the DX12 "subresource" (mip index)

    commandList->CopyTextureRegion(&dst, 0, 0, 0, &src, nullptr);
}
```

> **Note on "subresource":** `SubresourceIndex` here refers to a mip level or array slice of a texture — this is the standard DX12 meaning of the word. This is **different** from the engine's buffer "subresource" concept (a byte range within a large buffer). See [Part 6-5](../part6/part6_5_buffer_subresource.md) for the distinction.

---

## Compression Formats

Compressed textures store less data in VRAM, reducing bandwidth usage. Decompression happens in hardware.

| Format | Ratio | Alpha | Best Use |
|--------|-------|-------|----------|
| BC1 (DXT1) | 6:1 | 1-bit | Opaque color textures |
| BC3 (DXT5) | 4:1 | Smooth | Color + alpha |
| BC4 | 2:1 | N/A | Single channel (AO, heightmaps) |
| BC5 | 2:1 | N/A | 2-channel (normal map XY) |
| BC6H | 6:1 | N/A | HDR color |
| BC7 | 4–8:1 | Smooth | Highest quality general-purpose |

```cpp
DXGI_FORMAT_BC1_UNORM   // BC1
DXGI_FORMAT_BC3_UNORM   // BC3
DXGI_FORMAT_BC4_UNORM   // BC4
DXGI_FORMAT_BC5_UNORM   // BC5
DXGI_FORMAT_BC6H_UF16   // BC6H HDR
DXGI_FORMAT_BC7_UNORM   // BC7
```

---

## Render Target Texture

A texture that the GPU renders into, then reads as a shader resource in a later pass.

```cpp
D3D12_RESOURCE_DESC rtDesc = {};
rtDesc.Dimension         = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
rtDesc.Width             = 1920;
rtDesc.Height            = 1080;
rtDesc.Format            = DXGI_FORMAT_R16G16B16A16_FLOAT;  // HDR
rtDesc.Flags             = D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET;  // required

D3D12_CLEAR_VALUE clearValue = {};
clearValue.Format   = rtDesc.Format;
clearValue.Color[0] = 0.0f;  // optimization hint: expected clear color

device->CreateCommittedResource(
    &defaultHeapProps, D3D12_HEAP_FLAG_NONE,
    &rtDesc,
    D3D12_RESOURCE_STATE_RENDER_TARGET,
    &clearValue,
    IID_PPV_ARGS(&renderTargetTexture)
);

device->CreateRenderTargetView(renderTargetTexture, nullptr, rtvHandle);  // write view
device->CreateShaderResourceView(renderTargetTexture, nullptr, srvHandle); // read view
```

Usage pattern:
```
State: RENDER_TARGET → render into it
Barrier: RENDER_TARGET → PIXEL_SHADER_RESOURCE
State: PIXEL_SHADER_RESOURCE → sample in next pass
```

---

## Depth Buffer Texture

Created as `TYPELESS` so it can be viewed as either a depth format (for DSV writes) or a float format (for SRV reads in shaders like SSAO or shadow comparisons):

```cpp
D3D12_RESOURCE_DESC depthDesc = {};
depthDesc.Format = DXGI_FORMAT_R32_TYPELESS;     // typeless: view decides the format
depthDesc.Flags  = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;

D3D12_CLEAR_VALUE depthClear = {};
depthClear.Format                = DXGI_FORMAT_D32_FLOAT;
depthClear.DepthStencil.Depth    = 1.0f;

device->CreateCommittedResource(..., &depthBuffer);

// DSV for depth writes (pipeline output)
D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc = {};
dsvDesc.Format        = DXGI_FORMAT_D32_FLOAT;
dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2D;
device->CreateDepthStencilView(depthBuffer, &dsvDesc, dsvHandle);

// SRV for depth reads in shaders
D3D12_SHADER_RESOURCE_VIEW_DESC depthSrvDesc = {};
depthSrvDesc.Format              = DXGI_FORMAT_R32_FLOAT;  // read as float
depthSrvDesc.ViewDimension       = D3D12_SRV_DIMENSION_TEXTURE2D;
depthSrvDesc.Texture2D.MipLevels = 1;
device->CreateShaderResourceView(depthBuffer, &depthSrvDesc, depthSrvHandle);
```
