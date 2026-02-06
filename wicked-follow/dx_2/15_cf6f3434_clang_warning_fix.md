# 커밋 #15: cf6f3434 - Some clang warning fixes

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `cf6f3434` |
| 날짜 | 2025-11-26 |
| 작성자 | Turánszki János |
| 카테고리 | 빌드 / 코드 품질 |
| 우선순위 | 낮음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | switch문에 `default: break;` 추가 |
| (기타 파일들) | 다양한 clang 경고 수정 |

---

## 배경 지식: Clang 경고

### Clang vs MSVC

```
┌─────────────────────────────────────────────────────────────────┐
│              컴파일러별 경고 차이                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MSVC (Visual Studio):                                          │
│  - Windows 기본 컴파일러                                        │
│  - 상대적으로 관대한 경고                                       │
│  - default case 없어도 경고 안 함 (기본 설정)                   │
│                                                                 │
│  Clang/GCC:                                                     │
│  - Linux/Mac 기본 컴파일러                                      │
│  - 더 엄격한 경고                                               │
│  - -Wswitch-default: default case 없으면 경고                   │
│                                                                 │
│  크로스 플랫폼 프로젝트에서는 모든 컴파일러 경고를 0으로        │
│  유지하는 것이 좋은 관행                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### -Wswitch-default 경고

```cpp
// Clang 경고 발생 코드
switch (format)
{
    case DXGI_FORMAT_R8_UNORM:
        return Format::R8_UNORM;
    case DXGI_FORMAT_R8G8_UNORM:
        return Format::R8G8_UNORM;
    // ... 많은 case들 ...
    case DXGI_FORMAT_BC7_UNORM_SRGB:
        return Format::BC7_UNORM_SRGB;
    // default 없음 → 경고!
}

// warning: enumeration value 'DXGI_FORMAT_UNKNOWN' not handled in switch
// warning: switch missing default case [-Wswitch-default]
```

---

## 문제: default case 누락

### DX12 _ConvertFormat 함수

```cpp
Format _ConvertFormat(DXGI_FORMAT format)
{
    switch (format)
    {
        case DXGI_FORMAT_R8_UNORM:
            return Format::R8_UNORM;
        case DXGI_FORMAT_R8G8_UNORM:
            return Format::R8G8_UNORM;
        // ... 수십 개의 format 매핑 ...
        case DXGI_FORMAT_BC7_UNORM_SRGB:
            return Format::BC7_UNORM_SRGB;
        // DXGI_FORMAT은 100개 이상의 값이 있음
        // 모든 값을 처리하지 않으면 경고
    }
    return Format::UNKNOWN;
}
```

### Clang 경고 메시지

```
warning: enumeration values 'DXGI_FORMAT_UNKNOWN', 'DXGI_FORMAT_R32G32B32A32_TYPELESS',
'DXGI_FORMAT_R32G32B32A32_FLOAT' ... not handled in switch [-Wswitch]
```

---

## 해결: default case 추가

```cpp
Format _ConvertFormat(DXGI_FORMAT format)
{
    switch (format)
    {
        case DXGI_FORMAT_R8_UNORM:
            return Format::R8_UNORM;
        // ... 모든 case들 ...
        case DXGI_FORMAT_BC7_UNORM_SRGB:
            return Format::BC7_UNORM_SRGB;
        default:  // ✅ 추가
            break;
    }
    return Format::UNKNOWN;
}
```

### default case의 역할

```
┌─────────────────────────────────────────────────────────────────┐
│              default case 추가 효과                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 경고 제거                                                   │
│     - Clang의 -Wswitch-default 경고 해결                        │
│                                                                 │
│  2. 의도 명확화                                                 │
│     - "나머지 값들은 의도적으로 처리하지 않음"                  │
│     - 실수로 case를 빠뜨린 게 아님을 명시                       │
│                                                                 │
│  3. 안전한 폴백                                                 │
│     - 예상치 못한 값이 들어와도 크래시 없이 처리                │
│     - 함수 끝의 return Format::UNKNOWN으로 도달                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 비유: 택배 분류

```
┌─────────────────────────────────────────────────────────────────┐
│                    택배 분류 비유                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  switch (지역코드) {                                            │
│      case 서울: 서울창고로;                                     │
│      case 부산: 부산창고로;                                     │
│      case 대구: 대구창고로;                                     │
│      // 나머지 지역은?                                          │
│  }                                                              │
│                                                                 │
│  default 없음:                                                  │
│  "제주도 택배가 왔는데 어디로 보내죠?" (경고)                   │
│                                                                 │
│  default 있음:                                                  │
│  "그 외 지역은 기타창고로 보내세요" (명확)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 크로스 플랫폼 빌드

### WickedEngine의 플랫폼 지원

```
WickedEngine 지원 플랫폼:
- Windows (MSVC, Clang)
- Linux (GCC, Clang)
- Mac (Clang) ← 커밋 #21에서 추가됨

각 플랫폼에서 경고 없이 빌드되어야 함
→ 모든 컴파일러의 경고를 만족시키는 코드 작성 필요
```

### 경고를 에러로 처리

```cmake
# 일반적인 CI/CD 설정
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_compile_options(-Werror)  # 경고를 에러로
endif()
```

경고를 에러로 처리하면 경고 하나도 빌드 실패로 이어지므로
모든 경고를 수정해야 함.

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | `_ConvertFormat` switch문에 `default: break;` 추가 |

### 코드 위치

```cpp
// GraphicsDevice_DX12.cpp - _ConvertFormat 함수
Format _ConvertFormat(DXGI_FORMAT format)
{
    switch (format)
    {
        // ... 모든 case들 ...
        case DXGI_FORMAT_BC7_UNORM_SRGB:
            return Format::BC7_UNORM_SRGB;
        default:
            break;
    }
    return Format::UNKNOWN;
}
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 문제 | Clang에서 switch문 default case 누락 경고 |
| 해결 | `default: break;` 추가 |
| 목적 | 크로스 플랫폼 빌드 시 경고 제거 |
| VizMotive | ✅ 적용 완료 |

### 핵심 교훈

> **switch문에는 default case 추가**
>
> - 모든 enum 값을 처리하지 않을 때 명시적으로 default 추가
> - 의도를 명확히 하고 컴파일러 경고 방지
> - 크로스 플랫폼 호환성 향상

> **모든 컴파일러 경고를 0으로 유지**
>
> - MSVC, GCC, Clang 모두에서 경고 없이 빌드
> - CI/CD에서 경고를 에러로 처리하면 코드 품질 유지
