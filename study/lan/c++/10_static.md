# C++의 static

C++에서 `static`은 **문맥에 따라 의미가 완전히 다르다.** 같은 키워드지만 3가지 독립적인 용도로 쓰인다.

---

## 1. 파일 스코프 static (전역 변수/함수에 붙이는 경우)

```cpp
// foo.cpp
static int count = 0;          // 파일 스코프 static 변수
static void helper() { ... }   // 파일 스코프 static 함수
```

**의미: "이 이름은 이 .cpp 파일 밖으로 노출하지 않는다."**

링커가 이 심볼을 외부에 공개하지 않는다. 다른 .cpp 파일에서 `extern int count;`로 접근하려 해도 링크 에러가 난다.

### static이 없으면

```cpp
// foo.cpp
int count = 0;  // 외부 링크 (external linkage)

// bar.cpp
extern int count;  // foo.cpp의 count에 접근 가능
count = 5;         // 의도치 않은 직접 접근!
```

### static이 있으면

```cpp
// foo.cpp
static int count = 0;  // 내부 링크 (internal linkage)

// bar.cpp
extern int count;  // 링크 에러: count를 찾을 수 없음
```

### 언제 쓰는가

- 외부에 숨겨야 하는 전역 변수/함수
- 같은 이름을 다른 .cpp에서도 쓰고 싶을 때 (서로 다른 인스턴스로 분리)

```cpp
// Allocator.cpp
static SharedBlockAllocator* block_allocators[256] = {};
// → 외부 DLL이 직접 접근 불가, get_shared_block_allocator() 함수를 통해서만 접근
```

### Java와 비교

Java의 `private static` 필드와 개념이 가장 가깝다 — "이 파일(클래스) 안에서만 쓸 수 있는 공유 변수". 단 Java는 클래스 단위, C++은 파일(.cpp) 단위로 제한된다.

---

## 2. 함수 내 static (지역 변수에 붙이는 경우)

```cpp
void foo()
{
    static int call_count = 0;  // 함수 내 static
    call_count++;
    // ...
}
```

**의미: "이 변수는 프로그램 전체 수명 동안 딱 한 번만 초기화되고, 함수가 끝나도 사라지지 않는다."**

일반 지역 변수는 함수 호출마다 생성/소멸되지만, `static` 지역 변수는 처음 호출 시 한 번 초기화되고 이후에는 유지된다.

```cpp
void foo()
{
    int normal = 0;    // 매 호출마다 0으로 초기화
    static int s = 0;  // 첫 호출에만 0으로 초기화, 이후 유지

    normal++;
    s++;
    printf("%d %d\n", normal, s);
}

foo();  // 출력: 1 1
foo();  // 출력: 1 2
foo();  // 출력: 1 3
```

### 초기화 시점

C++11부터 함수 내 static 변수의 초기화는 **스레드 안전**하게 보장된다. 첫 번째 호출 시 단 한 번만 초기화된다.

```cpp
// 이 패턴이 자주 쓰인다 (Singleton-like)
Foo& get_instance()
{
    static Foo instance;  // 처음 호출 시 한 번 생성, 이후 같은 객체 반환
    return instance;
}
```

### 언제 쓰는가

- 호출 횟수 카운터
- 처음 한 번만 생성하는 객체 (Singleton)
- 지연 초기화 (처음 필요할 때 만들기)

```cpp
// Allocator.h의 실제 예시
template<typename T, typename... ARG>
inline shared_ptr<T> make_shared_single(ARG&&... args)
{
    static SharedHeapAllocator<T>* allocator = new SharedHeapAllocator<T>;
    //     ↑ 처음 호출 시 한 번만 생성, 이후 재사용
    return allocator->allocate(std::forward<ARG>(args)...);
}
```

---

## 3. 클래스 멤버 static

```cpp
struct Counter
{
    static int total;       // static 멤버 변수
    static void reset();    // static 멤버 함수

    int id;
};
```

**의미: "이 멤버는 객체마다 따로 존재하지 않고, 클래스 전체에서 하나만 존재한다."**

Java의 `static` 멤버와 동일한 개념이다.

### static 멤버 변수

```cpp
struct Counter
{
    static int total;  // 선언
    int id;
};

int Counter::total = 0;  // 정의 (별도 .cpp에서)

Counter a, b, c;
Counter::total = 5;  // 모든 객체가 공유하는 하나의 변수
```

```
메모리:
  Counter::total  [5]          ← 하나만 존재
  a.id            [?]          ← 각 객체마다 존재
  b.id            [?]
  c.id            [?]
```

### static 멤버 함수

```cpp
struct Counter
{
    static int total;
    static void reset() { total = 0; }  // static 함수: this 없음, static 멤버만 접근 가능
    int id;
};

Counter::reset();  // 객체 없이 호출 가능
```

- `this` 포인터가 없다 → 비static 멤버(`id`)에 접근 불가
- 객체 없이 `클래스::함수()` 형태로 호출

### inline static (C++17)

헤더에 static 멤버 변수를 정의할 수 있다. 별도 .cpp 파일 불필요.

```cpp
// 변경 전 (C++17 이전)
// .h: static int total;
// .cpp: int Counter::total = 0;  ← 별도 정의 필요

// 변경 후 (C++17, inline static)
struct Counter
{
    inline static int total = 0;  // 헤더에서 선언+정의 동시에
};
```

```cpp
// Allocator.h의 실제 예시
template<typename T, size_t block_size = 256>
inline static SharedBlockAllocatorImpl<T, block_size>* shared_block_allocator
    = new SharedBlockAllocatorImpl<T, block_size>;
// → 타입 T마다 하나의 allocator 인스턴스 (템플릿 변수)
```

---

## 4. 세 가지 비교

| 문맥 | 코드 | 의미 |
|------|------|------|
| 전역/파일 스코프 | `static int x;` | 이 .cpp 파일 밖에 심볼 노출 안 함 |
| 함수 내 지역 | `static int x;` | 함수 종료 후에도 유지, 최초 1회만 초기화 |
| 클래스 멤버 | `static int x;` | 모든 객체가 공유하는 하나의 인스턴스 |

```cpp
static int file_var = 0;   // (1) 파일 스코프: 이 파일 밖으로 안 나감

void foo()
{
    static int local = 0;  // (2) 함수 내: 호출 간 유지
}

struct Bar
{
    static int member;     // (3) 클래스 멤버: 모든 Bar 객체가 공유
};
```

---

## 5. Java static과 전체 비교

| | Java `static` | C++ 클래스 멤버 `static` | C++ 파일 스코프 `static` | C++ 함수 내 `static` |
|---|---|---|---|---|
| 위치 | 클래스 멤버 | 클래스 멤버 | 전역/함수 밖 | 함수 안 |
| 의미 | 클래스에 귀속 | 클래스에 귀속 | 파일 안에서만 접근 | 수명 연장 |
| Java 대응 | `static` | `static` | `private`(파일 단위) | (Java에 없음) |
