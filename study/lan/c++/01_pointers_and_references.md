# 포인터와 참조 (Pointers & References)

Java와 가장 큰 차이. Java는 모든 객체가 내부적으로 참조지만 포인터를 직접 다루지 않는다.
C++에서는 메모리 주소를 직접 다룰 수 있고, 포인터와 참조가 명확히 분리되어 있다.

---

## 포인터 (`*`)

변수의 **메모리 주소**를 담는 변수.

```cpp
int x = 42;
int* p = &x;     // p는 x의 주소를 담는다
                 // & = "주소를 가져와라" (address-of 연산)

*p = 99;         // p가 가리키는 곳의 값을 99로 변경  → x가 99가 됨
                 // * = "포인터가 가리키는 값" (dereference)
```

### Java와 비교

| | Java | C++ |
|---|---|---|
| 객체 생성 | `Foo f = new Foo()` | `Foo* f = new Foo()` (힙) 또는 `Foo f` (스택) |
| 메서드 호출 | `f.method()` | `f->method()` (포인터) / `f.method()` (값) |
| null 체크 | `if (f != null)` | `if (f != nullptr)` |
| 해제 | GC가 자동 | `delete f` (직접 해제해야 함) |

### `->` 연산자

포인터로 멤버에 접근할 때 사용. `(*p).method()`의 축약형.

```cpp
Foo* f = new Foo();
f->method();    // (*f).method() 와 동일
f->value = 10;  // (*f).value = 10 과 동일
```

VizMotive 엔진 예시:
```cpp
// GraphicsDevice_DX12.cpp
hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence[buffer][queue]));
//           ↑ device는 ComPtr<ID3D12Device5> — 포인터처럼 동작
```

---

## 참조 (`&`)

기존 변수의 **별명(alias)**. 포인터와 달리 항상 유효한 값을 가리킨다.
null이 될 수 없고, 선언 시 반드시 초기화해야 한다.

```cpp
int x = 42;
int& ref = x;   // ref는 x의 별명

ref = 99;       // x도 99가 됨 (같은 메모리를 가리킴)
                // *없이 그냥 씀
```

### 함수 파라미터에서의 참조

```cpp
// Java: 기본형은 값 복사, 객체는 참조 전달
// C++: 명시적으로 선택

void byValue(int x)   { x = 99; }  // 복사본 변경 → 원본 변화 없음
void byRef(int& x)    { x = 99; }  // 원본 변경
void byConstRef(const int& x) { }  // 읽기 전용 참조 (복사 없이 안전하게 전달)
```

VizMotive 엔진 예시:
```cpp
// 큰 구조체를 복사 없이 전달할 때 const& 사용
void DrawScene(const Visibility& vis, RENDERPASS renderPass, CommandList cmd)
//                              ↑ Visibility를 복사하지 않고 읽기전용으로 전달
```

### `auto&` 패턴

범위 기반 for에서:
```cpp
std::vector<CommandList> list = ...;

for (auto& item : list)   // item은 각 원소의 참조 (복사 없음, 수정 가능)
for (const auto& item : list)  // 읽기 전용 참조 (복사 없음, 수정 불가)
for (auto item : list)    // 복사 (원본 안 건드림)
```

---

## `const` 포인터와 포인터 `const` — 헷갈리는 패턴

```cpp
int x = 1, y = 2;

int* p         = &x;  // 포인터도 값도 변경 가능
const int* p   = &x;  // 값 변경 불가, 포인터(주소) 변경 가능
int* const p   = &x;  // 포인터 변경 불가, 값 변경 가능
const int* const p = &x; // 둘 다 변경 불가
```

읽는 법: `*` 기준으로 **오른쪽의 const** = 포인터 자체가 const, **왼쪽의 const** = 가리키는 값이 const.

---

## `nullptr`

Java의 `null`에 해당. C에서 쓰던 `NULL` (= 정수 0) 대신 타입 안전한 `nullptr` 사용.

```cpp
int* p = nullptr;
if (p == nullptr) { ... }  // null 체크

// 함수 오버로딩에서 차이:
void foo(int* p) { ... }
void foo(int x)  { ... }
foo(NULL);    // 어떤 foo가 호출될지 모호함 (NULL = 0 은 정수)
foo(nullptr); // 명확하게 포인터 버전 호출
```

---

## 스택 vs 힙 할당

```cpp
// 스택 할당 — 스코프 끝나면 자동 해제
Foo f;           // 스코프 끝나면 자동 소멸자 호출
Foo f2(1, 2);    // 생성자에 인자 전달

// 힙 할당 — 수동 해제 필요 (raw pointer)
Foo* f = new Foo();
delete f;        // 안 하면 메모리 누수

// 힙 할당 — 자동 해제 (스마트 포인터, 권장)
auto f = std::make_shared<Foo>();  // 참조 카운팅으로 자동 해제
auto f = std::make_unique<Foo>();  // 단독 소유, 스코프 끝나면 자동 해제
```

→ 스마트 포인터는 `smart_pointers_and_raii.md` 참고.
