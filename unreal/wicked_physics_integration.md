# Wicked Engine + Jolt 물리 통합 아키텍처

Wicked Engine이 Jolt Physics를 어떻게 붙였는지. 소스: `C:\graphics\wicked\origin92\WickedEngine\WickedEngine\`  
분석 관점: 통합 경계 / 데이터 흐름 / 스레딩 (UE5와 동일한 틀로 비교)

---

## 전체 흐름 요약

```
[① 물리 오브젝트 등록]
Scene에 RigidBodyPhysicsComponent 추가
  → RunPhysicsUpdateSystem() 첫 호출 시
  → AddRigidBody() → Jolt BodyInterface에 Body 생성 및 추가

[② 매 프레임: 물리 시작]
wiScene.cpp: Scene::Update()
  → wi::physics::RunPhysicsUpdateSystem(ctx, scene, dt)
      → Kinematic Body: Transform → Jolt 위치 동기화
      → physics_system.Update(TIMESTEP, ...) ← 고정 타임스텝 축적기

[③ 물리 실행 (Jolt 내부)]
Jolt PhysicsSystem::Update()
  → JobSystem으로 병렬 Job 실행
  → Gravity → Collision → SolveVelocity → Integrate → SolvePosition

[④ 결과 반영]
RunPhysicsUpdateSystem() 계속
  → body_interface.GetWorldTransform(bodyID)  ← Jolt에서 결과 읽기
  → 보간 (alpha = accumulator / TIMESTEP)
  → transform->translation_local, rotation_local 업데이트  ← 렌더링에 반영
```

---

## ① 물리 오브젝트 등록

### 컴포넌트 구조

Wicked Engine은 ECS(Entity Component System) 기반.  
물리가 필요한 Entity에 `RigidBodyPhysicsComponent`를 붙이면 됨.

`wiScene_Components.h` (line ~405):

```cpp
struct RigidBodyPhysicsComponent {
    // 충돌 형상 선택
    enum CollisionShape { BOX, SPHERE, CAPSULE, CONVEX_HULL,
                          TRIANGLE_MESH, CYLINDER, HEIGHTFIELD };
    
    // 물리 속성
    float mass;
    float friction;
    float restitution;
    float damping_linear;
    float damping_angular;
    
    // 특수 타입 (선택)
    struct Vehicle { enum Type { Car, Motorcycle }; ... };
    struct Character { float maxSlopeAngle; float gravityFactor; };
    
    // Jolt Body 핸들 (내부 관리)
    wi::allocator::shared_ptr<void> physicsobject;  // RigidBody* 캐스팅
};
```

UE5의 `FBodyInstance`에 해당하는 구조. `physicsobject`가 Jolt `BodyID`를 담은 내부 구조체 포인터.

---

### 내부 RigidBody 구조

`wiPhysics_Jolt.cpp` (line ~302):

```cpp
struct RigidBody {
    BodyID bodyID;                    // Jolt Body 핸들
    ShapeRefC shape;                  // 충돌 형상
    EMotionType motiontype;           // Static/Kinematic/Dynamic
    float friction, restitution;
    
    // 보간용 이전 상태
    Vec3 prev_position;
    Quat prev_rotation;
    
    // 계층 구조 지원
    XMFLOAT4X4 parentMatrix;          // 부모 엔티티의 월드 트랜스폼
    XMFLOAT4X4 parentMatrixInverse;
    Mat44 additionalTransform;        // 메시 오프셋 보정
    Mat44 additionalTransformInverse;
    
