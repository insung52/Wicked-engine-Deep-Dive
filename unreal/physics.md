# Physics Engine Integration Study

**목표**: 자체 엔진(VizMotive Engine / 회사 엔진)에 물리를 붙이기 전에,  
UE5(Chaos)와 Wicked Engine이 물리를 어떻게 통합했는지 분석하고 문서화한다.

소스 경로:
- UE5: `D:\graphics_d\unreals\UnrealEngine`
- Wicked Engine: `C:\graphics\wicked\origin92\WickedEngine`
- VizMotive Engine: `C:\graphics\vizmotive\my\VizMotive-Engine`

---

## 분석 관점 (3가지 핵심 질문)

어떤 엔진을 보든 이 3가지를 중심으로 트레이싱한다.

| 질문 | 핵심 내용 |
|------|-----------|
| **① 통합 경계** | 게임루프 어느 지점에서 물리 스텝을 호출하는가? |
| **② 데이터 흐름** | 물리 결과(transform, collision)가 어떻게 렌더링 오브젝트에 반영되는가? |
| **③ 스레딩 모델** | 물리가 어느 스레드에서 실행되고, 렌더와 어떻게 동기화하는가? |

---

## 진행 순서

```
Phase 1 — UE5 Chaos 통합 아키텍처 분석  (Claude 분석 → 내가 공부)
Phase 2 — Wicked Engine 물리 통합 분석   (Claude 분석 → 내가 공부)
Phase 3 — 비교 및 자체 엔진 적용 정리    (Claude 비교 문서 작성)
```

**UE5를 먼저 하는 이유:**
- 이전 Chaos 디버깅에서 핵심 파일들을 이미 파악함 (`ChaosScene.cpp`, `PhysicsSolverBase.cpp`, `PhysLevel.cpp`, `LevelTick.cpp`)
- 분석 난이도는 "엔진 크기"가 아니라 "볼 파일 범위"에 달려 있음 — 통합 아키텍처 트레이싱은 파일 수가 한정적
- Wicked Engine을 나중에 보면 "UE5 대비 어떻게 다른가" 관점으로 훨씬 빠르게 이해 가능

---

## Phase 1: UE5 Chaos 통합 아키텍처

### 분석 목표

Chaos 솔버 내부 수식/알고리즘이 아니라, **렌더링 엔진(UE5)이 물리 엔진(Chaos)을 어떻게 붙였는가** 의 구조를 파악한다.

### 분석 항목

#### 1-A. 물리 씬 생성 및 Actor 등록

- `UWorld` → `FPhysScene` 생성 경로
- Actor/Component가 물리 바디로 등록되는 과정
  - `UPrimitiveComponent::CreatePhysicsState()`
  - Chaos Particle 생성 및 Solver 등록

#### 1-B. 게임루프에서 물리 호출 (통합 경계)

이미 파악된 흐름:
```
UWorld::Tick()
  └ RunTickGroup(TG_StartPhysics)     ← 물리 스텝 시작
  └ RunTickGroup(TG_DuringPhysics)    ← 물리 실행 중 (비블로킹)
  └ RunTickGroup(TG_EndPhysics)       ← 물리 완료 대기
        └ ProcessUntilTasksComplete   ← 게임스레드 pump loop
```

추가로 파악할 것:
- `FStartPhysicsTickFunction` / `FEndPhysicsTickFunction` 의 전체 구현
- `FChaosScene::StartFrame()` → `AdvanceAndDispatch_External()` 호출 경로
- `IsUsingAsyncResults()` 분기 (동기/비동기 모드 차이)

#### 1-C. 물리 결과 → 렌더링 반영 (데이터 흐름)

- `FChaosScene::EndFrame()` 내부: `SyncBodies()` 동작
- Chaos Particle transform → `USceneComponent` transform 업데이트 경로
- `FPhysicsCommand::ExecuteWrite` / `FBodyInstance::SetBodyTransform`

#### 1-D. 스레딩 모델

이미 파악된 내용:
- `EThreadingMode::TaskGraph` (PIE 기본)
- `FPhysicsSolverAdvanceTask` → TaskGraph Worker에서 실행
- `RHIThread: WaitForTasks (66.8ms)` 동안 게임스레드가 BP 틱 직렬 처리

추가로 파악할 것:
- `EThreadingMode::DedicatedThread` 와 TaskGraph 모드의 차이
- `bTickPhysicsAsync` 옵션이 동기화 방식에 미치는 영향

#### 1-E. 차량/캐릭터 물리 특수 처리 (선택)

- `FChaosVehicleManagerAsyncCallback::OnPreSimulate_Internal` (이미 파악)
- 일반 RigidBody vs Vehicle vs Character(Capsule)의 처리 경로 차이

### 결과물

`C:\graphics\deepdive\Wicked-engine-Deep-Dive\unreal\ue5_physics_integration.md`

---

## Phase 2: Wicked Engine 물리 통합

### 분석 목표

Wicked Engine이 어떤 물리 라이브러리를 사용하는지 확인하고,  
Phase 1과 동일한 3가지 관점(통합 경계 / 데이터 흐름 / 스레딩)으로 트레이싱한다.

### 분석 항목

#### 2-A. 사용 라이브러리 파악

- CMakeLists.txt / 빌드 설정에서 물리 라이브러리 확인
- Bullet Physics, NVIDIA PhysX, 자체 구현 중 어느 것인지

#### 2-B. 게임루프에서 물리 호출 (통합 경계)

- Wicked Engine의 메인 루프 구조 파악
- `StepSimulation()` 또는 동등한 함수 호출 위치

#### 2-C. 물리 결과 → 렌더링 반영

- Wicked의 Scene/Entity 시스템에서 물리 transform 반영 경로

#### 2-D. 스레딩 모델

- Wicked Engine의 Job System과 물리 실행 관계

### 결과물

`C:\graphics\deepdive\Wicked-engine-Deep-Dive\unreal\wicked_physics_integration.md`

---

## Phase 3: 비교 및 자체 엔진 적용 정리

### 비교 문서

| 항목 | UE5 (Chaos) | Wicked Engine | 자체 엔진 추천 |
|------|-------------|---------------|---------------|
| 물리 라이브러리 | Chaos (자체) | ? | ? |
| 통합 경계 | TG_StartPhysics tick | ? | ? |
| 스레딩 | TaskGraph Worker | ? | ? |
| 동기화 | ProcessUntilTasksComplete | ? | ? |
| 결과 반영 | SyncBodies() | ? | ? |

### 결과물

`C:\graphics\deepdive\Wicked-engine-Deep-Dive\unreal\physics_comparison.md`

---

## 현재 상태

- [x] UE5 Chaos 스레딩 구조 파악 (chaos_vehicle_debug.md)
- [x] 30탱크 성능 병목 분석 완료
- [x] 물리 엔진 기초 개념 정리 → physics_basics.md  ← 여기서부터 읽기
- [x] Phase 1: UE5 통합 아키텍처 분석 → ue5_physics_integration.md
- [x] Phase 2: Wicked Engine 분석 → wicked_jolt_engine.md / wicked_physics_integration.md
- [x] Phase 3: 비교 문서 → physics_comparison.md
