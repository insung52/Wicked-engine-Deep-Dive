# DLL과 링킹 기초

---

## 1. 실행 파일의 종류

C++ 코드를 빌드하면 세 가지 형태의 파일이 나온다.

### .exe (Executable)
프로그램의 진입점(`main()`). OS가 직접 실행하는 파일.

### .lib (Static Library)
컴파일된 코드 묶음. **링크 시점에 .exe 안으로 복사**되어 사라진다.
배포할 때 .lib 파일은 필요 없다 — 이미 .exe 안에 들어있다.

### .dll (Dynamic Link Library)
컴파일된 코드 묶음. **.exe와 별도 파일로 존재**하며, **런타임에 로드**된다.
배포할 때 .dll 파일도 같이 배포해야 한다.

```
Static Library (.lib):                Dynamic Library (.dll):

[코드 A]  [코드 B]                    app.exe        engine.dll
    ↓ 링크 시점에 복사                  │              │
  app.exe                             └──────────────┘
  [코드 A][코드 B]                      런타임에 로드
```

### 비교

| | .exe | .lib | .dll |
|---|---|---|---|
| 독립 실행 | ✅ | ❌ | ❌ |
| 링크 시점 | — | exe에 합쳐짐 | 별도 유지 |
| 로드 시점 | 즉시 | (이미 exe 안) | 런타임 |
| 코드 공유 | ❌ | ❌ | ✅ (여러 프로세스가 같은 .dll 사용 가능) |
| 메모리 분리 | — | — | 각 .dll은 자체 메모리 공간 |

---

## 2. 링킹(Linking)이란

C++ 빌드는 두 단계로 나뉜다.

```
[.cpp 파일들]
     ↓ 컴파일러 (cl.exe)
[.obj 파일들]  (각 .cpp가 독립적으로 번역된 기계어 덩어리)
     ↓ 링커 (link.exe)
[.exe / .dll]  (obj들을 하나로 묶고, 심볼 참조를 해결)
```

### 심볼(symbol)이란

컴파일러가 함수, 변수에 부여하는 이름. 링커가 "이 이름이 어디에 정의되어 있는지" 찾아서 연결한다.

```cpp
// a.cpp
void foo() { ... }   // 심볼 "foo" 정의

// b.cpp
void foo();          // 선언만 (어딘가에 정의가 있다고 가정)
void bar() { foo(); }  // 링커가 "foo"를 찾아서 연결
```

링크 에러 `unresolved external symbol`은 링커가 심볼을 찾지 못했다는 뜻이다.

---

## 3. DLL export / import

DLL은 자신의 함수/변수를 **외부에 공개(export)** 해야 다른 .exe나 .dll이 사용할 수 있다.

### `__declspec(dllexport)` / `__declspec(dllimport)`

Windows MSVC 전용 키워드.

```cpp
// engine.dll을 만들 때:
__declspec(dllexport) void Initialize();   // 이 함수를 DLL 밖에 공개

// app.exe에서 engine.dll을 사용할 때:
__declspec(dllimport) void Initialize();   // 이 함수가 DLL에서 온다고 표시
```

`dllimport` 없이도 동작하지만, 붙이면 컴파일러가 더 효율적인 코드를 생성한다.

### 실제 코드에서의 패턴

매번 export/import를 다르게 쓰기 번거로우므로 매크로로 정리한다:

```cpp
// Engine.h
#ifdef ENGINE_EXPORTS           // Engine.dll을 빌드할 때 이 매크로 정의
    #define ENGINE_API __declspec(dllexport)
#else                           // Engine.dll을 사용하는 쪽
    #define ENGINE_API __declspec(dllimport)
#endif

ENGINE_API void Initialize();   // 빌드 대상에 따라 자동으로 export/import
ENGINE_API int GetVersion();
```

- **Engine.dll 빌드 시**: `ENGINE_EXPORTS`가 정의되어 `ENGINE_API = dllexport`
- **app.exe 빌드 시**: `ENGINE_EXPORTS`가 없으므로 `ENGINE_API = dllimport`

### export된 것과 아닌 것의 차이

```cpp
// engine.dll

ENGINE_API void PublicFunc();   // app.exe에서 호출 가능
static void InternalFunc();     // engine.dll 내부에서만 사용, 외부 접근 불가
```

export하지 않은 함수/변수는 DLL 내부 구현 세부사항이다. 헤더에 선언해도 다른 DLL에서는 링크 에러.

---

## 4. 헤더 파일과 컴파일 단위

### 헤더는 "복사-붙여넣기"

`#include`는 단순히 헤더 파일의 내용을 그 자리에 붙여넣는 전처리기 명령이다.

```cpp
// math.h
int add(int a, int b) { return a + b; }

// foo.cpp
#include "math.h"   // → foo.cpp에 add 함수 코드가 복사됨

// bar.cpp
#include "math.h"   // → bar.cpp에도 add 함수 코드가 복사됨
```

`math.h`를 include한 .cpp 파일마다 `add` 함수의 코드가 복사된다.

### ODR (One Definition Rule)

C++ 규칙: **하나의 심볼은 프로그램 전체에서 딱 하나만 정의되어야 한다.**

위 예시처럼 `add`가 foo.cpp와 bar.cpp 양쪽에 정의되면 ODR 위반 → 링커 에러.

**해결 방법 3가지:**

