# 람다와 클로저 (Lambda & Closures)

Java 8에도 람다가 있지만, C++ 람다는 **캡처(capture)** 방식이 명시적이라는 큰 차이가 있다.

---

## 기본 문법

```cpp
// Java
Runnable r = () -> System.out.println("hello");

// C++
auto f = []() { std::cout << "hello"; };
f();  // 호출
```

형식: `[캡처목록](파라미터) -> 반환타입 { 본문 }`

- `[]` — 캡처 없음
- `(파라미터)` — 생략 가능 (파라미터 없을 때)
- `-> 반환타입` — 생략 가능 (컴파일러가 추론)

---

## 캡처 목록 `[...]` — Java와의 핵심 차이

Java에서는 람다가 외부 변수를 자동으로 캡처한다 (사실상 effectively final만).  
C++에서는 **어떻게 캡처할지 명시**해야 한다.

| 캡처 | 의미 |
|------|------|
| `[]` | 외부 변수 캡처 없음 |
| `[x]` | x를 **값으로** 캡처 (복사본) |
| `[&x]` | x를 **참조로** 캡처 (원본) |
| `[=]` | 사용하는 모든 외부 변수를 **값으로** 캡처 |
| `[&]` | 사용하는 모든 외부 변수를 **참조로** 캡처 |
| `[this]` | 현재 객체의 `this` 포인터를 캡처 |
| `[=, &x]` | 기본은 값 캡처, x만 참조 캡처 |

```cpp
int count = 0;
std::string name = "Alice";

// 값 캡처 — 람다 내부에서 count를 수정해도 외부 count는 변화 없음
auto f1 = [count]() { /* count 읽기만 가능 */ };

// 참조 캡처 — 람다 내부에서 count 수정 시 외부도 변함
auto f2 = [&count]() { count++; };

// this 캡처 — 멤버 함수 안에서 쓸 때
auto f3 = [this]() { this->someMethod(); };
```

---

## VizMotive 엔진 실제 코드 예시

### jobsystem::Execute 에서의 람다 (Renderer.cpp)

```cpp
jobsystem::context ctx;

// [this, cmd]를 값으로 캡처
jobsystem::Execute(ctx, [this, cmd](jobsystem::JobArgs args) {
    device->WaitCommandList(cmd, cmd_prepareframe);
    DrawScene(visMain, RENDERPASS_MAIN, cmd, flags);
});

jobsystem::Wait(ctx);
```

- `this` 캡처: `device`, `visMain` 등 멤버에 접근하기 위해
- `cmd` 캡처: 값 캡처 (커맨드리스트 핸들 = 정수형)

### LoadingScreen 에서의 람다 (wiLoadingScreen.cpp)

```cpp
std::thread([this]() {
    wi::jobsystem::Wait(ctx);
    wi::eventhandler::Subscribe_Once(wi::eventhandler::EVENT_THREAD_SAFE_POINT,
        [this](uint64_t) {
            finish();   // this->finish() 호출
        });
}).detach();
```

람다 안에 람다 중첩. 바깥 람다의 `this` 캡처가 안쪽 람다에는 자동으로 전달되지 않으므로
안쪽 람다도 `[this]`로 명시적으로 캡처.

### AddLoadingComponent (wiLoadingScreen.cpp)

```cpp
// [=] : component, main, fadeSeconds, fadeColor, fadetype 모두 값으로 캡처
addLoadingFunction([=](wi::jobsystem::JobArgs args) {
    component->Load();
});
onFinished([=] {
    main->ActivatePath(component, fadeSeconds, fadeColor, fadetype);
});
```

---

## 람다를 변수에 저장하기

```cpp
// auto로 타입 추론
auto myLambda = [](int x) { return x * 2; };

// std::function으로 타입 지정 (인터페이스로 쓸 때)
std::function<void(int)> callback = [](int x) { std::cout << x; };

// 함수 파라미터로 받을 때
void doSomething(std::function<void()> task) {
    task();
}
```

---

## 즉시 호출 (Immediately Invoked Lambda)

```cpp
// 람다를 선언과 동시에 바로 호출
int result = [](int x, int y) { return x + y; }(3, 4);  // = 7
```

VizMotive에서의 패턴:
```cpp
// dx12_check 매크로에서 사용
#define dx12_check(call) [&]() { HRESULT hr = call; dx12_assert(...); return hr; }()
//                                                                              ↑ 즉시 호출
```

---

## 캡처 주의사항: 수명(lifetime) 문제

참조 캡처 `[&]`는 람다가 외부 변수보다 오래 살면 위험.

```cpp
std::function<void()> createDangerousLambda() {
    int local = 42;
    return [&local]() { std::cout << local; };
    // local은 함수 종료 시 소멸 → 람다 실행 시 dangling reference!
}

// 해결: 값 캡처 사용
return [local]() { std::cout << local; };  // local의 복사본을 캡처
```
