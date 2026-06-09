# 렌더링 엔진 물리 통합 분석

**목표:** 자체 엔진(VizMotive / 회사 엔진)에 물리를 붙이기 전,  
UE5(Chaos)와 Wicked Engine(Jolt)이 물리를 어떻게 통합했는지 분석하고 적용 방향 도출.

분석 관점 3가지 — 모든 문서가 이 틀을 기준으로 작성됨:

| 관점 | 핵심 질문 |
|------|-----------|
| **통합 경계** | 게임루프 어디서 물리 스텝을 호출하는가? |
| **데이터 흐름** | 물리 결과가 렌더링 오브젝트에 어떻게 전달되는가? |
| **스레딩** | 물리가 어느 스레드에서 실행되고 렌더와 어떻게 동기화하는가? |

---

## 문서 맵

아래 순서로 읽는 것을 권장. 각 문서는 독립적으로 읽을 수 있지만, 앞 문서의 개념을 전제로 함.

```
① physics_basics.md          ← 물리 엔진 기초 (CPU/GPU, 강체, 충돌, PBD vs 임펄스)
        │
        ▼
② chaos_engine.md             ← UE5의 물리 엔진 Chaos 내부 구조
③ ue5_physics_integration.md  ← UE5가 Chaos를 붙인 방법 (소스 코드 분석)
        │
        ▼
④ wicked_jolt_engine.md       ← Wicked Engine의 물리 엔진 Jolt 내부 구조
⑤ wicked_physics_integration.md ← Wicked Engine이 Jolt를 붙인 방법 (소스 코드 분석)
        │
        ▼
⑥ physics_comparison.md       ← 이 문서: 두 엔진 비교 + 자체 엔진 적용 방향
        │
        ▼
⑦ chaos_vehicle_debug.md      ← 실전 케이스: 30탱크 성능 병목 분석 및 최적화
```

| 문서 | 내용 |
|------|------|
| [physics_basics.md](physics_basics.md) | 물리 엔진 기초 개념. CPU vs GPU, SIMD, 강체/충돌/솔버/적분, 게임루프에서의 위치 |
| [chaos_engine.md](chaos_engine.md) | Chaos가 뭔지. PBD 알고리즘, Particle/SOA/Island 구조, 모듈 구성, Chaos vs Jolt |
| [ue5_physics_integration.md](ue5_physics_integration.md) | UE5 통합 아키텍처. Actor 등록 경로, TickGroup 흐름, SyncBodies, 스레딩 모드 |
| [wicked_jolt_engine.md](wicked_jolt_engine.md) | Jolt가 뭔지. 임펄스 기반 알고리즘, Body/Shape/Constraint, Island, JobSystem |
| [wicked_physics_integration.md](wicked_physics_integration.md) | Wicked 통합 아키텍처. ECS 구조, 고정 타임스텝 축적기, 결과 반영, 보간 |
| [chaos_vehicle_debug.md](chaos_vehicle_debug.md) | 30탱크 성능 분석. SetTracksTransform 병목, GPU 이전 시 개선 추정 |
| [chaos.md](chaos.md) | stat unit / Insights 측정 방법론 레퍼런스 |

---

## 1. 물리 엔진 기초

> 상세 내용: [physics_basics.md](physics_basics.md)

물리 엔진은 **"매 프레임 물체들의 미래 위치를 계산해주는 소프트웨어"**. 렌더링 엔진은 물리 엔진이 알려준 위치에 그림만 그림.

```
렌더링 엔진 ──① 물체 등록──► 물리 엔진
            ◄─② 물리 스텝─── (내부: 충돌감지 → 솔버 → 적분)
            ──③ 결과 읽기──► transform 업데이트
            ──④ 렌더링
```

**CPU vs GPU**: 강체 물리는 CPU. 데이터 의존성(충돌 체인)과 즉각적인 게임 로직 연결이 필요하기 때문. 파티클/천/유체는 독립적이라 GPU 적합.

---

