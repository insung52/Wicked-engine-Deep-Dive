# 커밋 #25: 24824a1c - texture upload improvements

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `24824a1c` |
| 날짜 | 2026-01-24 |
| 작성자 | Turánszki János |
| 카테고리 | 최적화 / 리팩토링 |
| 우선순위 | 낮음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphics.h | `GetMipCount(TextureDesc)`, `GetTextureSubresourceCount()` 헬퍼 함수 추가 |
| wiGraphicsDevice_DX12.cpp | CreateTexture 코드 정리, mip_levels 계산 시점 변경 |

---

## 배경 지식: mip_levels와 Subresource

### mip_levels = 0의 의미

```
┌─────────────────────────────────────────────────────────────────┐
│              TextureDesc.mip_levels 값의 의미                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  mip_levels = 0:                                                │
│  → "최대 mipmap 개수 자동 계산"                                 │
│  → log2(max(width, height, depth)) + 1                          │
│                                                                 │
│  mip_levels = N (N > 0):                                        │
│  → "정확히 N개의 mipmap 사용"                                   │
│                                                                 │
│  예: 256x256 텍스처                                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ mip_levels = 0 → 자동 계산 = 9 (256→128→64→32→16→8→4→2→1) │  │
│  │ mip_levels = 1 → mip 0만 (256x256)                        │  │
│  │ mip_levels = 4 → mip 0~3 (256, 128, 64, 32)               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Subresource 개수 계산

```
┌─────────────────────────────────────────────────────────────────┐
│              Subresource 개수                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  총 Subresource 수 = array_size × mip_levels                    │
│                                                                 │
│  예: Texture2DArray (3 slices, 4 mips)                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │        Mip0    Mip1    Mip2    Mip3                       │  │
│  │ Slice0 [0]     [1]     [2]     [3]                        │  │
│  │ Slice1 [4]     [5]     [6]     [7]                        │  │
│  │ Slice2 [8]     [9]     [10]    [11]                       │  │
│  │                                                           │  │
│  │ 총 Subresource = 3 × 4 = 12                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  footprints 배열 크기 = Subresource 개수                        │
│  → 각 subresource의 메모리 레이아웃 정보 저장                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 문제: mip_levels 계산 시점과 코드 중복

### 기존 코드의 문제점

```cpp
// 변경 전 - 문제가 있는 코드 흐름
bool GraphicsDevice_DX12::CreateTexture(...)
{
    texture->desc = *desc;

    // 1. 원본 desc->mip_levels 값으로 리소스 생성 시도
    resourcedesc.MipLevels = desc->mip_levels;  // ⚠️ 0일 수 있음!

    // ... 중간에 많은 코드 ...

    // 2. 나중에야 mip_levels 계산
    if (texture->desc.mip_levels == 0)
    {
        texture->desc.mip_levels = GetMipCount(width, height, depth);
    }

    // 3. footprints 크기 계산 - std::max(1u, ...) 필요
    internal_state->footprints.resize(
        desc->array_size * std::max(1u, desc->mip_levels)  // ⚠️ 복잡한 계산
    );
}
```

### 문제점

```
┌─────────────────────────────────────────────────────────────────┐
│              기존 코드의 문제                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. mip_levels 계산 시점이 늦음                                 │
│     - resourcedesc 설정 시 원본 값(0 가능) 사용                 │
│     - 나중에 계산하면 이미 틀린 값으로 리소스 생성됨            │
│                                                                 │
│  2. 코드 중복                                                   │
│     - GetMipCount(width, height, depth) 여러 곳에서 호출        │
│     - std::max(1u, desc->mip_levels) 패턴 반복                  │
│     - array_size * mip_levels 계산 반복                         │
│                                                                 │
│  3. 가독성 저하                                                 │
│     - 의도 파악 어려움                                          │
│     - 수정 시 여러 곳 변경 필요                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 해결: 헬퍼 함수 추가 + 계산 시점 변경

### 새로운 헬퍼 함수

```cpp
// 1. GetMipCount(TextureDesc) - desc에서 mip count 반환
constexpr uint32_t GetMipCount(const TextureDesc& desc)
{
    // mip_levels가 0이면 최대값 계산, 아니면 그대로 반환
    return desc.mip_levels == 0
        ? GetMipCount(desc.width, desc.height, desc.depth)
        : desc.mip_levels;
}

