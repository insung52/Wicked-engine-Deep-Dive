# 커밋 #24: 2e823bb2 - region copy support for CPU textures

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `2e823bb2` |
| 날짜 | 2026-01-18 |
| 작성자 | Turánszki János |
| 카테고리 | 기능 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphics.h | `GetPlaneSlice()`, `ComputeSubresource()` 헬퍼 함수 추가 |
| wiGraphicsDevice_DX12.cpp | `CopyTexture()` 4가지 usage 시나리오 처리 |

---

## 배경 지식: DX12 텍스처 복사와 Footprint

### GPU 텍스처 vs CPU 텍스처

```
┌─────────────────────────────────────────────────────────────────┐
│              DX12 텍스처 메모리 레이아웃                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GPU 텍스처 (Usage::DEFAULT):                                   │
│  ─────────────────────────────                                  │
│  - 실제 텍스처 리소스 (ID3D12Resource with TEXTURE dimension)   │
│  - GPU 메모리에 최적화된 레이아웃 (타일링, 압축 등)            │
│  - Subresource index로 mip/slice 접근                          │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  GPU Texture (Optimized Layout)                        │     │
│  │  ┌─────────────────┬─────────┬───────┬─────┐           │     │
│  │  │   Mip 0         │  Mip 1  │ Mip 2 │Mip3 │           │     │
│  │  │   (타일링됨)    │         │       │     │           │     │
│  │  └─────────────────┴─────────┴───────┴─────┘           │     │
│  │  → Subresource 0     → Sub 1   → Sub 2  → Sub 3        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  CPU 텍스처 (Usage::UPLOAD / READBACK):                         │
│  ───────────────────────────────────────                        │
│  - 실제로는 버퍼! (ID3D12Resource with BUFFER dimension)        │
│  - 선형 메모리 레이아웃                                         │
│  - Footprint으로 각 subresource 위치 지정                      │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  CPU Texture = Linear Buffer                           │     │
│  │  ┌─────────────────┬─────────┬───────┬─────┬─────────┐ │     │
│  │  │ Footprint[0]    │ FP[1]   │ FP[2] │FP[3]│ Padding │ │     │
│  │  │ Offset: 0       │ Off: X  │Off: Y │Off:Z│         │ │     │
│  │  │ RowPitch: 256   │ RP: 128 │RP: 64 │RP:32│         │ │     │
│  │  └─────────────────┴─────────┴───────┴─────┴─────────┘ │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Footprint 구조

```cpp
// D3D12_PLACED_SUBRESOURCE_FOOTPRINT
struct D3D12_PLACED_SUBRESOURCE_FOOTPRINT
{
    UINT64 Offset;      // 버퍼 내 시작 위치
    D3D12_SUBRESOURCE_FOOTPRINT Footprint;
    {
        DXGI_FORMAT Format;   // 포맷
        UINT Width;           // 너비
        UINT Height;          // 높이
        UINT Depth;           // 깊이
        UINT RowPitch;        // 행 간격 (정렬된 값)
    }
};
```

```
┌─────────────────────────────────────────────────────────────────┐
│              Footprint 개념                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  예: 4x4 텍스처, RGBA8 포맷 (픽셀당 4바이트)                    │
│                                                                 │
│  논리적 데이터:              실제 메모리 (RowPitch=256 정렬):   │
│  ┌────────────────┐          ┌────────────────┬─────────────┐   │
│  │ R R R R        │          │ R R R R        │   Padding   │   │
│  │ G G G G        │    →     │ G G G G        │   Padding   │   │
│  │ B B B B        │          │ B B B B        │   Padding   │   │
│  │ A A A A        │          │ A A A A        │   Padding   │   │
│  └────────────────┘          └────────────────┴─────────────┘   │
│   16 bytes/row                256 bytes/row (DX12 요구사항)     │
│                                                                 │
│  RowPitch = 256 (DX12 텍스처 row는 256바이트 정렬 필요)         │
│  실제 데이터 = 16바이트, 나머지 = 패딩                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 문제: CopyTexture가 CPU 텍스처 위치를 잘못 지정

### 기존 코드 (변경 전)

```cpp
// 변경 전 - 모든 경우에 subresource index만 사용
void GraphicsDevice_DX12::CopyTexture(...)
{
    UINT srcPlane = src_aspect == ImageAspect::STENCIL ? 1 : 0;
    UINT dstPlane = dst_aspect == ImageAspect::STENCIL ? 1 : 0;

    const UINT src_subresource = D3D12CalcSubresource(srcMip, srcSlice, srcPlane, ...);
    const UINT dst_subresource = D3D12CalcSubresource(dstMip, dstSlice, dstPlane, ...);

    // ❌ 모든 경우에 subresource index 사용
    CD3DX12_TEXTURE_COPY_LOCATION src_location(src_resource, src_subresource);
    CD3DX12_TEXTURE_COPY_LOCATION dst_location(dst_resource, dst_subresource);

    commandList->CopyTextureRegion(&dst_location, ..., &src_location, ...);
}
```

