# 메모리 관리와 Placement New

Java에서의 메모리 관리는 전적으로 JVM(가비지 컬렉터)이 담당한다. `new`를 통해 객체를 생성하면 힙(Heap) 어딘가에 메모리가 할당되고 자동으로 초기화되며, 사용이 끝나면 언젠가 알아서 수거된다.

C++에서는 **"메모리를 확보하는 것(Allocation)"**과 **"확보한 메모리에 객체를 초기화하는 것(Construction)"**을 명확히 분리할 수 있다. 이 분리의 핵심이 바로 **Placement New**다. VizMotive나 WickedEngine 같은 게임 엔진에서 성능을 영혼까지 끌어모으기 위해 필수적으로 쓰는 기법이다.

---

## 1. 기본 `new`의 동작 방식 (Java와 유사)

일반적인 `new` 연산자는 다음 두 가지 일을 순서대로 합쳐서 처리한다:

```cpp
// 일반적인 객체 생성
Vector2* p = new Vector2(10, 20);
```
위 코드는 내부적으로 두 단계를 거친다:
1. **Allocation (할당)**: OS에 요청해 `Vector2` 크기(예: 8바이트)만큼의 힙 메모리를 찾아서 가져온다. (이 과정이 매우 느림)
2. **Construction (생성)**: 가져온 그 메모리 주소에서 `Vector2(10, 20)` 생성자를 호출해 값을 채운다.

## 2. Placement New (위치 지정 new)

만약 우리가 "이미 메모리를 뭉텅이로 확보해 놨으니, 거기다 생성자만 호출해줘!" 라고 하고 싶으면 어떻게 해야 할까? (이것이 커스텀 할당자의 핵심 요구사항이다)
이때 사용하는 문법이 **Placement New**이다.

```cpp
#include <new> // placement new를 쓰기 위한 헤더

// 1. 메모리만 미리 크게 확보 (값 초기화 전혀 없음, 그냥 빈 공간)
void* preAllocatedMemory = malloc(sizeof(Vector2) * 100);

// 2. 확보해둔 메모리의 특정 주소(예: 첫 번째 위치)에 객체를 "생성(초기화)만" 한다.
// 문법: new (메모리주소) 타입(생성자인자)
Vector2* p1 = new (preAllocatedMemory) Vector2(10, 20);
```

이 코드는 OS에 새로운 메모리를 달라고 요청하지 않는다. 단지 `preAllocatedMemory`가 가리키는 기존 주소로 찾아가서 `Vector2(10, 20)` 생성자만 실행할 뿐이다. **엄청나게 빠르다.**

---

## 3. 소멸과 Placement New의 짝꿍

일반 `new`로 만든 건 `delete`로 지운다. `delete`는 1) 소멸자를 부르고, 2) 메모리를 OS에 반환한다.

하지만 Placement New로 만든 객체는 **절대 `delete`를 해서는 안 된다.** 메모리를 OS에 돌려주면 안 되기 때문이다(우리가 미리 뭉텅이로 떼어온 메모리의 일부분일 뿐이다).
대신 **명시적으로 소멸자만 호출**해야 한다. (Java에서는 소멸자를 직접 호출한다는 개념 자체가 없으므로 매우 낯선 문법이다.)

```cpp
// 1. 소멸자만 명시적으로 호출 (메모리를 비우는 정리 작업만 함)
p1->~Vector2();

// 2. 나중에 확보해뒀던 거대한 메모리 덩어리를 한 번에 해제
free(preAllocatedMemory);
```

---

## 4. VizMotive 엔진에서의 실제 활용 (Block Allocator)

게임 렌더링 중에는 1프레임(약 16ms) 안에도 수만 개의 임시 객체(`CopyCMD` 등)가 만들어졌다 사라진다. 매번 기본 `new`를 쓰면 메모리 파편화(Fragmentation)가 심해지고 게임이 심각하게 느려진다.

이를 해결하기 위해 `Block Allocator` (또는 Free List Allocator) 구조를 쓴다.

```cpp
// 엔진 내부의 Block Allocator 핵심 로직 요약

// 배열/벡터 형태로 거대한 메모리를 "한 번에" 잡아둔다 (엔진 시작 시)
uint8_t memoryPool[1000 * sizeof(CopyCMD)];
void* free_list[1000]; // 비어있는 메모리 주소들을 기억하는 스택

// 객체가 필요할 때: Placement New 사용
void* slot = free_list.back();  // 비어있는 주소 하나를 꺼냄
free_list.pop_back();
CopyCMD* cmd = new (slot) CopyCMD();  // 할당 없이 그 자리에 생성자만 호출! (초고속!!!)

// 사용이 끝났을 때: 소멸자 호출 후 주소 반납
cmd->~CopyCMD();                 // 메모리 해제가 아님! 내부 정리만 함
free_list.push_back(slot);       // 다 쓴 주소표를 다시 빈 목록에 반납
```

이 패턴 덕분에 엔진 실행 중에는 실질적인 메인 메모리 할당(`malloc`/`new`)이 단 한 번도 일어나지 않게 되어, 일정한 프레임 레이트를 보장할 수 있다.
