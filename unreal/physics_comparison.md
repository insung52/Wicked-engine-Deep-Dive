# Physics Integration 비교: UE5 + Chaos vs Wicked Engine + Jolt

Phase 3 — 두 엔진의 물리 통합 방식 비교 및 자체 엔진 적용 방향.

---

## 물리 엔진 자체 비교

| 항목 | Chaos (UE5) | Jolt (Wicked) |
|------|-------------|---------------|
| 제작 | Epic Games (자체) | Guerrilla Games → 오픈소스 |
| 라이선스 | UE 종속 | **MIT** (완전 자유) |
| 알고리즘 | **PBD** (위치 기반) | **임펄스 기반** |
| 충돌 감지 | CPU (AABBTree BVH, GJK) | CPU (QuadTree BVH, GJK) |
| SIMD | IntelISPC | 자체 Vec4/Mat44 인트린직 |
| Island 병렬화 | 있음 | 있음 |
| 스레딩 인터페이스 | UE TaskGraph 종속 | **JobSystem 추상화** (교체 가능) |
| Soft Body | 있음 | 있음 |
| Ragdoll | 있음 | 있음 |
| Vehicle | ChaosVehicles 플러그인 | VehicleConstraint (내장) |
| 파괴(Destruction) | **주력 기능** | 없음 |
| 성숙도 | 비교적 신생, 버그 많음 | AAA 실전 검증, 안정적 |
| 에너지 보존 | 근사 (반복 수 의존) | 더 정확 |

### 알고리즘 차이 한 줄 요약

```
Chaos (PBD):    충돌 시 위치를 직접 이동시켜 보정 → 속도는 역산
Jolt (Impulse): 충돌 시 임펄스로 속도를 변경    → 위치는 적분 (일반적인 방법)
```

복잡한 제약(관절, 차량 서스펜션)에서:
- PBD: 위치 공간에서 반복 수렴 → 발산 없음, 에너지 손실 있음
- 임펄스: 속도 공간에서 반복 수렴 → 에너지 보존 정확, 특정 설정에서 불안정 가능

---

## 통합 경계 비교

"렌더링 엔진이 물리 스텝을 언제, 어떻게 호출하는가"

### UE5: TickGroup 시스템

```
LevelTick.cpp:
  RunTickGroup(TG_PrePhysics)
  RunTickGroup(TG_StartPhysics)          ← StartFrame() → 물리 비동기 디스패치
  RunTickGroup(TG_DuringPhysics, false)  ← 물리 실행 중 (비블로킹, 틱 큐에 쌓임)
  RunTickGroup(TG_EndPhysics)            ← 물리 완료 대기 + 쌓인 틱 처리
  RunTickGroup(TG_PostPhysics)
```

- 물리 시작과 완료 대기가 **분리**된 구조
- `TG_DuringPhysics` 구간 동안 게임 로직(AI, 애니메이션 등) 병렬 실행 가능
- 단, `ProcessUntilTasksComplete` pump loop에서 BP 틱이 직렬 실행되는 부작용 존재 (30탱크 73ms 병목)

### Wicked Engine: 단일 함수

```
wiScene.cpp:
  wi::jobsystem::Wait(ctx);                        // 애니메이션 완료 대기
  wi::physics::RunPhysicsUpdateSystem(ctx, scene, dt);  // ← 물리 전체 처리
  wi::jobsystem::Wait(ctx);
  UpdateTransforms(ctx);
```

- 물리 시작~완료~결과 반영을 **하나의 함수 안에서** 순서대로 처리
- 구조가 단순하고 추적하기 쉬움
- 물리가 완전히 끝나기 전에 다른 작업을 겹칠 수 없음

### 비교

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 구조 | TickGroup 분리 (Start / During / End) | 단일 함수 |
| 물리 중 게임 로직 병렬 실행 | 가능 (TG_DuringPhysics) | 불가 |
| 코드 추적 난이도 | 높음 (TickFunction, 이벤트 체인) | **낮음** (함수 한 개) |
| 자체 엔진 적용 난이도 | 높음 | **낮음** |

---

## 타임스텝 비교

"물리를 프레임 시간에 맞추는가, 고정 스텝으로 나누는가"

### UE5: 렌더 프레임에 맞춤 (기본)

```
렌더 dt → 그대로 Solver에 전달 (bTickPhysicsAsync=false 기본)

30fps → dt = 33ms → 물리 1스텝 33ms
60fps → dt = 16ms → 물리 1스텝 16ms
120fps → dt = 8ms → 물리 1스텝 8ms

프레임레이트가 달라지면 물리 결과도 달라질 수 있음
```

### Wicked Engine: 고정 타임스텝 축적기