### 문제점

```
┌─────────────────────────────────────────────────────────────────┐
│              CPU 텍스처 복사 문제                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  상황: UPLOAD 텍스처 (버퍼) → DEFAULT 텍스처 복사               │
│                                                                 │
│  기존 코드:                                                     │
│  CD3DX12_TEXTURE_COPY_LOCATION src(upload_buffer, subresource); │
│                                                                 │
│  문제:                                                          │
│  - UPLOAD "텍스처"는 실제로 버퍼                                │
│  - 버퍼에는 subresource 개념이 없음!                            │
│  - subresource index로 위치를 지정할 수 없음                    │
│                                                                 │
│  올바른 방법:                                                   │
│  CD3DX12_TEXTURE_COPY_LOCATION src(buffer, footprint);          │
│  - Footprint으로 버퍼 내 정확한 위치(Offset)와 레이아웃 지정    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 해결: 4가지 Usage 시나리오 처리

### 변경된 코드 (변경 후)

```cpp
void GraphicsDevice_DX12::CopyTexture(...)
{
    const TextureDesc& src_desc = src->GetDesc();
    const TextureDesc& dst_desc = dst->GetDesc();

    // ✅ GetPlaneSlice 사용 (더 많은 aspect 처리)
    const UINT srcPlane = GetPlaneSlice(src_aspect);
    const UINT dstPlane = GetPlaneSlice(dst_aspect);
    const UINT src_subresource = D3D12CalcSubresource(...);
    const UINT dst_subresource = D3D12CalcSubresource(...);

    CD3DX12_TEXTURE_COPY_LOCATION src_location;
    CD3DX12_TEXTURE_COPY_LOCATION dst_location;

    // ✅ 4가지 시나리오별 처리
    if (src_desc.usage == Usage::UPLOAD && dst_desc.usage == Usage::DEFAULT)
    {
        // 시나리오 1: CPU → GPU (업로드)
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_resource, footprints[src_subresource]);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_resource, dst_subresource);
    }
    else if (src_desc.usage == Usage::DEFAULT && dst_desc.usage == Usage::READBACK)
    {
        // 시나리오 2: GPU → CPU (리드백)
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_resource, src_subresource);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_resource, footprints[dst_subresource]);
    }
    else if (src_desc.usage == Usage::DEFAULT && dst_desc.usage == Usage::DEFAULT)
    {
        // 시나리오 3: GPU → GPU
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_resource, src_subresource);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_resource, dst_subresource);
    }
    else
    {
        // 시나리오 4: CPU → CPU
        src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_resource, footprints[src_subresource]);
        dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_resource, footprints[dst_subresource]);
    }

    commandList->CopyTextureRegion(&dst_location, dstX, dstY, dstZ, &src_location, srcbox);
}
```

---

## 4가지 복사 시나리오

```
┌─────────────────────────────────────────────────────────────────┐
│              텍스처 복사 시나리오                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  시나리오 1: UPLOAD → DEFAULT (업로드)                          │
│  ──────────────────────────────────────                         │
│  ┌─────────────────┐        ┌─────────────────┐                 │
│  │ CPU Buffer      │   →    │ GPU Texture     │                 │
│  │ (UPLOAD)        │        │ (DEFAULT)       │                 │
│  │ [Footprint]     │        │ [Subresource]   │                 │
│  └─────────────────┘        └─────────────────┘                 │
│  src: footprints[i]         dst: subresource                    │
│                                                                 │
│  용도: 텍스처 데이터 GPU로 전송                                 │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  시나리오 2: DEFAULT → READBACK (리드백)                        │
│  ───────────────────────────────────────                        │
│  ┌─────────────────┐        ┌─────────────────┐                 │
│  │ GPU Texture     │   →    │ CPU Buffer      │                 │
│  │ (DEFAULT)       │        │ (READBACK)      │                 │
│  │ [Subresource]   │        │ [Footprint]     │                 │
│  └─────────────────┘        └─────────────────┘                 │
│  src: subresource           dst: footprints[i]                  │
│                                                                 │
│  용도: GPU 렌더 결과 CPU로 읽기 (스크린샷, 디버깅)              │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  시나리오 3: DEFAULT → DEFAULT (GPU 간 복사)                    │
│  ──────────────────────────────────────────                     │
│  ┌─────────────────┐        ┌─────────────────┐                 │
│  │ GPU Texture     │   →    │ GPU Texture     │                 │
│  │ (DEFAULT)       │        │ (DEFAULT)       │                 │
│  │ [Subresource]   │        │ [Subresource]   │                 │
│  └─────────────────┘        └─────────────────┘                 │
│  src: subresource           dst: subresource                    │
│                                                                 │
│  용도: mipmap 생성, 텍스처 복제 등                              │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  시나리오 4: UPLOAD/READBACK → UPLOAD/READBACK (CPU 간 복사)    │
│  ─────────────────────────────────────────────────────────────  │
│  ┌─────────────────┐        ┌─────────────────┐                 │
│  │ CPU Buffer      │   →    │ CPU Buffer      │                 │
│  │ [Footprint]     │        │ [Footprint]     │                 │
│  └─────────────────┘        └─────────────────┘                 │
│  src: footprints[i]         dst: footprints[i]                  │
│                                                                 │
│  용도: 스테이징 버퍼 간 복사 (드문 케이스)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 헬퍼 함수: GetPlaneSlice

