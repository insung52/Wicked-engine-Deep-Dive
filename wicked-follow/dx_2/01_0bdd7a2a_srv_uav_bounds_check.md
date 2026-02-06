# 커밋 #1: 0bdd7a2a - Fix out of bounds crash in SRV and UAV subresource vectors

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `0bdd7a2a` |
| 날짜 | 2025-10-22 |
| 작성자 | Turánszki János |
| 카테고리 | 버그수정 / 안정성 |
| 우선순위 | 높음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | `GetDescriptorIndex()`에서 subresource 벡터 범위 검사 추가 |

---

## 배경 지식: Subresource란?

### 텍스처의 Subresource

DX12에서 텍스처는 여러 **subresource**로 구성됩니다. Subresource는 mip level과 array slice의 조합입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Texture2DArray 예시                           │
│                    (array_size=3, mip_levels=4)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Array Slice 0:                                                 │
│  ┌────────┬────────┬────────┬────────┐                          │
│  │ Mip 0  │ Mip 1  │ Mip 2  │ Mip 3  │                          │
│  │ 256x256│ 128x128│ 64x64  │ 32x32  │                          │
│  │ [0]    │ [1]    │ [2]    │ [3]    │ ← Subresource Index      │
│  └────────┴────────┴────────┴────────┘                          │
│                                                                 │
│  Array Slice 1:                                                 │
│  ┌────────┬────────┬────────┬────────┐                          │
│  │ Mip 0  │ Mip 1  │ Mip 2  │ Mip 3  │                          │
│  │ [4]    │ [5]    │ [6]    │ [7]    │ ← Subresource Index      │
│  └────────┴────────┴────────┴────────┘                          │
│                                                                 │
│  Array Slice 2:                                                 │
│  ┌────────┬────────┬────────┬────────┐                          │
│  │ Mip 0  │ Mip 1  │ Mip 2  │ Mip 3  │                          │
│  │ [8]    │ [9]    │ [10]   │ [11]   │ ← Subresource Index      │
│  └────────┴────────┴────────┴────────┘                          │
│                                                                 │
│  총 Subresource 수 = array_size × mip_levels = 3 × 4 = 12       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Subresource Index 계산

```cpp
// DX12 공식
subresource_index = mip + (slice * mip_count) + (plane * mip_count * array_size)

// 예: Array Slice 1, Mip 2인 경우
// subresource = 2 + (1 * 4) + (0 * 4 * 3) = 6
```

---

## 배경 지식: SRV/UAV Subresource View

### 개별 Subresource에 대한 View 생성

텍스처 전체가 아닌 **특정 mip level이나 array slice만** 셰이더에서 접근하고 싶을 때, 해당 subresource에 대한 별도 View를 생성합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Subresource View 예시                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Texture 전체 (mip 4개, array 3개)                               │
│  ┌────┬────┬────┬────┐                                          │
│  │ M0 │ M1 │ M2 │ M3 │ Slice 0                                  │
│  ├────┼────┼────┼────┤                                          │
│  │ M0 │ M1 │ M2 │ M3 │ Slice 1                                  │
│  ├────┼────┼────┼────┤                                          │
│  │ M0 │ M1 │ M2 │ M3 │ Slice 2                                  │
│  └────┴────┴────┴────┘                                          │
│                                                                 │
│  SRV 전체: 모든 mip, 모든 slice 접근                             │
│  srv (internal_state->srv)                                      │
│                                                                 │
│  SRV Subresource[0]: Mip 0만 접근                                │
│  SRV Subresource[1]: Mip 1만 접근                                │
│  ...                                                            │
│  (internal_state->subresources_srv[] 벡터에 저장)               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### DX12 구조체

```cpp
struct Resource_DX12 {
    // 전체 리소스에 대한 기본 View
    SingleDescriptor srv;   // 전체 SRV
    SingleDescriptor uav;   // 전체 UAV

    // 개별 subresource에 대한 View 벡터
    std::vector<SingleDescriptor> subresources_srv;  // Subresource별 SRV
    std::vector<SingleDescriptor> subresources_uav;  // Subresource별 UAV
    // ...
};
```

---

