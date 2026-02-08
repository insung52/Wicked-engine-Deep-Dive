# Topic: Texture Operations (텍스처 처리)

## 개요

텍스처 생성, 복사, 관리와 관련된 개선 사항들.

## 관련 커밋

| 순서 | 커밋 | 날짜 | 핵심 변경 |
|------|------|------|----------|
| 1 | [dx1 #17 `6c973af6`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/17_6c973af6_bc_texture_block_calculation.md) | 2025-06-28 | BC 텍스처 블록 계산 수정 |
| 2 | [dx2 #17 `a5162e04`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/17_a5162e04_shader_readable_swapchain.md) | 2025-12-01 | Shader-readable swapchain |
| 3 | [dx2 #20 `bf2f369b`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/20_bf2f369b_delete_subresources_rtv_dsv.md) | 2025-12-31 | DeleteSubresources RTV/DSV 정리 |
| 4 | [dx2 #24 `2e823bb2`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/24_2e823bb2_cpu_texture_region_copy.md) | 2026-01-18 | CPU 텍스처 region copy (footprint) |
| 5 | [dx2 #25 `24824a1c`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/25_24824a1c_texture_upload_improvements.md) | 2026-01-24 | 텍스처 헬퍼 함수 추가 |

---

## 변화 흐름

### 1. BC 텍스처 블록 계산 수정 (dx1 #17)

**문제**: Block Compressed 텍스처의 mip 레벨별 블록 수 계산 오류

```cpp
// 변경 전 (잘못됨)
numBlocksX = baseMipBlocksX >> mip;  // 나머지 픽셀 누락!

// 변경 후 (올바름)
mipWidth = std::max(1u, width >> mip);
numBlocksX = (mipWidth + blockSize - 1) / blockSize;  // ceiling division
```

**핵심 공식**: `num_blocks = (mip_pixels + block_size - 1) / block_size`

**증상**: 텍스처 저장 시 하위 mip 데이터 누락, LOD 전환 시 아티팩트

---

### 2. Shader-readable Swapchain (dx2 #17)

**목적**: Back buffer를 쉐이더에서 직접 읽기 (후처리, 화면 캡처)

**변경**:
```cpp
// SwapChain 생성 시
desc.Usage = DXGI_USAGE_RENDER_TARGET_OUTPUT | DXGI_USAGE_SHADER_INPUT;

// Back buffer 관리
struct SwapChain_DX12 {
    std::vector<Texture_DX12> backbuffers;  // RTV + SRV 모두 포함
};
```

**효과**: `GetBackBuffer()`로 얻은 텍스처를 SRV로 바인딩 가능

---

### 3. DeleteSubresources RTV/DSV 정리 (dx2 #20)

**문제**: `DeleteSubresources()`가 RTV/DSV descriptor를 정리하지 않음

```
┌─────────────────────────────────────────────────────────────────┐
│  Texture 리소스 소유 구조 (변경 전)                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Resource_DX12 (base)                                           │
│  ├─ resource (ID3D12Resource)                                   │
│  ├─ subresources_srv[]   ← destroy_subresources()가 정리       │
│  └─ subresources_uav[]   ← destroy_subresources()가 정리       │
│                                                                 │
│  Texture_DX12 : Resource_DX12                                   │
│  ├─ subresources_rtv[]   ← ❌ 정리 안 됨!                      │
│  └─ subresources_dsv[]   ← ❌ 정리 안 됨!                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**해결**: `Texture_DX12`를 `Resource_DX12`에 병합

```cpp
// 변경 후: 모든 descriptor가 Resource_DX12에 있음
struct Resource_DX12 {
    // ... 기존 멤버 ...
    std::vector<SingleDescriptor> subresources_rtv;
    std::vector<SingleDescriptor> subresources_dsv;
};
```

**증상**: Descriptor leak → 장기 실행 시 heap 고갈

---

### 4. CPU 텍스처 Region Copy (dx2 #24)

**핵심 개념**: CPU 텍스처(UPLOAD/READBACK)는 실제로 **버퍼**

```
GPU 텍스처 (DEFAULT):          CPU 텍스처 (UPLOAD/READBACK):
┌─────────────────────┐        ┌─────────────────────┐
│ ID3D12Resource      │        │ ID3D12Resource      │
│ Dimension: TEXTURE  │        │ Dimension: BUFFER   │ ← 버퍼!
│ Subresource 지원    │        │ Footprint 필요      │
└─────────────────────┘        └─────────────────────┘
```

**4가지 CopyTexture 시나리오**:

| 시나리오 | src | dst | 처리 |
|----------|-----|-----|------|
| UPLOAD → DEFAULT | footprint | subresource | 업로드 |
| DEFAULT → READBACK | subresource | footprint | 리드백 |
| DEFAULT → DEFAULT | subresource | subresource | GPU 간 복사 |
| CPU → CPU | footprint | footprint | 스테이징 |

```cpp
// 올바른 copy location 설정
if (src_internal->desc.usage == Usage::UPLOAD ||
    src_internal->desc.usage == Usage::READBACK) {
    src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_resource, src_footprint);
} else {
    src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_resource, src_subresource);
}
```

**헬퍼 함수**:
- `GetPlaneSlice(Format)`: 다중 평면 포맷(depth/stencil)의 평면 인덱스
- `ComputeSubresource()`: mip + slice + plane → subresource index

---

### 5. 텍스처 헬퍼 함수 (dx2 #25)

**새 함수들**:

```cpp
// mip_levels=0을 자동 계산 최대값으로 처리
uint32_t GetMipCount(const TextureDesc& desc);

// 전체 subresource 개수 계산
uint32_t GetTextureSubresourceCount(const TextureDesc& desc);
```

**개선**: `CreateTexture()`에서 mip_levels 계산을 resource 생성 전으로 이동

```cpp
// 변경 전: resource 생성 후 계산 (복잡)
// 변경 후: resource 생성 전 계산 (명확)
texture->desc.mip_levels = GetMipCount(texture->desc);
// ... 이후 resource 생성 ...
```

---

## 핵심 개념 정리

### Footprint vs Subresource

```
┌─────────────────────────────────────────────────────────────────┐
│  Subresource (GPU 텍스처)                                        │
│  ─────────────────────────                                      │
│  index = mip + (slice × mipLevels) + (plane × mipLevels × slices)│
│  GPU 최적화 레이아웃 (타일링, 압축)                             │
│                                                                 │
│  Footprint (CPU 텍스처 = 버퍼)                                  │
│  ─────────────────────                                          │
│  struct {                                                       │
│      UINT64 Offset;     // 버퍼 내 시작 위치                    │
│      UINT Width, Height, Depth;                                 │
│      UINT RowPitch;     // 256바이트 정렬                       │
│  }                                                              │
│  선형 메모리 레이아웃                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-plane 포맷

```
Depth-Stencil (D24S8):
┌─────────────────────────────────────────┐
│ Plane 0: Depth (24-bit)                 │
│ Plane 1: Stencil (8-bit)                │
└─────────────────────────────────────────┘

Subresource 계산:
Depth 접근:   plane = 0 → ComputeSubresource(mip, slice, 0)
Stencil 접근: plane = 1 → ComputeSubresource(mip, slice, 1)
```

---

## 교훈

### 1. CPU 텍스처 = 버퍼
UPLOAD/READBACK 텍스처는 DX12에서 실제로 버퍼.
복사 시 subresource index가 아닌 footprint 사용 필요.

### 2. Descriptor 누수
Subresource별로 생성한 RTV/DSV도 명시적으로 정리 필요.
구조 설계 시 모든 descriptor가 같은 곳에서 관리되도록.

### 3. BC 텍스처 블록 계산
Ceiling division 사용: `(pixels + blockSize - 1) / blockSize`
단순 shift는 나머지 픽셀을 잃음.

### 4. mip_levels=0 처리
API에서 0은 "자동 계산 최대값" 의미.
내부적으로 실제 값으로 변환하여 저장.

---

## VizMotive 적용

모든 커밋 적용 완료:
- `ComputeTextureMemorySizeInBytes()`: BC 블록 계산 수정
- SwapChain: SRV 지원 추가
- `Resource_DX12`: RTV/DSV 멤버 병합
- `CopyTexture()`: 4가지 시나리오 처리
- `GetMipCount()`, `GetTextureSubresourceCount()` 헬퍼 추가
