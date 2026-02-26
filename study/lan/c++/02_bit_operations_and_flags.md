# 비트 연산과 플래그 (Bit Operations & Flags)

Java에도 비트 연산이 있지만, C++에서는 **상태 플래그 조합**에 자주 쓰인다.
VizMotive/WickedEngine 코드에서 `has_flag()`, `|=`, `&=`, `~` 등이 많이 등장한다.

---

## 기본 비트 연산자

| 연산 | 기호 | 의미 | 예시 |
|------|------|------|------|
| AND | `&` | 두 비트 모두 1이면 1 | `0b1010 & 0b1100 = 0b1000` |
| OR  | `\|` | 둘 중 하나라도 1이면 1 | `0b1010 \| 0b0101 = 0b1111` |
| XOR | `^` | 다르면 1 | `0b1010 ^ 0b1100 = 0b0110` |
| NOT | `~` | 비트 반전 | `~0b1010 = 0b0101` |
| 왼쪽 시프트 | `<<` | 비트를 왼쪽으로 n칸 이동 (×2ⁿ) | `1 << 3 = 8` |
| 오른쪽 시프트 | `>>` | 비트를 오른쪽으로 n칸 이동 (÷2ⁿ) | `8 >> 2 = 2` |

---

## 플래그 패턴

여러 상태를 하나의 정수에 비트 단위로 저장하는 패턴. 메모리를 아끼고 조합이 쉽다.

```cpp
// 각 플래그를 비트 하나씩 할당
enum class Flags
{
    NONE           = 0,
    IMPORT_RETAIN  = 1 << 0,  // 0001
    IMPORT_NORMAL  = 1 << 1,  // 0010
    IMPORT_BLOCK   = 1 << 2,  // 0100
    STREAMING      = 1 << 3,  // 1000
};
```

VizMotive 엔진 실제 코드 (`wiResourceManager.h`):
```cpp
enum class Flags
{
    NONE = 0,
    IMPORT_COLORGRADINGLUT  = 1 << 0,
    IMPORT_RETAIN_FILEDATA  = 1 << 1,
    IMPORT_NORMALMAP        = 1 << 2,
    IMPORT_BLOCK_COMPRESSED = 1 << 3,
    IMPORT_DELAY            = 1 << 4,
    STREAMING               = 1 << 5,
};
```

---

## 플래그 조작 관용구

```cpp
Flags f = Flags::NONE;

// 플래그 켜기 (OR 대입)
f |= Flags::STREAMING;         // f에 STREAMING 비트를 추가

// 플래그 끄기 (AND + NOT 대입)
f &= ~Flags::STREAMING;        // f에서 STREAMING 비트를 제거
//    ↑ ~Flags::STREAMING = STREAMING 비트만 0, 나머지 1 → AND하면 그 비트만 지워짐

// 플래그 켜져있는지 확인
bool isStreaming = (f & Flags::STREAMING) != 0;
// 또는 has_flag() 헬퍼 사용
bool isStreaming = has_flag(f, Flags::STREAMING);
```

VizMotive 엔진 실제 코드:
```cpp
// wiResourceManager.cpp
flags &= ~Flags::STREAMING;  // 스트리밍 비트 끄기
if (has_flag(flags, Flags::STREAMING)) { ... }  // 스트리밍 비트 확인
```

---

## `has_flag()` 헬퍼 함수

`(a & b) != 0` 의 가독성 있는 래퍼. 엔진에서 직접 정의해서 쓴다.

```cpp
template <typename T>
inline bool has_flag(T value, T flag)
{
    return (value & flag) != 0;
    // 또는 (value & flag) == flag  — flag의 모든 비트가 켜져 있어야 true
}
```

---

## `enable_bitmask_operators` 패턴

`enum class`는 기본적으로 `|`, `&` 연산자를 지원하지 않는다.
VizMotive/WickedEngine에서는 이를 활성화하기 위한 트릭을 쓴다.

```cpp
// 이 enum class가 비트 연산을 지원하도록 표시
template<>
struct enable_bitmask_operators<ResourceMiscFlag> {
    static const bool enable = true;
};

// 이 struct가 enable=true이면 |, &, ~, ^ 연산자를 자동으로 제공
// (실제 연산자는 CommonInclude.h 등에서 enable=true인 경우만 작동하도록 정의)
```

---

## 정수 타입 (`uint8_t`, `uint32_t`, `uint64_t`)

Java의 `int`는 항상 32비트, `long`은 64비트로 고정.
C++에서는 플랫폼에 따라 `int` 크기가 다를 수 있어 **고정 폭 정수 타입**을 쓴다.

| 타입 | 범위 | Java 대응 |
|------|------|------|
| `uint8_t`  | 0 ~ 255 (부호 없음 8비트) | (없음, `byte`는 부호 있음) |
| `uint16_t` | 0 ~ 65535 | `char` (부호 없음 16비트) |
| `uint32_t` | 0 ~ 4294967295 | `long` 일부 범위 |
| `uint64_t` | 0 ~ 2⁶⁴-1 | `long` (부호 있음이라 차이 있음) |
| `int32_t`  | -2³¹ ~ 2³¹-1 | `int` |
| `int64_t`  | -2⁶³ ~ 2⁶³-1 | `long` |

VizMotive 엔진에서 자주 보이는 패턴:
```cpp
uint64_t frame_fence_values[BUFFERCOUNT] = {};  // fence 값 (크게 증가하므로 64비트)
uint32_t FRAMECOUNT = 0;                        // 프레임 카운터
uint32_t bufferindex = FRAMECOUNT % 2;          // 버퍼 인덱스
```

---

## 비트필드 (Bit field)

구조체의 멤버에 사용할 비트 수를 직접 지정. 메모리 절약에 유용.

```cpp
struct MeshRenderingVariant
{
    struct Bits
    {
        uint32_t renderpass    : 4;   // 4비트만 사용 (0~15 값 표현 가능)
        uint32_t shadertype    : 4;   // 4비트
        uint32_t blendmode     : 2;   // 2비트 (0~3)
        uint32_t tessellation  : 1;   // 1비트 (0 또는 1)
        uint32_t alphatest     : 1;   // 1비트
    } bits;
    uint32_t raw;   // 전체를 하나의 uint32_t로도 접근 가능
};
```

VizMotive 엔진 실제 코드 (Renderer.cpp에서):
```cpp
variant.bits.renderpass = renderPass;
variant.bits.shadertype = SCU32(material.GetShaderType());
variant.bits.tessellation = tessellatorRequested;
```