```cpp
// 1. inline 키워드 (C++17 이전부터 함수에 사용 가능)
inline int add(int a, int b) { return a + b; }
// → 각 .cpp에 복사되어도 ODR 예외 처리, 링커가 중복 제거

// 2. 헤더에는 선언만, 정의는 .cpp에
// math.h:   int add(int a, int b);           (선언)
// math.cpp: int add(int a, int b) { ... }   (정의 1개만)

// 3. template (암묵적으로 inline처럼 동작)
template<typename T>
T add(T a, T b) { return a + b; }
```

---

## 5. `inline` 변수 (C++17)

C++17 이전에는 `inline`이 함수에만 적용됐다. C++17부터 **변수**에도 `inline`을 붙일 수 있다.

```cpp
// config.h
inline int global_count = 0;   // 헤더에 정의, ODR 위반 없음
```

**단일 실행 파일(.exe) 안에서는 딱 하나의 인스턴스만 존재**한다. 어디서 include해도 같은 변수를 본다.

---

## 6. inline 변수의 DLL 경계 문제 ← 핵심

### 문제

`inline` 변수의 "하나의 인스턴스" 보장은 **같은 빌드 단위** 안에서만 적용된다.

DLL은 **독립적으로 빌드**된다. 같은 헤더를 include해도 DLL마다 별도로 컴파일한다.

```cpp
// allocator.h
inline int* block_allocators[256] = {};   // inline 변수
```

```
Engine.dll 빌드:
  allocator.h를 include한 모든 .cpp → Engine.dll
  → block_allocators[256] 인스턴스 A 생성 (전부 nullptr)

GBackendDX12.dll 빌드:
  allocator.h를 include한 모든 .cpp → GBackendDX12.dll
  → block_allocators[256] 인스턴스 B 생성 (전부 nullptr)
```

링크 단계가 **DLL마다 따로** 실행되므로, 각 DLL이 자기 자신의 인스턴스를 갖게 된다.

### 실제 문제 시나리오

```
GBackendDX12.dll:
  block_allocators[0] = &texture_allocator   ← 등록 (인스턴스 B)

Engine.dll에서 조회:
  block_allocators[0]                        ← 인스턴스 A 조회
  → nullptr!  (Engine.dll은 등록한 적 없음)
  → nullptr->inc_refcount() → 크래시
```

**같은 이름의 변수지만, 두 DLL이 서로 다른 메모리를 보고 있다.**

### 비교

```
단일 exe (WickedEngine):
  모든 코드가 한 링크 단계 → block_allocators 인스턴스 하나 → 문제없음

다중 DLL (VizMotive):
  Engine.dll 링크 단계  → 인스턴스 A
  DX12.dll 링크 단계   → 인스턴스 B  ← 서로 다른 인스턴스!
```

---

## 7. 해결 방법들

### 방법 1: `extern` + `.cpp` 정의 (전통적)

```cpp
// allocator.h
extern int* block_allocators[256];   // 선언만 (어딘가에 딱 하나 정의)

// allocator.cpp (Engine.dll의 .cpp 중 하나)
int* block_allocators[256] = {};     // 정의 1개
```

`extern`은 "이 변수는 다른 곳에 정의되어 있다"는 선언. 링커가 하나의 정의를 찾아서 연결한다.

단, 이 경우 `block_allocators`는 **Engine.dll의 것**으로 고정된다. GBackendDX12.dll이 접근하려면 Engine.dll에서 export해야 한다.

```cpp
// allocator.h
ENGINE_API extern int* block_allocators[256];   // export까지 포함
```

### 방법 2: 포인터를 직접 저장 (apply_allocator_shared_ptr 방식)

전역 배열 조회를 아예 없애고, 생성 시점에 allocator 포인터를 객체 안에 직접 저장한다.

```cpp
// WickedEngine: 전역 배열 조회 → DLL 경계에서 실패
struct shared_ptr {
    uint64_t handle;   // ptr | allocator_id
    // 사용 시: block_allocators[handle & 0xFF]->inc_refcount(...)
};

// VizMotive: 직접 포인터 → DLL 어디서든 안전
struct shared_ptr {
    T* ptr;
    Allocator* allocator;   // 어느 DLL의 allocator인지 직접 가리킴
    // 사용 시: allocator->inc_refcount(...)
};
```

전역 배열을 보지 않으므로, DLL 경계를 넘어도 항상 올바른 allocator에 접근한다.

---

## 8. 정리

| 개념 | 요약 |
|------|------|
| `.exe` | OS가 직접 실행하는 진입점 |
| `.dll` | 런타임에 로드되는 코드 묶음, 별도 메모리 공간 |
| `.lib` | 링크 시점에 .exe에 합쳐지는 코드 묶음 |
| 링킹 | 심볼 참조를 해결해서 실행 파일을 완성하는 단계 |
| `dllexport` | DLL 밖에 함수/변수를 공개 |
| `dllimport` | 공개된 함수/변수를 사용하겠다고 표시 |
| `inline` 변수 | 헤더에 정의 가능, ODR 면제 — 단, **DLL마다 별도 인스턴스** |
| `extern` | "다른 곳에 정의가 있다"는 선언, 링커가 하나의 정의에 연결 |
| DLL 경계 문제 | 같은 inline 변수도 DLL마다 다른 인스턴스 → 전역 공유 불가 |