    // 특수 컴포넌트
    VehicleConstraint* vehicle_constraint;
    Ref<Character> character;
};
```

---

### Body 생성 및 등록 시점

`wiPhysics_Jolt.cpp` (line ~2146):

```cpp
// RunPhysicsUpdateSystem 초반부 — 병렬 처리
wi::jobsystem::Dispatch(ctx, scene.rigidbodies.GetCount(), dispatchGroupSize, [&](auto& args) {
    RigidBodyPhysicsComponent& physicscomponent = scene.rigidbodies[args.jobIndex];
    
    // physicsobject가 없거나 파라미터 변경됐으면 새로 생성
    if (physicscomponent.physicsobject == nullptr || physicscomponent.IsRefreshParametersNeeded()) {
        AddRigidBody(scene, entity, physicscomponent, *transform, mesh);
    }
});
```

`AddRigidBody()` 내부:
1. `CollisionShape` 타입에 따라 Jolt Shape 생성 (Box, Sphere, ConvexHull, 메시 등)
2. `BodyCreationSettings` 구성 (위치, 회전, Shape, MotionType)
3. `body_interface.CreateAndAddBody()` → Jolt 내부에 등록
4. `RigidBody` 구조체에 `bodyID` 보관

UE5와의 차이: UE5는 Actor 생성 시 자동 등록(`OnCreatePhysicsState`), Wicked는 **매 프레임 RunPhysicsUpdateSystem에서 lazy 등록**.

---

## ② 게임루프와의 연결 (통합 경계)

### 호출 위치

`wiScene.cpp` (line 258):

```cpp
void Scene::Update(float dt) {
    // ...
    wi::jobsystem::Wait(ctx);                      // 애니메이션 완료 대기
    wi::physics::RunPhysicsUpdateSystem(ctx, *this, dt);  // ← 물리 업데이트
    wi::jobsystem::Wait(ctx);
    UpdateTransforms(ctx);                          // 트랜스폼 계층 업데이트
    // ...
}
```

UE5의 `TG_StartPhysics` ~ `TG_EndPhysics` 구조와 달리, **단일 함수 호출로 시작~끝까지 처리**. 물리가 끝날 때까지 기다린 뒤 다음 단계로 진행.

---

## ③ 고정 타임스텝 축적기 (Fixed Timestep Accumulator)

`wiPhysics_Jolt.cpp` (line ~2833):

```cpp
static constexpr float TIMESTEP = 1.0f / 60.0f;  // 고정 물리 스텝 (line ~93)
static constexpr int ACCURACY   = 4;              // 최대 스텝 수

physics_scene.accumulator += dt;                  // 렌더 프레임 시간 축적

while (physics_scene.accumulator >= TIMESTEP) {
    // 보간 활성화 시: 이전 위치 저장
    if (IsInterpolationEnabled()) {
        SavePreviousState();  // prev_position, prev_rotation 저장
    }
    
    // Jolt 물리 스텝 실행
    physics_scene.physics_system.Update(TIMESTEP, COLLISION_STEPS,
                                        &temp_allocator, &job_system);
    
    physics_scene.accumulator -= TIMESTEP;
}

// 보간 가중치 계산 (0~1)
physics_scene.alpha = physics_scene.accumulator / TIMESTEP;
```

**왜 고정 타임스텝인가:**
- 물리 시뮬레이션은 `dt`에 민감 → 가변 렌더 `dt`를 그대로 쓰면 결과가 달라짐
- 60Hz로 고정해서 결정론적(deterministic)이고 안정적인 시뮬레이션 보장

**보간:**
- 렌더 120fps, 물리 60fps면 렌더 프레임 사이에 물리 결과가 없음
- `alpha`를 써서 이전 물리 결과와 현재 물리 결과 사이를 선형 보간
- 시각적으로 부드러운 움직임 제공

UE5에는 이 패턴이 없음 — UE5는 렌더 프레임과 물리 프레임을 맞춤 (`bTickPhysicsAsync=false` 기본).

---

## ④ 물리 결과 → 렌더링 반영

`wiPhysics_Jolt.cpp` (line ~2931):

```cpp
// Dynamic Body 결과 읽기 (모든 rigidbody를 병렬로 처리)
wi::jobsystem::Dispatch(ctx, scene.rigidbodies.GetCount(), ..., [&](auto& args) {
    // Jolt에서 월드 트랜스폼 읽기
    Mat44 mat = body_interface.GetWorldTransform(physicsobject.bodyID);
    
    // 메시 오프셋 역변환 (모델 원점 ≠ 질량 중심 보정)
    mat = mat * physicsobject.additionalTransformInverse;
    
    Vec3 position = mat.GetTranslation();
    Quat rotation = mat.GetQuaternion().Normalized();
    
    // 보간 적용
    if (IsInterpolationEnabled()) {
        position = position * alpha + physicsobject.prev_position * (1 - alpha);
        rotation = physicsobject.prev_rotation.SLERP(rotation, alpha);
    }
    
    // TransformComponent 업데이트 (렌더링이 이걸 읽음)
    transform->translation_local = cast(position);
    transform->rotation_local = cast(rotation);
    
    // 부모 공간으로 변환 (계층 구조 지원)
    transform->MatrixTransform(physicsobject.parentMatrixInverse);
});
```

**UE5 SyncBodies와의 차이:**
- UE5: `PullPhysicsStateForEachDirtyProxy` — Dirty한 Particle만 복사 (최적화)
- Wicked: **모든 rigidbody를 매 프레임 읽음** — 단순하지만 항상 최신 상태 보장

---

## 데이터 흐름 다이어그램

```
[씬 초기화 / 첫 프레임]
RigidBodyPhysicsComponent 추가
  → AddRigidBody()
  → Jolt BodyInterface.CreateAndAddBody()
  → physicsobject.bodyID 보관