```cpp
TIMESTEP = 1/60 초

accumulator += render_dt

while (accumulator >= TIMESTEP) {
    physics.Update(TIMESTEP);    // 항상 1/60초 단위로만 실행
    accumulator -= TIMESTEP;
}
alpha = accumulator / TIMESTEP; // 보간 가중치
```

```
30fps 렌더:  매 프레임 물리 2스텝 실행 (1/60 × 2 = 1/30)
60fps 렌더:  매 프레임 물리 1스텝 실행
120fps 렌더: 2프레임마다 물리 1스텝 실행 → 보간으로 부드럽게
```

결과: **프레임레이트와 무관하게 동일한 물리 시뮬레이션** 보장.

### 비교

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 방식 | 렌더 dt 그대로 | **고정 타임스텝 축적기** |
| 결정론적(Deterministic) | 프레임레이트 의존 | **보장** |
| 구현 복잡도 | 낮음 | 약간 높음 (accumulator 관리) |
| 보간 | 선택 (bTickPhysicsAsync) | 기본 제공 |
| 물리 정확도 | 가변 dt에 의존 | 일정 |

---

## 데이터 흐름 비교

"물리 결과가 렌더링 오브젝트에 어떻게 전달되는가"

### UE5: Dirty Proxy 방식

```
Chaos Solver 실행
  → 변경된 Particle을 DirtyList에 마킹

SyncBodies()
  → DirtyList만 순회
  → Physics Buffer → Game Thread Buffer 복사
  → ComponentMaps로 UPrimitiveComponent 찾기
  → SetWorldTransform() 호출

정지한 물체 = Dirty 없음 = 복사 스킵 (최적화)
```

### Wicked Engine: 전체 읽기 방식

```
Jolt.Update() 완료
  → 모든 RigidBody를 병렬로 순회
  → body_interface.GetWorldTransform(bodyID)
  → 보간 적용
  → transform->translation_local, rotation_local 업데이트
```

### 비교

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 방식 | **Dirty Particle만 복사** | 전체 읽기 |
| 정지 물체 비용 | 0 (스킵) | 매 프레임 읽기 |
| 구현 복잡도 | 높음 (Dirty 마킹 시스템 필요) | **낮음** |
| 보간 위치 | 별도 async 모드 | **결과 복사 시 인라인** |

물체 수가 매우 많으면 UE5의 Dirty 방식이 유리. 수백 개 수준이면 Wicked 방식도 충분.

---

## 스레딩 모델 비교

### UE5

```
물리 Task → UE TaskGraph Worker (Hi-Pri → 일반 fallback)
게임 Thread → ProcessUntilTasksComplete pump loop로 대기하면서 다른 태스크 실행

구조:
  GameThread ──→ AdvanceAndDispatch() ──→ [TaskGraph Worker] FPhysicsSolverAdvanceTask
      │                                           │
      │  ProcessUntilTasksComplete                │
      │  (다른 태스크 실행하며 대기)              │ 시뮬레이션 중
      │                                           │
      ←──────────────────────────────────────────  완료 이벤트
      FinishPhysicsSim() → SyncBodies()
```

- UE TaskGraph에 **완전히 종속** → 다른 엔진에서 재사용 불가
- Hi-Pri Worker 존재 여부에 따라 실행 위치가 달라지는 복잡성

### Wicked Engine

```
Jolt.Update(dt, steps, &allocator, &job_system)
                                       ↑
                              Wicked의 wi::jobsystem 어댑터
                              (Jolt JobSystem 인터페이스 구현)

Jolt 내부에서 Job 의존성 그래프 생성:
  ApplyGravity → FindCollisions(병렬) → SolveVelocity(병렬) → Integrate → SolvePosition(병렬)
```

- Jolt의 `JobSystem`은 **추상 인터페이스** → 엔진의 스레드 시스템을 어댑터로 연결
- UE TaskGraph든, 자체 스레드풀이든, std::thread든 연결 가능

### 비교

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 스레딩 방식 | UE TaskGraph 종속 | **JobSystem 어댑터 (교체 가능)** |
| 다른 엔진 이식성 | 없음 | **높음** |
| 물리 중 게임 작업 | pump loop에서 처리 | 물리 완료 후 처리 |
| 구조 파악 난이도 | 높음 | **낮음** |

---

## 씬 구조 비교

### UE5: Actor-Component 계층

```
UWorld
  └ ULevel
      └ AActor (BP_M1A2)
          └ USkeletalMeshComponent
              └ FBodyInstance            ← 물리-렌더 연결점
                  └ Chaos Particle Handle
```

- 물리 바디가 Component 안에 캡슐화
- `ComponentMaps`로 양방향 매핑 유지
- 계층 구조가 깊어서 추적이 복잡

### Wicked Engine: ECS (Entity Component System)

