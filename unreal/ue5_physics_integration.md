# UE5 Chaos 물리 통합 아키텍처

UE 5.7 소스 기준. 소스 경로: `D:\Epic games\UE_5.7\Engine\Source\Runtime\`  
분석 관점: 렌더링 엔진이 물리 엔진을 어떻게 붙였는가 (통합 경계 / 데이터 흐름 / 스레딩).

---

## 전체 흐름 요약

```
[① Actor 등록]
UPrimitiveComponent::OnCreatePhysicsState()
  → FBodyInstance::InitBody()
  → TInitBodiesHelperBase::InitBodies()
  → PhysScene->AddActorsToScene_AssumesLocked()   ← Chaos Solver에 등록

[② 매 프레임: 물리 시작]
LevelTick: RunTickGroup(TG_StartPhysics)
  → FStartPhysicsTickFunction::ExecuteTick()
  → UWorld::StartPhysicsSim()
  → FChaosScene::StartFrame()
  → Solver->AdvanceAndDispatch_External()          ← 물리 스텝 디스패치

[③ 물리 실행 (비동기)]
FPhysicsSolverAdvanceTask                          ← TaskGraph Worker에서 실행
  → Solver.AdvanceSolverBy()                       ← 실제 시뮬레이션 (STAT_ChaosTick)

[④ 매 프레임: 물리 완료 대기 + 결과 반영]
LevelTick: RunTickGroup(TG_EndPhysics)
  → FEndPhysicsTickFunction::ExecuteTick()
  → ProcessUntilTasksComplete (pump loop)          ← 물리 완료까지 다른 태스크 실행
  → UWorld::FinishPhysicsSim()
  → FChaosScene::EndFrame()
  → SyncBodies() → PullPhysicsStateForEachDirtyProxy_External()  ← Physics→GameThread 복사
```

---

## ① Actor → Chaos Solver 등록 경로

Actor가 월드에 배치되거나 BeginPlay가 호출될 때 물리 바디가 생성되고 Solver에 등록된다.

### 1단계: OnCreatePhysicsState

`Engine\Private\Components\PrimitiveComponent.cpp` line ~873

```cpp
void UPrimitiveComponent::OnCreatePhysicsState()
{
    Super::OnCreatePhysicsState();
    if (!BodyInstance.IsValidBodyInstance())
    {
        UBodySetup* BodySetup = GetBodySetup();
        if (BodySetup)
        {
            FTransform BodyTransform = GetComponentTransform();
            BodyInstance.InitBody(BodySetup, BodyTransform, this, GetWorld()->GetPhysicsScene());
        }
    }
}
```

- `UBodySetup`: 메시에 붙어있는 콜리전 형태 정의 (Box/Sphere/Capsule/Convex)
- `GetWorld()->GetPhysicsScene()`: 현재 월드의 `FChaosScene` 반환

### 2단계: InitBody → Static/Dynamic 분기

`Engine\Private\PhysicsEngine\BodyInstance.cpp` line ~1720

```cpp
void FBodyInstance::InitBody(UBodySetup* Setup, const FTransform& Transform,
    UPrimitiveComponent* PrimComp, FPhysScene* InRBScene, ...)
{
    if (bIsStatic)
    {
        FInitSingleBodyHelperStatic InitBodyHelper(...);
        InitBodyHelper.InitBodies();
    }
    else
    {
        FInitSingleBodyHelperDynamic InitBodyHelper(...);
        InitBodyHelper.InitBodies();   // ← 여기서 Chaos Particle 생성 및 등록
    }
}
```

Static(움직이지 않는 지형 등)과 Dynamic(탱크, 캐릭터 등)으로 분기됨.

### 3단계: InitBodies → Solver에 최종 등록

`Engine\Private\PhysicsEngine\BodyInstance.cpp` line ~1583

```cpp
void TInitBodiesHelperBase::InitBodies()
{
    CreateShapesAndActors();   // Chaos Shape + Actor 생성

    FPhysicsCommand::ExecuteWrite(PhysScene, [&]()
    {
        // Chaos Solver의 씬에 Actor 추가
        PhysScene->AddActorsToScene_AssumesLocked(ActorHandles);
        
        // Component ↔ Physics Actor 매핑 등록
        PhysScene->AddToComponentMaps(BI->OwnerComponent.Get(), ActorHandle);
        
        // 충돌 이벤트 구독 (bNotifyRigidBodyCollision 설정 시)
        if (BI->bNotifyRigidBodyCollision)
            LocalPhysScene->RegisterForCollisionEvents(PrimComp);
    });
}
```

`FPhysicsCommand::ExecuteWrite`: Physics 락을 잡고 Solver 내부 상태를 안전하게 수정하는 래퍼.

**등록 완료 후 상태:**
- Chaos 내부에 `FGeometryParticle` (Static) 또는 `FRigidBodyHandle` (Dynamic) 생성됨
- `ComponentMaps`: `UPrimitiveComponent* ↔ Chaos Particle Handle` 양방향 매핑 유지
- 이 매핑이 나중에 SyncBodies에서 `Physics → USceneComponent.transform` 복사에 사용됨

---

## ② 게임루프에서 물리 호출 (통합 경계)

### LevelTick.cpp의 TickGroup 순서

`Engine\Private\LevelTick.cpp`

```cpp
RunTickGroup(TG_PrePhysics);
RunTickGroup(TG_StartPhysics);         // ← 물리 시작 (블로킹)
RunTickGroup(TG_DuringPhysics, false); // ← 물리 실행 중 (비블로킹)
RunTickGroup(TG_EndPhysics);           // ← 물리 완료 대기 (블로킹)
RunTickGroup(TG_PostPhysics);
```

- `TG_DuringPhysics`의 `false` = `bBlockTillComplete=false` → 틱들이 큐에만 쌓이고 즉시 실행되지 않음
- `TG_EndPhysics`의 `ProcessUntilTasksComplete` pump loop가 물리 완료를 기다리는 동안 그 큐를 처리함

### FStartPhysicsTickFunction::ExecuteTick

`Engine\Private\PhysicsEngine\PhysLevel.cpp` line ~239

```cpp
void FStartPhysicsTickFunction::ExecuteTick(float DeltaTime, ...)
{
    Target->StartPhysicsSim();
}