// 2. GetTextureSubresourceCount - 총 subresource 개수
constexpr uint32_t GetTextureSubresourceCount(const TextureDesc& desc)
{
    const uint32_t mips = GetMipCount(desc);
    return desc.array_size * mips;
}
```

### 개선된 CreateTexture

```cpp
// 변경 후 - 깔끔한 코드 흐름
bool GraphicsDevice_DX12::CreateTexture(...)
{
    texture->desc = *desc;

    // ✅ 1. mip_levels 먼저 계산 (0이면 최대값으로)
    texture->desc.mip_levels = GetMipCount(texture->desc);

    // ✅ 2. 계산된 값으로 리소스 생성
    resourcedesc.MipLevels = texture->desc.mip_levels;

    // ... 리소스 생성 코드 ...

    // ✅ 3. 헬퍼 함수로 간단하게 크기 계산
    internal_state->footprints.resize(GetTextureSubresourceCount(texture->desc));
}
```

---

## 코드 비교

```
┌─────────────────────────────────────────────────────────────────┐
│                      변경 전                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  // mip_levels 계산이 여기저기 흩어져 있음                      │
│                                                                 │
│  resourcedesc.MipLevels = desc->mip_levels;  // 0일 수 있음!    │
│                                                                 │
│  // ... 많은 코드 후 ...                                        │
│                                                                 │
│  if (texture->desc.mip_levels == 0) {                           │
│      texture->desc.mip_levels = GetMipCount(w, h, d);           │
│  }                                                              │
│                                                                 │
│  footprints.resize(desc->array_size * std::max(1u, mip_levels));│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      변경 후                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  // ✅ 함수 시작 부분에서 한 번만 계산                          │
│  texture->desc.mip_levels = GetMipCount(texture->desc);         │
│                                                                 │
│  // ✅ 이후 모든 곳에서 계산된 값 사용                          │
│  resourcedesc.MipLevels = texture->desc.mip_levels;             │
│                                                                 │
│  // ✅ 헬퍼 함수로 의도 명확                                    │
│  footprints.resize(GetTextureSubresourceCount(texture->desc));  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 헬퍼 함수 사용처

### ComputeTextureMemorySizeInBytes에서도 사용

```cpp
// 텍스처 메모리 크기 계산 함수
constexpr size_t ComputeTextureMemorySizeInBytes(const TextureDesc& desc)
{
    size_t size = 0;
    const uint32_t bytes_per_block = GetFormatStride(desc.format);
    const uint32_t pixels_per_block = GetFormatBlockSize(desc.format);

    // ✅ GetMipCount(desc) 사용
    const uint32_t mips = GetMipCount(desc);

    for (uint32_t layer = 0; layer < desc.array_size; ++layer)
    {
        for (uint32_t mip = 0; mip < mips; ++mip)
        {
            const uint32_t mip_width = std::max(1u, desc.width >> mip);
            const uint32_t mip_height = std::max(1u, desc.height >> mip);
            const uint32_t mip_depth = std::max(1u, desc.depth >> mip);
            const uint32_t num_blocks_x = (mip_width + pixels_per_block - 1) / pixels_per_block;
            const uint32_t num_blocks_y = (mip_height + pixels_per_block - 1) / pixels_per_block;
            size += num_blocks_x * num_blocks_y * mip_depth * bytes_per_block;
        }
    }
    size *= desc.sample_count;
    return size;
}
```

---

## 다이어그램: 개선 효과