```
Scene
  ├ ComponentManager<TransformComponent>
  ├ ComponentManager<RigidBodyPhysicsComponent>  ← physicsobject(BodyID) 보관
  ├ ComponentManager<MeshComponent>
  └ ComponentManager<...>

Entity = 정수 ID.
같은 ID를 각 ComponentManager에서 조회하면 해당 Entity의 컴포넌트를 얻음.
```

- 물리/렌더링/트랜스폼이 **같은 레벨**에 나란히 존재
- 순회 시 cache-friendly (같은 타입의 컴포넌트가 메모리에 연속)
- 구조가 flat해서 추적하기 쉬움

### 비교

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 구조 | Actor → Component → BodyInstance 계층 | **ECS (flat)** |
| 물리-렌더 연결 | ComponentMaps 양방향 매핑 | Entity ID로 직접 조회 |
| 캐시 효율 | 보통 | **높음** |
| 추가/제거 편의성 | Actor 시스템 따름 | 컴포넌트 단독 추가/제거 |

---

## 오브젝트 등록 비교

| 항목 | UE5 | Wicked |
|------|-----|--------|
| 시점 | Actor 생성 즉시 (`OnCreatePhysicsState`) | **첫 Update 루프에서 lazy 등록** |
| 경로 | `InitBody()` → `AddActorsToScene_AssumesLocked()` | `AddRigidBody()` → `BodyInterface.CreateAndAddBody()` |
| 락 방식 | `FPhysicsCommand::ExecuteWrite` (Physics 락) | `BodyInterface` (Jolt 내부 락) |

---

## 전체 비교 요약

```
                    UE5 + Chaos              Wicked + Jolt
                    ──────────               ─────────────
물리 알고리즘       PBD (위치 기반)           임펄스 기반
통합 경계          TickGroup 3단계           단일 함수
타임스텝           렌더 dt 그대로            고정 1/60s 축적기
보간              선택적 (async 모드)        기본 제공
결과 복사          Dirty Particle만          전체 매 프레임
스레딩            UE TaskGraph 종속         JobSystem 추상화
씬 구조           Actor-Component 계층       ECS flat
이식성            낮음 (UE 전용)            높음 (어댑터 교체)
구현 복잡도        높음                      낮음
파괴 시스템        강력 (주력)               없음
라이선스          UE 종속                   MIT
```

---

## 자체 엔진 적용 방향

### 물리 라이브러리 선택

**Jolt 추천** (자체 엔진 기준):
- MIT 라이선스 — 상업/비상업 제한 없음
- JobSystem이 추상화돼 있어서 자체 스레드 시스템에 연결하기 쉬움
- Godot 4, Wicked Engine 등 실제 적용 레퍼런스 존재
- Chaos보다 성숙하고 안정적

Chaos는 UE와 분리가 어렵고 (UE 모듈 시스템에 깊게 통합), PBD가 반드시 유리한 것도 아님.

### 통합 순서 (단계별)

```
Step 1: Jolt 초기화 + 단순 강체 테스트
  PhysicsSystem 생성
  TempAllocator, JobSystemThreadPool 연결
  Box/Sphere Body 하나 추가 → Update() → 위치 읽기

Step 2: 씬 구조와 연결
  렌더 오브젝트 ↔ Jolt BodyID 매핑 테이블 구성
  오브젝트 생성/제거 시 Body 등록/해제

Step 3: 고정 타임스텝 축적기 구현
  accumulator += render_dt
  while(accumulator >= TIMESTEP) { Update(); }
  alpha로 보간

Step 4: 결과 반영
  Dynamic Body: GetWorldTransform() → 렌더 오브젝트 Transform 업데이트
  Kinematic Body: 렌더 Transform → MoveKinematic() (반대 방향)

Step 5: 스레딩 연결
  자체 스레드풀을 Jolt JobSystem 인터페이스로 래핑
```

### 핵심 통합 경계 (최소 구현)

```cpp
// 초기화
PhysicsSystem physics_system;
physics_system.Init(maxBodies, maxBodyPairs, maxContacts,
                    broadPhaseLayerInterface, objectVsBroadPhaseFilter, objectVsObjectFilter);

// 매 프레임
accumulator += dt;
while (accumulator >= FIXED_DT) {
    physics_system.Update(FIXED_DT, collisionSteps, &tempAllocator, &jobSystem);
    accumulator -= FIXED_DT;
}

// 결과 읽기
for (auto& [entityID, bodyID] : rigidBodyMap) {
    RVec3 pos = bodyInterface.GetPosition(bodyID);
    Quat  rot = bodyInterface.GetRotation(bodyID);
    scene.SetTransform(entityID, pos, rot);
}
```

Wicked Engine의 `wiPhysics_Jolt.cpp`가 이 글루 코드의 완성된 레퍼런스.