void UWorld::StartPhysicsSim()
{
    FPhysScene* PhysScene = GetPhysicsScene();
    PhysScene->StartFrame();
}
```

### FChaosScene::StartFrame

`PhysicsCore\Private\ChaosScene.cpp` line ~363

```cpp
void FChaosScene::StartFrame()
{
    const float UseDeltaTime = OnStartFrame(MDeltaTime);

    for (FPhysicsSolverBase* Solver : GetPhysicsSolvers())
    {
        if (FGraphEventRef SolverEvent = Solver->AdvanceAndDispatch_External(UseDeltaTime))
        {
            CompletionEvents.Add(SolverEvent);   // 완료 이벤트 보관
        }
    }
}
```

`CompletionEvents`에 보관된 이벤트가 `FEndPhysicsTickFunction`에서 완료 대기에 사용됨.

---

## ③ 스레딩 모델

### EThreadingMode 3가지

```cpp
enum class EThreadingModeTemp : uint8
{
    DedicatedThread,  // 전용 물리 스레드 (단일, 낮은 레이턴시)
    TaskGraph,        // TaskGraph Worker (멀티코어 병렬, 높은 처리량)
    SingleThread      // 게임 스레드에서 동기 실행 (디버그용)
};
```

PIE 환경 기본값: **TaskGraph**  
(`FChaosScene::FChaosScene()` 생성자에서 멀티스레딩 가용 여부에 따라 결정)

### AdvanceAndDispatch_External의 모드 분기

`Chaos\Private\Chaos\Framework\PhysicsSolverBase.cpp` line ~432

```cpp
FGraphEventRef FPhysicsSolverBase::AdvanceAndDispatch_External(FReal InDt)
{
    // ... 시간 계산 ...

    while (FPushPhysicsData* PushData = MarshallingManager.StepInternalTime_External())
    {
        if (ThreadingMode == EThreadingModeTemp::SingleThread)
        {
            // 게임 스레드에서 즉시 동기 실행
            FAllSolverTasks ImmediateTask(*this, PushData);
            ImmediateTask.AdvanceSolver();
        }
        else  // TaskGraph 또는 DedicatedThread
        {
            // 3단계 Task 체인 생성
            PendingTasks = TGraphTask<FPhysicsSolverProcessPushDataTask>
                ::CreateTask(&Prereqs).ConstructAndDispatchWhenReady(*this, PushData);

            PendingTasks = TGraphTask<FPhysicsSolverFrozenGTPreSimCallbacks>
                ::CreateTask(&Prereqs).ConstructAndDispatchWhenReady(*this);

            PendingTasks = TGraphTask<FPhysicsSolverAdvanceTask>       // ← 실제 시뮬레이션
                ::CreateTask(&Prereqs).ConstructAndDispatchWhenReady(*this, PushData);

            if (!IsUsingAsyncResults())
            {
                BlockingTasks = PendingTasks;   // 이 프레임에서 즉시 대기
            }
        }
    }
    return BlockingTasks;
}
```

**DedicatedThread vs TaskGraph의 실제 차이:**
- `AdvanceAndDispatch_External` 코드 자체는 동일 — 둘 다 `TGraphTask`로 디스패치
- 차이는 TaskGraph *스케줄러* 레벨에서 발생: DedicatedThread 모드는 전용 물리 스레드가 해당 태스크들을 순서대로 소비, TaskGraph 모드는 일반 Worker들이 병렬 처리

### Task 실행 위치

```cpp
// PhysicsSolverBase.cpp
FAutoConsoleTaskPriority CPrio_FPhysicsTickTask(
    TEXT("TaskGraph.TaskPriorities.PhysicsTickTask"),
    ENamedThreads::HighThreadPriority,   // Hi-Pri Worker 우선
    ENamedThreads::NormalTaskPriority,
    ENamedThreads::HighTaskPriority      // 없으면 일반 Worker의 높은 우선순위
);
```

PIE(에디터)에서는 Hi-Pri Worker가 없으므로 일반 TaskGraph Worker에서 `HighTaskPriority`로 실행됨.

### IsUsingAsyncResults 와 동기화 방식

```
bTickPhysicsAsync = false (기본값)
  → AsyncDt = -1 → IsUsingAsyncResults() = false
  → BlockingTasks = PendingTasks (이 프레임 결과를 이 프레임에서 바로 대기)