### 기존 코드

```cpp
// 변경 전 - STENCIL만 처리
UINT srcPlane = src_aspect == ImageAspect::STENCIL ? 1 : 0;
```

### 새로운 헬퍼 함수

```cpp
// 변경 후 - 모든 aspect 처리
constexpr uint32_t GetPlaneSlice(ImageAspect aspect)
{
    switch (aspect)
    {
    case ImageAspect::COLOR:
    case ImageAspect::DEPTH:
    case ImageAspect::LUMINANCE:
        return 0;          // 첫 번째 plane
    case ImageAspect::STENCIL:
    case ImageAspect::CHROMINANCE:
        return 1;          // 두 번째 plane
    default:
        break;
    }
    return 0;
}
```

### Plane 개념

```
┌─────────────────────────────────────────────────────────────────┐
│              Multi-Plane 포맷                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Depth-Stencil 텍스처 (D24_UNORM_S8_UINT):                      │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Plane 0: Depth (24-bit)                                   │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Plane 1: Stencil (8-bit)                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  YUV 텍스처 (NV12):                                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Plane 0: Y (Luminance)                                    │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │ Plane 1: UV (Chrominance)                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  복사 시 aspect로 어떤 plane을 복사할지 지정                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 헬퍼 함수: ComputeSubresource

```cpp
// Subresource index 계산
constexpr uint32_t ComputeSubresource(
    uint32_t mip,
    uint32_t slice,
    uint32_t plane,
    uint32_t mip_count,
    uint32_t array_size)
{
    return mip + slice * mip_count + plane * mip_count * array_size;
}

// ImageAspect 버전 (GetPlaneSlice 자동 호출)
constexpr uint32_t ComputeSubresource(
    uint32_t mip,
    uint32_t slice,
    ImageAspect aspect,
    uint32_t mip_count,
    uint32_t array_size)
{
    return ComputeSubresource(mip, slice, GetPlaneSlice(aspect), mip_count, array_size);
}
```

### Subresource 인덱싱

```
┌─────────────────────────────────────────────────────────────────┐
│              Subresource 인덱싱 공식                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  index = mip + slice * mip_count + plane * mip_count * array    │
│                                                                 │
│  예: 4 mips, 3 array slices, 2 planes                           │
│                                                                 │
│  Plane 0:                                                       │
│  ┌───────┬───────┬───────┬───────┐                              │
│  │ S0M0  │ S0M1  │ S0M2  │ S0M3  │  Array Slice 0               │
│  │   0   │   1   │   2   │   3   │                              │
│  ├───────┼───────┼───────┼───────┤                              │
│  │ S1M0  │ S1M1  │ S1M2  │ S1M3  │  Array Slice 1               │
│  │   4   │   5   │   6   │   7   │                              │
│  ├───────┼───────┼───────┼───────┤                              │
│  │ S2M0  │ S2M1  │ S2M2  │ S2M3  │  Array Slice 2               │
│  │   8   │   9   │  10   │  11   │                              │
│  └───────┴───────┴───────┴───────┘                              │
│                                                                 │
│  Plane 1: (indices 12-23)                                       │
│  ┌───────┬───────┬───────┬───────┐                              │
│  │  12   │  13   │  14   │  15   │  Array Slice 0               │
│  ├───────┼───────┼───────┼───────┤                              │
│  │  16   │  17   │  18   │  19   │  Array Slice 1               │
│  ├───────┼───────┼───────┼───────┤                              │
│  │  20   │  21   │  22   │  23   │  Array Slice 2               │
│  └───────┴───────┴───────┴───────┘                              │
│                                                                 │
│  예: slice=1, mip=2, plane=0 → 1*4 + 2 + 0*12 = 6               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Footprint 생성 코드