## 2. 물리 엔진 비교: Chaos vs Jolt

> 상세 내용: [chaos_engine.md](chaos_engine.md) / [wicked_jolt_engine.md](wicked_jolt_engine.md)

### 핵심 알고리즘 차이

| | Chaos (UE5) | Jolt (Wicked) |
|--|-------------|---------------|
| **방식** | PBD (Position-Based Dynamics) | 임펄스 기반 (Impulse-Based) |
| **충돌 반응** | 위치를 직접 수정 후 속도 역산 | 임펄스로 속도 수정 후 위치 적분 |
| **에너지 보존** | 근사 (반복 수에 의존) | 더 정확 |
| **복잡한 제약 안정성** | 유리 (위치 기반이라 발산 없음) | 특정 설정에서 불안정 가능 |

```
Chaos (PBD):
  예측 P = X + V*dt → 충돌 시 P를 직접 바깥으로 끌어당김 → V = (P-X)/dt 역산

Jolt (Impulse):
  충돌 감지 → 임펄스 J 계산 → V += J/m → X += V*dt
```

자세한 PBD vs 임펄스 비교, 여러 제약이 겹칠 때의 차이는 → [chaos_engine.md](chaos_engine.md) 참고.

### 엔진 전반 비교

| 항목 | Chaos (UE5) | Jolt (Wicked) |
|------|-------------|---------------|
| 제작 | Epic Games (자체) | Guerrilla → 오픈소스 |
| 라이선스 | UE 종속 | **MIT** |
| 메모리 구조 | **SOA** (타입별 배열, SIMD 최적화) | 일반 구조체 |
| Island 병렬화 | 있음 | 있음 |
| 스레딩 인터페이스 | UE TaskGraph 종속 | **JobSystem 추상화** (교체 가능) |
| Soft Body | 있음 | 있음 |
| 파괴(Destruction) | **주력 기능** | 없음 |
| 성숙도 | 신생, 버그 많음 | AAA 실전 검증, 안정적 |
| SIMD | IntelISPC | 자체 Vec4/Mat44 인트린직 |

---

## 3. 렌더링 엔진 통합 방식 비교

> 상세 내용: [ue5_physics_integration.md](ue5_physics_integration.md) / [wicked_physics_integration.md](wicked_physics_integration.md)

### 3-1. 통합 경계 — 게임루프 어디서 물리를 호출하는가

**UE5: TickGroup 3단계 분리**

```
LevelTick.cpp:
  RunTickGroup(TG_StartPhysics)          ← FChaosScene::StartFrame() → 물리 비동기 디스패치
  RunTickGroup(TG_DuringPhysics, false)  ← 물리 실행 중, BP 틱들이 큐에 쌓임
  RunTickGroup(TG_EndPhysics)            ← ProcessUntilTasksComplete로 완료 대기 + 쌓인 틱 처리
```

물리 시작과 완료가 분리되어 `TG_DuringPhysics` 구간에 다른 작업을 겹칠 수 있음. 단, BP 틱이 게임 스레드에서 직렬 실행되는 부작용 존재 → [chaos_vehicle_debug.md](chaos_vehicle_debug.md) 참고.

**Wicked Engine: 단일 함수**

```cpp
// wiScene.cpp line 258
wi::physics::RunPhysicsUpdateSystem(ctx, *this, dt);
// 한 함수 안에서 등록 확인 → 동기화 → Jolt.Update() → 결과 반영 순서대로 처리
```

구조가 단순하고 추적하기 쉬움. 물리 완료 전에 다른 작업을 겹치는 것은 불가.

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 구조 | TickGroup 3단계 | 단일 함수 |
| 물리 중 게임 로직 병렬 실행 | 가능 (TG_DuringPhysics) | 불가 |
| 코드 추적 난이도 | 높음 | **낮음** |

---

### 3-2. 오브젝트 등록 — 물리 바디가 언제 어떻게 생성되는가

**UE5**: Actor 생성 시 즉시