[매 프레임]
                  Game Thread (wiScene::Update)
                           │
1. AnimationUpdate         │
   (wi::jobsystem)         │
                           │ Wait()
                           │
2. RunPhysicsUpdateSystem  │
   ├─ Kinematic 동기화     │  Transform → Jolt (코드 제어 물체)
   │  MoveKinematic()      │
   ├─ accumulator += dt    │
   │  while(acc >= 1/60)  │
   │    Jolt.Update()  ────┼──► Jolt Job System (멀티코어)
   │    acc -= 1/60        │         Gravity
   │                       │         FindCollisions (병렬)
   │                       │         SolveVelocity  (병렬)
   │                       │         IntegrateVelocity
   │                       │         SolvePosition  (병렬)
   │  ◄────────────────────┤    완료
   ├─ alpha 계산           │
   └─ GetWorldTransform()  │  Jolt → Transform (보간 포함)
      transform 업데이트   │
                           │ Wait()
                           │
3. UpdateTransforms        │  계층 구조 최종 업데이트
                           │
4. 렌더링                  │  TransformComponent 읽어서 그림
```

---

## UE5 vs Wicked Engine 통합 방식 비교

| 항목 | UE5 + Chaos | Wicked + Jolt |
|------|-------------|---------------|
| **등록 시점** | Actor 생성 시 즉시 (`OnCreatePhysicsState`) | 첫 Update 루프에서 lazy 등록 |
| **통합 경계** | TickGroup (`TG_StartPhysics` ~ `TG_EndPhysics`) | 단일 함수 `RunPhysicsUpdateSystem()` |
| **타임스텝** | 렌더 프레임과 동일 (기본) | **고정 1/60초 축적기** |
| **보간** | `bTickPhysicsAsync=true`로 선택 | 기본 제공, `IsInterpolationEnabled()` |
| **스레딩** | UE TaskGraph (Hi-Pri Worker) | Wicked JobSystem (Jolt 어댑터) |
| **결과 복사** | Dirty Particle만 (`PullPhysicsStateForEachDirtyProxy`) | 매 프레임 전체 읽기 |
| **계층 구조** | USceneComponent 계층 | `parentMatrix` 수동 보정 |
| **씬 구조** | Component 기반 (Actor→Component→BodyInstance) | **ECS** (Entity + ComponentManager) |
| **물리 알고리즘** | PBD | **임펄스 기반** |

---

## Soft Body / Ragdoll 처리

### Soft Body

`wiPhysics_Jolt.cpp` (line ~2971):

```cpp
// Jolt SoftBody 정점 → GPU 스키닝용 본 매트릭스 생성
SoftBodyMotionProperties* motion = body.GetMotionProperties();
const Array<SoftBodyVertex>& soft_vertices = motion->GetVertices();

for (size_t i = 0; i < soft_vertices.size(); ++i) {
    // 인접 3 정점으로 로컬 방향 행렬 계산
    XMMATRIX W = GetOrientation(P0, P1, P2);
    
    // GPU 스키닝용 본 데이터로 변환
    physicscomponent.boneData[i].Create(W * inverseBindMatrix);
}
```

Soft Body 메시의 각 정점이 물리 시뮬레이션 → 그 위치/방향으로 스키닝 본 생성 → GPU가 렌더링 메시에 스키닝 적용.

### Ragdoll (Humanoid)

`wiPhysics_Jolt.cpp` (line ~3027):

```cpp
// 각 본 = Jolt 캡슐 Body
for (auto& rb : ragdoll.rigidbodies) {
    Mat44 mat = body_interface.GetWorldTransform(rb.bodyID);
    transform->translation_local = cast(position);
    transform->rotation_local = cast(rotation);
}
```

각 관절 본이 별개의 Jolt Body → `SwingTwistConstraint`로 연결 → 시뮬레이션 결과가 직접 스켈레탈 메시 본 트랜스폼을 구동.
