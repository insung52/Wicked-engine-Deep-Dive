# const와 constexpr

Java의 `final`과 비슷해 보이지만 C++의 `const`는 훨씬 다양한 맥락에서 쓰인다.

---

## `const` 변수

Java의 `final`과 유사. 선언 후 값 변경 불가.

```cpp
const int MAX_FRAMES = 3;
// MAX_FRAMES = 5;  // 컴파일 에러

// Java와 차이: Java final은 참조가 불변, C++ const는 값 자체가 불변
```

---

## `const` 함수 (멤버 함수)

Java에는 없는 개념. **이 함수는 객체를 수정하지 않는다**는 약속.

```cpp
class CommandList_DX12
{
    int id = 0;
public:
    int getId() const { return id; }  // ← const 함수: this의 멤버 변경 불가
    void setId(int v) { id = v; }     // ← 일반 함수: 멤버 변경 가능
};
```

VizMotive 엔진 실제 코드:
```cpp
// GraphicsDevice_DX12.h
inline bool IsValid() const { return commandList != nullptr; }
inline bool IsCompleted() const { return fence->GetCompletedValue() >= fenceValueSignaled; }
```

`const` 객체에서는 `const` 멤버 함수만 호출 가능:
```cpp
const CopyCMD cmd = ...;
cmd.IsValid();    // OK — const 함수
cmd.someMethod(); // 에러 — non-const 함수는 const 객체에서 호출 불가
```

---

## `const` 참조 파라미터

함수에 큰 객체를 **복사 없이 읽기 전용**으로 전달. 가장 많이 보이는 패턴.

```cpp
// ❌ 비효율: Visibility 전체를 복사
void DrawScene(Visibility vis, ...) { }

// ✅ 효율적: 복사 없이 참조, 수정 불가 보장
void DrawScene(const Visibility& vis, ...) { }
```

VizMotive 엔진 실제 코드:
```cpp
// Renderer.cpp
void GRenderPath3DDetails::DrawScene(const Visibility& vis, RENDERPASS renderPass, CommandList cmd, uint32_t flags)
void UpdatePerFrameData(const Scene& scene, const Visibility& vis, ...)
```

---

## `const` 포인터 — 헷갈리는 조합

```cpp
int x = 1;

const int* p = &x;    // 가리키는 값이 const (포인터는 변경 가능)
int* const p = &x;    // 포인터가 const (가리키는 값은 변경 가능)
const int* const p = &x; // 둘 다 const
```

읽는 법: `*`를 기준으로
- 왼쪽 `const` → 값이 불변
- 오른쪽 `const` → 포인터가 불변

---

## `constexpr` — 컴파일 타임 상수

`const`는 런타임에도 결정될 수 있다.
`constexpr`은 **컴파일 타임에 반드시 결정**되어야 하는 상수.

```cpp
const int n = someFunction();    // OK — 런타임에 결정되어도 됨
constexpr int n = someFunction(); // 에러 — someFunction이 constexpr이 아니면 불가

constexpr int SIZE = 1024;      // OK — 리터럴은 컴파일 타임 확정
constexpr int HALF = SIZE / 2;  // OK — 컴파일 타임에 계산
```

VizMotive 엔진 실제 코드:
```cpp
// GraphicsDevice_DX12.h
constexpr CommandList_DX12& GetCommandList(CommandList cmd) const { ... }
//        ↑ 이 함수는 컴파일러가 인라인하거나 최적화를 강하게 적용할 수 있음

// wiResourceManager.cpp
static constexpr size_t streaming_texture_min_size = 64 * 1024; // 64KB — 컴파일 타임
```

---

## `constexpr` 함수

함수도 `constexpr`로 선언하면 컴파일 타임에 평가될 수 있다.

```cpp
constexpr int square(int x) { return x * x; }

constexpr int val = square(5);  // 컴파일 타임에 25로 계산
int arr[square(4)];             // 컴파일 타임 계산이므로 배열 크기로 사용 가능
```

---

## `static constexpr` — 클래스 상수

Java의 `static final`에 해당.

```cpp
class Foo
{
    static constexpr int MAX = 100;  // Java: static final int MAX = 100;
    static constexpr const char* NAME = "foo";
};
```

---

## 정리

| 키워드 | 의미 | Java 대응 |
|--------|------|-----------|
| `const 변수` | 값 변경 불가 | `final` |
| `const 멤버 함수` | 객체 상태 변경 불가 | (없음) |
| `const T&` 파라미터 | 읽기 전용 참조 전달 | (없음, Java 참조는 항상 mutable) |
| `constexpr` | 컴파일 타임 상수 | `static final` (일부) |
| `static constexpr` | 클래스 레벨 컴파일 타임 상수 | `static final` |
