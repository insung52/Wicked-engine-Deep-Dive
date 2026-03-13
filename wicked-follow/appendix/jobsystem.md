# 백그라운드 작업 처리: JobSystem

Wicked Engine은 멀티 스레딩을 위해 자체적인 `wi::jobsystem` (또는 `jobsystem`)을 구현하여 사용합니다.
이 시스템은 C++11의 `std::thread`와 무잠금(lock-free) 큐 등을 활용하여, CPU의 여러 코어를 효율적으로 활용할 수 있게 해주는 엔진 레벨의 유틸리티입니다.

DX12나 Vulkan 같은 그래픽스 API의 기능이 아니라, **엔진에서 자체적으로 제공하는 병렬 처리 프레임워크**입니다.

## 기본 개념

- **Job**: 병렬로 실행할 수 있는 하나의 작업(Task) 단위. 주로 C++ 람다 함수로 정의됩니다.
- **Context**: 잡들을 그룹화하고 동기화(끝날 때까지 기다림)하기 위한 컨텍스트 객체입니다. 한 그룹의 잡들이 모두 완료되었는지 확인할 때 사용됩니다.
- **Worker Threads**: JobSystem을 초기화할 때, 엔진은 CPU 코어 개수만큼의 스레드 풀(Thread Pool)을 미리 생성해 둡니다.

## 주요 사용법

### 1. 단일 작업 병렬 실행 (Execute)

가장 기본적인 형태입니다. 작업을 백그라운드 스레드에서 실행하도록 요청합니다.

```cpp
#include "wiJobSystem.h"

// 1. 컨텍스트 생성 (동기화용)
wi::jobsystem::context ctx;

// 2. 잡 추가 (람다 함수)
wi::jobsystem::Execute(ctx, [](wi::jobsystem::JobArgs args) {
    // 백그라운드 스레드에서 실행될 작업
    ComputeSomethingComplex();
});

// 또 다른 워커 스레드에서 병렬로 실행됨
wi::jobsystem::Execute(ctx, [](wi::jobsystem::JobArgs args) {
    LoadSomeResource();
});

// 3. 메인 스레드는 위 두 작업이 모두 끝날 때까지 대기
wi::jobsystem::Wait(ctx);
```

### 2. 반복문 병렬 처리 (Dispatch)

배열이나 리스트의 요소들을 여러 스레드에 나눠서 병렬로 처리할 때 유용합니다.

```cpp
wi::jobsystem::context ctx;

uint32_t itemCount = 1000;
uint32_t groupSize = 100; // 한 잡이 처리할 단위 크기

// 1000개의 아이템을 100개씩 묶어서 10개의 Job으로 나누어 병렬 실행
wi::jobsystem::Dispatch(ctx, itemCount, groupSize, [](wi::jobsystem::JobArgs args) {
    // args.jobIndex: 전체 아이템 중 현재 처리할 아이템 인덱스
    // args.groupID: 잡 그룹 인덱스
    
    UpdateEntity(args.jobIndex);
});

wi::jobsystem::Wait(ctx);
```

### 3. 잡 내부에서 하위 잡 생성

잡 내부에서 또 다른 잡을 `Execute` 할 수 있습니다. 부모 `ctx`를 함께 넘기면, 부모 컨텍스트를 기다릴 때 자식 잡들까지 모두 재귀적으로 기다리게 됩니다.

## 잡 시스템과 그래픽스 (DX12) 커맨드 리스트

DX12(또는 Vulkan)에서는 커맨드 리스트(GraphicsCommandList) 기록을 여러 CPU 스레드에서 병렬로 수행할 수 있습니다. Wicked Engine은 이를 위해 `jobsystem`을 적극적으로 활용합니다.

**렌더링 병렬 처리 예시 (`Renderer.cpp` 등):**
```cpp
wi::jobsystem::context ctx;

// 워커 스레드 1: 메인 씬 물체들 렌더링 커맨드 기록
wi::jobsystem::Execute(ctx, [&](wi::jobsystem::JobArgs args) {
    CommandList cmd = device->BeginCommandList();
    DrawMainScene(cmd);
});

// 워커 스레드 2: UI 렌더링 커맨드 기록
wi::jobsystem::Execute(ctx, [&](wi::jobsystem::JobArgs args) {
    CommandList cmd = device->BeginCommandList();
    DrawUI(cmd);
});

wi::jobsystem::Wait(ctx);

// 병렬 기록이 모두 끝난 후, 생성된 커맨드 리스트들을 GPU(Queue)에 일괄 제출
device->SubmitCommandLists();
```

## apply_copy_allocator.md 에서의 jobsystem 역할

`WaitCommandList`는 **GPU 큐 간의 실행 순서를 보장**하기 위한 그래픽스 API 차원의 추상화 기능이고, 
`jobsystem`은 단순히 **CPU 상에서 C++ 함수의 실행을 여러 스레드에 분산**시키는 역할입니다.

* `jobsystem`: CPU 병렬 처리
* `WaitCommandList`: GPU 병렬/순차 제어

따라서 `jobsystem`으로 백그라운드 스레드에서 텍스처를 긴 시간 동안 로딩(디스크 I/O + VRAM 업로드 대기)하더라도, 메인 스레드(렌더 루프)는 블로킹되지 않고 계속 화면을 그려내어 유저에게 버벅임을 주지 않을 수 있습니다.