```
┌─────────────────────────────────────────────────────────────────┐
│              코드 개선 효과                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 단일 책임 원칙 (Single Responsibility)                      │
│     ───────────────────────────────────                         │
│     GetMipCount(desc):         mip 개수 계산 담당               │
│     GetTextureSubresourceCount: subresource 개수 계산 담당      │
│                                                                 │
│  2. 코드 중복 제거 (DRY - Don't Repeat Yourself)                │
│     ────────────────────────────────────────                    │
│     변경 전: mip 계산 로직이 5군데 이상에 흩어져 있음           │
│     변경 후: 헬퍼 함수 한 곳에서 정의, 호출만                   │
│                                                                 │
│  3. 버그 방지                                                   │
│     ──────────                                                  │
│     변경 전: mip_levels=0 처리를 잊으면 버그                    │
│     변경 후: GetMipCount가 항상 유효한 값 반환                  │
│                                                                 │
│  4. 가독성 향상                                                 │
│     ─────────────                                               │
│     변경 전: std::max(1u, desc->mip_levels) - 의도 불명확       │
│     변경 후: GetMipCount(desc) - 의도 명확                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## WickedEngine 추가 변경 (VizMotive 미적용)

### CreateTextureSubresourceDatas alignment 파라미터

WickedEngine에는 `CreateTextureSubresourceDatas()` 함수에 alignment 파라미터가 추가됨:

```cpp
// WickedEngine 전용 (VizMotive에 이 함수 없음)
inline void CreateTextureSubresourceDatas(
    const TextureDesc& desc,
    void* data_ptr,
    wi::vector<SubresourceData>& subresource_datas,
    uint32_t alignment = 1)  // ✅ 새 파라미터
{
    // ...
    // GPU별 row pitch 정렬
    subresource_data.row_pitch = align(
        (uint32_t)num_blocks_x * bytes_per_block,
        alignment  // ✅ 정렬 적용
    );
    // ...
}
```

VizMotive에는 이 함수가 없으므로 스킵.

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GBackend.h | `GetMipCount(const TextureDesc&)` 오버로드 추가 (라인 1912) |
| GBackend.h | `GetTextureSubresourceCount(const TextureDesc&)` 추가 (라인 1948) |
| GBackend.h | `ComputeTextureMemorySizeInBytes()`에서 `GetMipCount(desc)` 사용 (라인 1961) |
| GraphicsDevice_DX12.cpp | mip_levels 계산을 resource 생성 전으로 이동 (라인 3578) |
| GraphicsDevice_DX12.cpp | `footprints.resize()`에서 `GetTextureSubresourceCount()` 사용 (라인 3667) |

### 코드 위치

```cpp
// GBackend.h (라인 1912-1915)
constexpr uint32_t GetMipCount(const TextureDesc& desc)
{
    return desc.mip_levels == 0 ? GetMipCount(desc.width, desc.height, desc.depth) : desc.mip_levels;
}

// GBackend.h (라인 1948-1952)
constexpr uint32_t GetTextureSubresourceCount(const TextureDesc& desc)
{
    const uint32_t mips = GetMipCount(desc);
    return desc.array_size * mips;
}

// GraphicsDevice_DX12.cpp (라인 3578)
texture->desc.mip_levels = GetMipCount(texture->desc);

// GraphicsDevice_DX12.cpp (라인 3589)
resourcedesc.MipLevels = texture->desc.mip_levels;

// GraphicsDevice_DX12.cpp (라인 3667)
internal_state->footprints.resize(GetTextureSubresourceCount(texture->desc));
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 목적 | 텍스처 관련 헬퍼 함수 추가 + CreateTexture 코드 정리 |
| 변경 1 | `GetMipCount(TextureDesc)` - mip_levels=0 자동 처리 |
| 변경 2 | `GetTextureSubresourceCount()` - 총 subresource 개수 |
| 변경 3 | mip_levels 계산을 resource 생성 전으로 이동 |
| 효과 | 코드 중복 제거, 가독성 향상, 버그 방지 |
| VizMotive | ✅ 적용 완료 |

### 핵심 교훈

> **mip_levels=0은 "자동 계산"을 의미**
>
> TextureDesc에서 mip_levels=0은 특별한 의미:
> - "가능한 최대 mipmap 개수 사용"
> - log2(max(width, height, depth)) + 1로 계산
>
> GetMipCount(desc)로 항상 유효한 값을 얻어 사용하면 버그 방지.

> **계산 시점이 중요**
>
> mip_levels 계산은 리소스 생성 전에 먼저 수행해야 함:
> 1. mip_levels 계산 (0이면 최대값으로)
> 2. resourcedesc.MipLevels 설정
> 3. 리소스 생성
>
> 순서가 잘못되면 잘못된 mip 개수로 리소스 생성됨.

> **헬퍼 함수로 의도 명확화**
>
> ```cpp
> // 의도 불명확
> std::max(1u, desc->mip_levels)
>
> // 의도 명확
> GetMipCount(desc)
> GetTextureSubresourceCount(desc)
> ```
>
> 함수명으로 "무엇을 하는지" 바로 알 수 있음.