```cpp
// CreateTexture에서 footprint 미리 계산
bool GraphicsDevice_DX12::CreateTexture(...)
{
    // mip_levels 먼저 계산
    texture->desc.mip_levels = GetMipCount(texture->desc);

    // footprint 배열 크기 = subresource 개수
    internal_state->footprints.resize(GetTextureSubresourceCount(texture->desc));
    internal_state->rowSizesInBytes.resize(internal_state->footprints.size());
    internal_state->numRows.resize(internal_state->footprints.size());

    // DX12 API로 footprint 계산
    device->GetCopyableFootprints(
        &resourcedesc,
        0,                                      // FirstSubresource
        (UINT)internal_state->footprints.size(), // NumSubresources
        0,                                      // BaseOffset
        internal_state->footprints.data(),      // 출력: footprint 배열
        internal_state->numRows.data(),         // 출력: 행 수
        internal_state->rowSizesInBytes.data(), // 출력: 행 크기
        &internal_state->total_size             // 출력: 총 크기
    );
}
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GBackend.h | `GetPlaneSlice()` 헬퍼 함수 추가 (라인 1918) |
| GBackend.h | `ComputeSubresource()` 두 가지 오버로드 추가 (라인 1936, 1942) |
| GraphicsDevice_DX12.cpp | `CopyTexture()` 4가지 usage 시나리오 처리 (라인 7086-7109) |
| GraphicsDevice_DX12.cpp | `GetPlaneSlice()` 사용 (라인 7079-7080) |

### 코드 위치

```cpp
// GBackend.h (라인 1918-1945)
constexpr uint32_t GetPlaneSlice(ImageAspect aspect)
{
    switch (aspect)
    {
    case ImageAspect::COLOR:
    case ImageAspect::DEPTH:
    case ImageAspect::LUMINANCE:
        return 0;
    case ImageAspect::STENCIL:
    case ImageAspect::CHROMINANCE:
        return 1;
    default:
        break;
    }
    return 0;
}

constexpr uint32_t ComputeSubresource(uint32_t mip, uint32_t slice, uint32_t plane,
    uint32_t mip_count, uint32_t array_size)
{
    return mip + slice * mip_count + plane * mip_count * array_size;
}

// GraphicsDevice_DX12.cpp (라인 7086-7109)
if (src_desc.usage == Usage::UPLOAD && dst_desc.usage == Usage::DEFAULT)
{
    src_location = CD3DX12_TEXTURE_COPY_LOCATION(src_resource, src_internal->footprints[src_subresource]);
    dst_location = CD3DX12_TEXTURE_COPY_LOCATION(dst_resource, dst_subresource);
}
// ... 나머지 3가지 시나리오 ...
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 목적 | CPU 텍스처(UPLOAD/READBACK) 복사 시 footprint 기반 위치 지정 |
| 문제 | 기존 코드는 모든 경우에 subresource index만 사용 → CPU 텍스처 복사 오류 |
| 해결 | 4가지 usage 시나리오별로 올바른 copy location 타입 사용 |
| 헬퍼 | `GetPlaneSlice()`, `ComputeSubresource()` 추가 |
| VizMotive | ✅ 적용 완료 |

### 핵심 교훈

> **CPU 텍스처는 버퍼다**
>
> DX12에서 UPLOAD/READBACK "텍스처"는 실제로 버퍼:
> - GPU 텍스처 (DEFAULT): subresource index로 위치 지정
> - CPU 텍스처: footprint으로 버퍼 내 위치+레이아웃 지정
>
> 복사 시 양쪽의 usage를 확인하고 올바른 copy location 사용!

> **Footprint = 버퍼 내 텍스처 레이아웃 정보**
>
> - `Offset`: 버퍼 내 시작 위치
> - `RowPitch`: 행 간격 (256바이트 정렬)
> - `Width/Height/Depth`: 실제 크기
>
> `GetCopyableFootprints()`로 DX12가 자동 계산해줌.

> **Multi-Plane 포맷 고려**
>
> Depth-Stencil, YUV 등 multi-plane 포맷:
> - DEPTH → plane 0
> - STENCIL → plane 1
> - `GetPlaneSlice()`로 aspect에서 plane 인덱스 추출