bTickPhysicsAsync = true
  → IsUsingAsyncResults() = true
  → 이전 프레임 결과를 사용, 현 프레임 물리는 백그라운드에서 계속 실행
  → 1프레임 레이턴시 있지만 게임스레드 블로킹 감소
```

---

## ④ 물리 결과 → 렌더링 반영 (SyncBodies)

### FEndPhysicsTickFunction::ExecuteTick

`Engine\Private\PhysicsEngine\PhysLevel.cpp` line ~257

```cpp
void FEndPhysicsTickFunction::ExecuteTick(float DeltaTime, ...)
{
    FGraphEventArray PhysicsComplete = PhysScene->GetCompletionEvents();

    if (!PhysScene->IsCompletionEventComplete())
    {
        // 물리 완료되면 GameThread에서 FinishPhysicsSim 호출하도록 예약
        MyCompletionGraphEvent->DontCompleteUntil(
            FSimpleDelegateGraphTask::CreateAndDispatchWhenReady(
                FDelegate::CreateUObject(Target, &UWorld::FinishPhysicsSim),
                &PhysicsComplete,
                ENamedThreads::GameThread
            )
        );
    }
}
```

`ProcessUntilTasksComplete` pump loop: 물리 완료를 기다리는 동안 게임 스레드가 `ProcessOneOtherTask`로 다른 큐 태스크(TG_DuringPhysics 틱 등)를 실행. 이 구간에서 30대 탱크의 VehicleTick이 직렬 실행됨 (기존 분석 참고).

### FChaosScene::EndFrame → SyncBodies 체인

`PhysicsCore\Private\ChaosScene.cpp`

```cpp
void FChaosScene::EndFrame()                                  // line ~500
{
    CompletionEvents.Reset();

    for (FPhysicsSolverBase* Solver : GetPhysicsSolvers())
    {
        Solver->CastHelper([&](auto& Concrete)
        {
            SyncBodies(&Concrete);                            // line ~447
            Solver->FlipEventManagerBuffer();
            Concrete.SyncEvents_GameThread();
            Concrete.SyncQueryMaterials_External();
        });
    }
    OnPhysScenePostTick.Broadcast(this);
}