## 문제 상황

### GetDescriptorIndex() 함수

`GetDescriptorIndex()`는 리소스의 descriptor index를 반환하는 함수입니다.

```cpp
int GetDescriptorIndex(const GPUResource* resource, SubresourceType type, int subresource) const;
```

**파라미터**:
- `resource`: GPU 리소스 (Texture, Buffer 등)
- `type`: SRV 또는 UAV
- `subresource`: -1이면 전체, 0 이상이면 해당 subresource

### 문제: 범위 검사 없음

```cpp
// 변경 전 - 위험한 코드!
int GraphicsDevice_DX12::GetDescriptorIndex(
    const GPUResource* resource,
    SubresourceType type,
    int subresource) const
{
    auto internal_state = to_internal(resource);

    switch (type)
    {
    case SubresourceType::SRV:
        if (subresource < 0)
        {
            return internal_state->srv.index;
        }
        else
        {
            // ⚠️ 범위 검사 없음!
            return internal_state->subresources_srv[subresource].index;
        }
        break;
    // ... UAV도 동일
    }
}
```

### 크래시 시나리오

```
┌─────────────────────────────────────────────────────────────────┐
│                      Out-of-Bounds 크래시                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  상황: Texture가 2개의 mip level만 가짐                          │
│  → subresources_srv.size() = 2  (인덱스 0, 1만 유효)             │
│                                                                 │
│  호출: GetDescriptorIndex(texture, SRV, 5)                       │
│  → subresources_srv[5] 접근 시도                                │
│  → Out-of-bounds! 💥                                            │
│                                                                 │
│  증상:                                                          │
│  - Debug 빌드: assert 또는 exception                            │
│  - Release 빌드: 쓰레기 값 반환 또는 크래시                      │
│  - 간헐적: 메모리 상태에 따라 다른 결과                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 언제 발생하는가?

1. **잘못된 subresource 인덱스 전달**: 사용자 코드 버그
2. **리소스 재생성 후 오래된 인덱스 사용**: 텍스처 mip count 변경 후
3. **CreateSubresource 호출 없이 subresource 접근**: View 생성 안 한 상태
4. **멀티스레드 경쟁 조건**: View 생성 중에 접근

---

## 해결: 범위 검사 추가

### 수정된 코드

```cpp
// 변경 후 - 안전한 코드
int GraphicsDevice_DX12::GetDescriptorIndex(
    const GPUResource* resource,
    SubresourceType type,
    int subresource) const
{
    if (resource == nullptr || !resource->IsValid())
        return -1;

    auto internal_state = to_internal(resource);

    switch (type)
    {
    case SubresourceType::SRV:
        if (subresource < 0)
        {
            return internal_state->srv.index;
        }
        else
        {
            // ✅ 범위 검사 추가!
            if (subresource >= (int)internal_state->subresources_srv.size())
                return -1;
            return internal_state->subresources_srv[subresource].index;
        }
        break;

    case SubresourceType::UAV:
        if (subresource < 0)
        {
            return internal_state->uav.index;
        }
        else
        {
            // ✅ 범위 검사 추가!
            if (subresource >= (int)internal_state->subresources_uav.size())
                return -1;
            return internal_state->subresources_uav[subresource].index;
        }
        break;
    }

    return -1;
}
```

### 수정 핵심

```cpp
// SRV 범위 검사
if (subresource >= (int)internal_state->subresources_srv.size())
    return -1;

// UAV 범위 검사
if (subresource >= (int)internal_state->subresources_uav.size())
    return -1;
```

**-1 반환의 의미**:
- 유효하지 않은 descriptor index
- 호출자가 이를 체크하여 에러 처리 가능
- 크래시 대신 정상적인 에러 경로로 처리

---

## 방어적 프로그래밍 관점

### Before vs After

```
┌─────────────────────────────────────────────────────────────────┐
│                      방어적 프로그래밍                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  변경 전 (공격적):                                               │
│  ─────────────────                                              │
│  "호출자가 올바른 인덱스를 전달할 것이다"                        │
│  → 잘못된 인덱스 = 크래시                                       │
│  → 디버깅 어려움 (어디서 잘못된 인덱스가 왔는지 추적 필요)       │
│                                                                 │
│  변경 후 (방어적):                                               │
│  ─────────────────                                              │
│  "호출자가 잘못된 인덱스를 전달할 수 있다"                       │
│  → 잘못된 인덱스 = -1 반환 (안전한 실패)                        │
│  → 호출자가 -1 체크하여 처리 가능                               │
│  → 크래시 방지, 더 나은 에러 핸들링                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### API 설계 원칙