```
Actor BeginPlay
  → UPrimitiveComponent::OnCreatePhysicsState()
  → FBodyInstance::InitBody()  (Static/Dynamic 분기)
  → PhysScene->AddActorsToScene_AssumesLocked()  ← Chaos Solver 등록
  → ComponentMaps 구축 (Component ↔ Particle Handle 양방향)
```

자세한 코드 경로 → [ue5_physics_integration.md § Actor 등록](ue5_physics_integration.md)

**Wicked Engine**: 첫 Update 루프에서 Lazy 등록

```cpp
// wiPhysics_Jolt.cpp
if (physicscomponent.physicsobject == nullptr || IsRefreshParametersNeeded()) {
    AddRigidBody(scene, entity, physicscomponent, *transform, mesh);
    // → Jolt BodyInterface.CreateAndAddBody() → bodyID 보관
}
```

자세한 코드 경로 → [wicked_physics_integration.md § 등록](wicked_physics_integration.md)

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 등록 시점 | Actor 생성 즉시 | 첫 Update에서 lazy |
| 락 방식 | FPhysicsCommand::ExecuteWrite | Jolt BodyInterface 내부 락 |

---

### 3-3. 타임스텝 — 물리를 프레임 시간에 맞추는가

**UE5**: 렌더 dt 그대로 물리에 전달 (기본값)

```
60fps → 물리 dt = 16ms
30fps → 물리 dt = 33ms   ← 프레임레이트 따라 결과가 달라질 수 있음
```

**Wicked Engine**: 고정 타임스텝 축적기

```cpp
// wiPhysics_Jolt.cpp
static constexpr float TIMESTEP = 1.0f / 60.0f;

physics_scene.accumulator += dt;
while (physics_scene.accumulator >= TIMESTEP) {
    physics_system.Update(TIMESTEP, COLLISION_STEPS, &temp_allocator, &job_system);
    physics_scene.accumulator -= TIMESTEP;
}
physics_scene.alpha = physics_scene.accumulator / TIMESTEP;  // 보간 가중치
```

프레임레이트와 무관하게 항상 1/60초 단위로 시뮬레이션 → 결정론적(Deterministic) 보장. 렌더 프레임 사이의 중간 결과는 `alpha`로 선형 보간.

자세한 내용 → [wicked_physics_integration.md § 고정 타임스텝](wicked_physics_integration.md)

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 방식 | 렌더 dt 그대로 | **고정 1/60s 축적기** |
| 결정론적 | 프레임레이트 의존 | **보장** |
| 보간 | 선택 (bTickPhysicsAsync) | 기본 제공 |

---

### 3-4. 데이터 흐름 — 물리 결과가 렌더 오브젝트에 어떻게 전달되는가

**UE5: Dirty Proxy 방식**

```
Chaos Solver 완료 → 변경된 Particle만 DirtyList 마킹
EndFrame() → SyncBodies() → PullPhysicsStateForEachDirtyProxy_External()
  → ComponentMaps로 UPrimitiveComponent 탐색
  → SetWorldTransform()  ← 렌더링이 읽을 transform 업데이트
```

정지한 물체는 Dirty가 없으므로 복사 스킵 → 대규모에 유리. 자세한 코드 → [ue5_physics_integration.md § SyncBodies](ue5_physics_integration.md)

**Wicked Engine: 전체 읽기 방식**

```cpp
// wiPhysics_Jolt.cpp — 매 프레임 모든 RigidBody 읽기 (병렬)
Mat44 mat = body_interface.GetWorldTransform(physicsobject.bodyID);
// 보간 적용 후
transform->translation_local = cast(position);
transform->rotation_local    = cast(rotation);
```

자세한 코드 → [wicked_physics_integration.md § 결과 반영](wicked_physics_integration.md)

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 방식 | **Dirty Particle만 복사** | 전체 매 프레임 읽기 |
| 정지 물체 비용 | 0 (스킵) | 매 프레임 읽기 |
| 보간 위치 | 별도 async 모드 | 결과 복사 시 인라인 |