template<typename TSolver>
void FChaosScene::SyncBodies(TSolver* Solver)                 // line ~447
{
    OnSyncBodies(Solver);
}

void FChaosScene::OnSyncBodies(FPhysicsSolverBase* Solver)    // line ~389
{
    Solver->PullPhysicsStateForEachDirtyProxy_External(Dispatcher);
}
```

### PullPhysicsStateForEachDirtyProxy_External 내부

이 함수가 "물리 스레드 결과 → 게임 스레드 USceneComponent" 복사의 핵심.

```
PullPhysicsStateForEachDirtyProxy_External
  → DirtyProxyList를 순회 (변경된 Particle만)
  → 각 Proxy: Physics Thread Buffer → Game Thread Buffer 로 Transform/Velocity 복사
  → FBodyInstance::MoveBodyCallback 또는 SetBodyTransform 호출
  → USceneComponent::SetWorldTransform()   ← 렌더링에 반영될 최종 transform 업데이트
```

**중요**: "Dirty Proxy"만 처리 → 정지한 물체는 매 프레임 복사 불필요.  
`ComponentMaps`를 통해 `Chaos Particle ↔ UPrimitiveComponent` 매핑이 이 단계에서 사용됨.

---

## 전체 데이터 흐름 다이어그램

```
[에디터/런타임 초기화]
UBodySetup (콜리전 형태)
  → FBodyInstance::InitBody()
  → Chaos FGeometryParticle / FRigidBodyHandle 생성
  → FChaosScene::AddActorsToScene_AssumesLocked()
  → ComponentMaps 등록 (UPrimitiveComponent ↔ Particle Handle)

[매 프레임]
                    Game Thread                          Physics Worker
                        │                                      │
TG_StartPhysics         │                                      │
StartFrame()   ─────────┼──── AdvanceAndDispatch ──────────►   │
                        │     (Task 디스패치)                   │ AdvanceSolverBy()
                        │                                      │ (STAT_ChaosTick, 2.63ms)
TG_DuringPhysics        │                                      │
(bBlock=false)          │  VehicleTick×30 실행 (73ms)           │ 시뮬레이션 중
                        │  SetTracksTransform                  │
                        │  (pump loop에서 처리)                 │
                        │                                      │
TG_EndPhysics           │                                      │
ProcessUntilTasksComplete ◄──── 완료 이벤트 ──────────────────   │
FinishPhysicsSim()      │                                      │
EndFrame()              │                                      │
  SyncBodies()          │                                      │
  PullDirtyProxy ───────┼── Physics Buffer → Game Buffer       │
  SetWorldTransform     │   (USceneComponent transform 업데이트)│
                        │                                      │
TG_PostPhysics          │                                      │
(렌더링 준비)             │                                      │
```

---

## 핵심 요약 (자체 엔진 적용 관점)

| 항목 | UE5 방식 | 자체 엔진 적용 시 고려 |
|------|----------|----------------------|
| **등록** | `CreatePhysicsState()` → `InitBody()` → `AddActorsToScene()` | 렌더 오브젝트 생성 시점에 물리 바디 등록 함수 호출 |
| **통합 경계** | `TG_StartPhysics` Tick Function | 게임루프 한 지점에서 `PhysicsStep(dt)` 호출 |
| **스레딩** | TaskGraph Worker (비동기 디스패치) | 별도 스레드 또는 메인 루프 내 순차 실행부터 시작 |
| **동기화** | `ProcessUntilTasksComplete` + `SyncBodies()` | 물리 완료 후 transform 복사 단계 필수 |
| **결과 반영** | `PullPhysicsStateForEachDirtyProxy` → `SetWorldTransform` | Dirty 목록 관리로 불필요한 복사 최소화 |
| **매핑** | `ComponentMaps (Component ↔ ParticleHandle)` | 렌더 오브젝트 ↔ 물리 바디 양방향 핸들 유지 |