```cpp
// 좋은 API: 잘못된 입력에 대해 명확한 에러 값 반환
int GetDescriptorIndex(...) {
    if (invalid_input) return -1;  // 명확한 실패 표시
    return valid_index;
}

// 나쁜 API: 잘못된 입력에 대해 undefined behavior
int GetDescriptorIndex(...) {
    return vector[index];  // index가 범위 밖이면 크래시
}
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-28

### 적용 위치

**GraphicsDevice_DX12.cpp:5293-5330**:

```cpp
int GraphicsDevice_DX12::GetDescriptorIndex(const GPUResource* resource, SubresourceType type, int subresource) const
{
    if (resource == nullptr || !resource->IsValid())
        return -1;

    auto internal_state = to_internal(resource);

    switch (type)
    {
    case SubresourceType::SRV:
        if (subresource < 0)
        {
            return internal_state->srv.index;
        }
        else
        {
            if (subresource >= (int)internal_state->subresources_srv.size())
                return -1;  // ✅ 범위 검사
            return internal_state->subresources_srv[subresource].index;
        }
        break;
    case SubresourceType::UAV:
        if (subresource < 0)
        {
            return internal_state->uav.index;
        }
        else
        {
            if (subresource >= (int)internal_state->subresources_uav.size())
                return -1;  // ✅ 범위 검사
            return internal_state->subresources_uav[subresource].index;
        }
        break;
    }

    return -1;
}
```

---

## 추가 고려사항: 다른 접근 위치

### 현재 VizMotive 코드에서 범위 검사 없는 위치

`GetDescriptorIndex()` 외에도 `subresources_srv[]`, `subresources_uav[]`에 직접 접근하는 곳들이 있습니다:

```cpp
// 1. RefreshRootDescriptors (line 1808, 1821)
int descriptor_index = subresource < 0 ? internal_state->srv.index
    : internal_state->subresources_srv[subresource].index;  // ⚠️ 범위 검사 없음

// 2. BindDescriptorTable (line 1913, 1930)
D3D12_CPU_DESCRIPTOR_HANDLE src_handle = subresource < 0 ? internal_state->srv.handle
    : internal_state->subresources_srv[subresource].handle;  // ⚠️ 범위 검사 없음

// 3. Root Descriptor 바인딩 (line 2045, 2081)
address += internal_state->subresources_srv[subresource].buffer_offset;  // ⚠️ 범위 검사 없음

// 4. RenderPassBegin RESOLVE (line 6257)
descriptor = subresource < 0 ? internal_state->srv
    : internal_state->subresources_srv[subresource];  // ⚠️ 범위 검사 없음
```

**이 위치들도 범위 검사가 필요할 수 있음**. 다만:
- 내부 코드이므로 호출자가 제한적
- `GetDescriptorIndex()`는 외부 노출 API이므로 우선 수정

---

## 요약

| 항목 | 내용 |
|------|------|
| 문제 | `GetDescriptorIndex()`에서 subresource 벡터 범위 검사 없음 |
| 증상 | 잘못된 인덱스 전달 시 out-of-bounds 크래시 |
| 해결 | `size()` 비교 후 범위 벗어나면 -1 반환 |
| 효과 | 크래시 대신 안전한 실패, 디버깅 용이 |
| VizMotive | ✅ 적용 완료 (2026-01-28) |

### 핵심 교훈

> **벡터/배열 접근 전에는 항상 범위 검사**
>
> 외부에서 인덱스를 받는 함수는 특히 주의 필요.
> 잘못된 인덱스에 대해 크래시 대신 에러 값(-1, nullptr 등)을 반환하여
> 호출자가 에러를 처리할 수 있게 해야 함.