---

### 3-5. 스레딩 모델

**UE5**

```
StartFrame() → TGraphTask<FPhysicsSolverAdvanceTask> 생성 → TaskGraph Worker에서 실행
                                                              (HighThreadPriority, 낙오 시 HighTaskPriority)
GameThread → ProcessUntilTasksComplete pump loop → 완료 대기하면서 다른 태스크 실행
완료 → FinishPhysicsSim() → SyncBodies()
```

UE TaskGraph에 완전 종속. 다른 엔진에서 재사용 불가. 자세한 내용 → [ue5_physics_integration.md § 스레딩](ue5_physics_integration.md)

**Wicked Engine**

```cpp
physics_system.Update(TIMESTEP, COLLISION_STEPS, &temp_allocator, &job_system);
//                                                                  ↑
//                                              wi::jobsystem 어댑터 (Jolt JobSystem 인터페이스 구현)
```

Jolt의 `JobSystem`은 추상 인터페이스 → 엔진의 스레드 시스템을 어댑터로 연결. UE TaskGraph든 자체 스레드풀이든 교체 가능. 자세한 내용 → [wicked_physics_integration.md § 스레딩](wicked_physics_integration.md)

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 스레딩 방식 | UE TaskGraph 종속 | **JobSystem 추상화** |
| 다른 엔진 이식성 | 없음 | **높음** |

---

### 3-6. 씬 구조

**UE5: Actor-Component 계층**

```
UWorld → AActor → UPrimitiveComponent → FBodyInstance → Chaos Particle Handle
```

**Wicked Engine: ECS (Entity Component System)**

```
Scene
  ├ ComponentManager<TransformComponent>
  ├ ComponentManager<RigidBodyPhysicsComponent>  (physicsobject = Jolt BodyID)
  └ ComponentManager<MeshComponent>
Entity = 정수 ID. 같은 ID로 각 Manager에서 조회.
```

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 구조 | Actor-Component 계층 (깊음) | **ECS flat** |
| 물리-렌더 연결 | ComponentMaps 양방향 | Entity ID 직접 조회 |
| 캐시 효율 | 보통 | **높음** |

---

## 4. 실전 케이스: 30탱크 성능 분석

> 전체 분석: [chaos_vehicle_debug.md](chaos_vehicle_debug.md)

### 발견 요약

UE5에서 BP_M1A2 탱크 30대를 동시 시뮬레이션할 때 8 FPS (~122ms) 병목 발생.

**프레임 시간 분해 (30대, ~122ms):**

```
총 122ms
  ├─ VehicleTick Blueprint      73ms  (60%)  ← 게임 스레드 직렬
  │     ├─ SetTracksTransform   52ms  (43%)  ← 핵심 병목
  │     │     └─ FinalDistanceCalculation 18ms
  │     ├─ VehicleMesh          25ms  (20%)
  │     └─ NS_SkidMarks         18ms  (15%)
  ├─ Chaos 물리 솔버             3ms  ( 2%)  ← Worker 스레드 (병목 아님)
  └─ 렌더링 + 기타              46ms  (38%)  ← Render/RHI 스레드 (병목 아님)
```

**구조적 원인:**
- `TG_DuringPhysics` (bBlock=false) → 30대 BP 틱이 큐에 쌓임
- `ProcessUntilTasksComplete` pump loop → 게임 스레드에서 하나씩 직렬 처리
- Blueprint VM이 게임 스레드 전용 → Worker 분산 불가
- `USplineComponent`가 CPU 전용 → 157개 트랙 링크 위치를 매 프레임 CPU에서 계산

**핵심**: 트랙 링크는 순수 비주얼 요소(물리 충돌 없음)인데, BP + USplineComponent 구조 때문에 CPU 게임 스레드에서 직렬 실행 중.

### 최적화 방향

| 방법 | 절감 | 난이도 | 비고 |
|------|------|--------|------|
| 거리 기반 트랙 LOD | ~35ms | 낮음 | BP 수정만으로 즉시 적용 |
| UV 스크롤 (GPU Material) | ~40ms | 중간 | 트랙 질감 단순화 |
| Vertex Shader (GPU) | **~52ms** | 높음 | 157 링크 → GPU 병렬, CPU 비용 ~0ms |
| C++ + ParallelFor | ~70ms | 높음 | 전면 재작성, 근본 해결 |
| NS_SkidMarks LOD | ~10ms | 낮음 | 원거리 파티클 비활성화 |

**GPU 이전 시 예상:**  
SetTracksTransform 52ms 절감 → 122ms → ~70ms → **8 FPS → 14~20 FPS**  
(게임 스레드 병목 해소 시 현재 유휴 중인 GPU 66ms도 활용 가능)

---

## 5. 전체 비교 요약

```
                    UE5 + Chaos              Wicked Engine + Jolt
                    ──────────────────       ────────────────────
물리 알고리즘       PBD (위치 기반)           임펄스 기반
등록 시점          Actor 생성 즉시           첫 Update에서 lazy
통합 경계          TickGroup 3단계           단일 함수
타임스텝           렌더 dt 그대로            고정 1/60s 축적기
결과 복사          Dirty Particle만           전체 매 프레임
스레딩            UE TaskGraph 종속         JobSystem 추상화 (교체 가능)
씬 구조           Actor-Component 계층       ECS flat
이식성            낮음 (UE 전용)            높음
구현 복잡도        높음                      낮음
파괴 시스템        강력                      없음
에너지 보존        근사                      정확
라이선스          UE 종속                   MIT
```

---

## 6. 자체 엔진 적용 방향

### 물리 라이브러리 선택: Jolt 추천

- **MIT 라이선스** — 제한 없음
- **JobSystem 추상화** — 자체 스레드 시스템에 어댑터만 작성하면 연결 가능
- **실전 레퍼런스** — Wicked Engine의 `wiPhysics_Jolt.cpp`가 완성된 글루 코드 예시
- **안정성** — Chaos보다 성숙하고 버그 적음

### 통합 순서 (단계별)

```
Step 1: Jolt 초기화 + 단순 강체 테스트
  PhysicsSystem 생성 → TempAllocator, JobSystemThreadPool 연결
  Box Body 하나 추가 → Update() → 위치 읽기 확인

Step 2: 씬 연결
  렌더 오브젝트 ↔ Jolt BodyID 매핑 테이블 구성
  생성/제거 시 Body 등록/해제

Step 3: 고정 타임스텝 축적기 구현
  accumulator += render_dt
  while(accumulator >= 1/60) { Update(); accumulator -= 1/60; }
  alpha = accumulator / (1/60)  // 보간 가중치

Step 4: 결과 반영
  Dynamic: GetWorldTransform(bodyID) → 렌더 Transform 업데이트
  Kinematic: 렌더 Transform → MoveKinematic() (반대 방향)
```

### 최소 구현 코드 구조

```cpp
// 초기화
PhysicsSystem physics_system;
physics_system.Init(maxBodies, maxBodyPairs, maxContacts, ...);

// 매 프레임
accumulator += dt;
while (accumulator >= FIXED_DT) {
    physics_system.Update(FIXED_DT, collisionSteps, &tempAllocator, &jobSystem);
    accumulator -= FIXED_DT;
}
float alpha = accumulator / FIXED_DT;

// 결과 읽기 + 보간
for (auto& [entityID, bodyID] : rigidBodyMap) {
    Vec3 pos  = Lerp(prevPos,  bodyInterface.GetPosition(bodyID),  alpha);
    Quat rot  = Slerp(prevRot, bodyInterface.GetRotation(bodyID),  alpha);
    scene.SetTransform(entityID, pos, rot);
}
```

`wiPhysics_Jolt.cpp` → [wicked_physics_integration.md](wicked_physics_integration.md) 가 이 구조의 완성된 레퍼런스.
