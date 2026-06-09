# Jolt Physics 엔진

Wicked Engine이 사용하는 물리 엔진.

---

## Jolt가 뭔가

Guerrilla Games(Horizon Zero Dawn 개발사)의 Jorrit Rouwe가 제작하고, 2021년 오픈소스로 공개한 물리 엔진. 현대 멀티코어 CPU에 맞게 처음부터 설계됨.

**채택 현황:**
- Wicked Engine (자체 통합)
- Godot 4.x (공식 채택, 2023~)
- 다수의 인디/미들웨어 엔진들

**Chaos와 비교했을 때 포지션:**

| | Chaos (UE5) | Jolt |
|--|-------------|------|
| 제작 주체 | Epic (대형 스튜디오) | Guerrilla 개인 프로젝트 → 오픈소스 |
| 라이선스 | UE 라이선스 종속 | MIT (완전 자유) |
| 알고리즘 | PBD (위치 기반) | **임펄스 기반** (속도 기반) |
| 목표 | UE5 전체 통합 (파괴, 천, 유체까지) | 빠르고 정확한 강체/차량/캐릭터 |
| 성숙도 | 비교적 신생, 버그 많음 | 안정적, 검증된 AAA 게임 실전 사용 |

---

## 핵심 알고리즘: 임펄스 기반

Jolt는 Chaos(PBD)와 달리 **임펄스 기반(Impulse-Based)** 방식. PhysX와 같은 계열이지만 현대적으로 재구현.

```
매 프레임 시뮬레이션 루프:

① 중력/외부 힘 → 속도(V) 업데이트
② 충돌 감지 (Broad + Narrow Phase)
③ 속도 제약 해결 (SolveVelocityConstraints)
     - 충돌 임펄스 계산 → V에 적용
     - 관절 제약 임펄스 계산 → V에 적용
     - 반복(iteration)하여 수렴
④ 위치 적분 (IntegrateVelocity)
     X += V * dt
     R += W * dt
⑤ 위치 제약 해결 (SolvePositionConstraints)
     - Baumgarte 안정화로 관통 보정
⑥ CCD (선택): 고속 물체 터널링 감지
```

Chaos와 달리 "예측 위치 P"를 쓰지 않음. V를 먼저 정확히 계산하고 X에 적분하는 전통 방식.

---

## Body — 물리 세계의 기본 단위

`Jolt/Physics/Body/Body.h`

```cpp
class Body {
    RVec3                mPosition;          // 월드 위치 (질량 중심)
    Quat                 mRotation;          // 월드 회전
    AABox                mBounds;            // 월드 공간 AABB
    RefConst<Shape>      mShape;             // 충돌 형상
    MotionProperties*    mMotionProperties;  // 속도/힘 데이터 (Static이면 nullptr)
    uint64               mUserData;          // 사용자 정의 데이터
    CollisionGroup       mCollisionGroup;    // 충돌 레이어
};
```

**Motion Type** (`Jolt/Physics/Body/MotionType.h`):

```cpp
enum class EMotionType : uint8 {
    Static,     // 움직이지 않음. mMotionProperties = nullptr
    Kinematic,  // 힘에 반응 안 함. 속도로만 이동 (코드로 직접 제어)
    Dynamic,    // 완전한 물리 시뮬레이션
};
```

Static은 `mMotionProperties`가 `nullptr`라는 게 중요. 메모리 낭비 없고 솔버가 건드리지 않음.

**주요 인터페이스:**
```cpp
// 속도
body.GetLinearVelocity()    / SetLinearVelocity()
body.GetAngularVelocity()   / SetAngularVelocity()

// 힘 (Dynamic만)
body.AddForce(Vec3)
body.AddTorque(Vec3)

// 위치 (읽기는 모든 타입, 쓰기는 BodyInterface 통해서)
body.GetPosition()
body.GetRotation()

// 상태
body.IsActive()
body.IsStatic() / IsKinematic() / IsDynamic()
```

---

## Shape — 충돌 형상

`Jolt/Physics/Collision/Shape/Shape.h`

```
EShapeType (대분류):
  Convex      → Box, Sphere, Capsule, Cylinder, ConvexHull
  Compound    → 여러 Shape 조합 (StaticCompound, MutableCompound)
  Decorated   → 기존 Shape에 변환 추가 (회전, 스케일, 오프셋)
  Mesh        → 임의 삼각형 메시 (Static 전용)
  HeightField → 지형
  SoftBody    → 소프트바디 전용
```

**핵심 특징:**
- Shape는 Body와 분리 — 여러 Body가 같은 Shape를 공유 가능 (Ref 카운팅)
- `CompoundShape`으로 복잡한 물체 표현 (탱크 = 차체 Box + 포탑 Box 조합)
- `DecoratedShape`으로 기존 Shape에 오프셋/스케일 적용 (복사 없이)

---

## 충돌 감지 파이프라인

### Broad Phase

`Jolt/Physics/Collision/BroadPhase/`

```
BroadPhaseQuadTree (기본):
  - 모든 Body를 AABB로 감싼 쿼드트리
  - UpdatePrepare() / UpdateFinalize() 로 매 프레임 갱신
  - FindCollidingPairs() → 겹치는 AABB 쌍 목록 반환

BroadPhaseBruteForce (개발/디버그):
  - 모든 쌍 전수 검사 (O(n²))
```

**레이어 시스템:**
- `BroadPhaseLayer`: 물체 카테고리 (Static, Movable, Sensor 등)
- `ObjectVsBroadPhaseLayerFilter`: 레이어 간 충돌 여부 결정
- Static끼리는 충돌 검사 자체를 건너뜀 → 성능 절감

### Narrow Phase

Broad Phase가 후보를 추리면 정밀 형상 충돌 검사.

```
Shape 조합마다 전용 알고리즘:
  Sphere vs Sphere   → 두 중심 거리 비교 (가장 빠름)
  Box vs Box         → SAT (Separating Axis Theorem)
  ConvexHull vs Any  → GJK / EPA 알고리즘
  Mesh vs Convex     → BVH 순회 후 삼각형별 GJK
```

출력: **Contact Manifold** (접촉점, 법선, 침투 깊이)

---

## Constraint(제약) 시스템

`Jolt/Physics/Constraints/`

두 Body 사이의 상대 운동을 제한하는 시스템.

**제공되는 Constraint 종류:**

```
FixedConstraint      — 두 Body를 완전히 고정 (용접)
PointConstraint      — 한 점에서 연결 (볼 소켓)
HingeConstraint      — 1축 회전만 허용 (문 경첩)
SliderConstraint     — 1축 이동만 허용 (서랍)
DistanceConstraint   — 두 점 사이 거리 유지 (로프)
ConeConstraint       — 원뿔 범위 내 회전 허용
SwingTwistConstraint — 스윙+트위스트 제한 (어깨 관절)
SixDOFConstraint     — 6자유도 완전 제어
PathConstraint       — 경로를 따라 이동
GearConstraint       — 기어 비율로 회전 연동
PulleyConstraint     — 도르래 (한쪽이 내려가면 한쪽 올라감)
VehicleConstraint    — 차량 전용 (서스펜션, 타이어 물리)
```

**Constraint 해결 방식:**

```cpp
// 속도 레벨 제약 (주)
virtual bool SolveVelocityConstraint(float dt) = 0;
// → 접촉/관절 조건을 만족하는 임펄스 계산 → V 수정

// 위치 레벨 제약 (보조 — 드리프트 보정)
virtual bool SolvePositionConstraint(float dt, float inBaumgarte) = 0;
// → Baumgarte 안정화: 약간의 위치 보정으로 관통/이탈 방지
```

`inBaumgarte` 계수: 0이면 위치 오류 무시, 1이면 한 프레임에 완전 보정 (진동 유발). 보통 0.2~0.4.

---

## Island (아일랜드)

Jolt도 Chaos처럼 Island 기반 병렬화 사용.

```cpp
// PhysicsSystem.h 내부
void JobBuildIslandsFromConstraints(...);

// Island = 제약/충돌로 연결된 Body 그룹
// Island 사이에는 의존성 없음 → 병렬 처리 가능
// Sleep: 충분히 오래 정지한 Island → 비활성화 (솔버 제외)
```

---

## Job System — 멀티스레딩

`Jolt/Core/JobSystem.h`

Jolt 자체 Job 인터페이스를 정의하고, 엔진이 구현체를 주입.

```cpp
class JobSystem {
    // 작업 생성
    JobHandle CreateJob(const char* name, ColorArg color,
                        const JobFunction& func, uint32 numDependencies = 0);
    
    // 의존성 그래프로 실행 순서 제어
    // A → B → C 순서로 실행하고 싶으면:
    //   B.AddDependency(A)
    //   C.AddDependency(B)
    
    // 완료 대기
    void WaitForJobs(Barrier* barrier);
};
```

Wicked Engine은 자신의 Job System(`wi::jobsystem`)을 Jolt에 어댑터로 연결해서 사용.

**Update() 한 번에 생성되는 Job들:**

```
JobApplyGravity
JobSetupVelocityConstraints
JobBuildIslandsFromConstraints
JobFindCollisions          ← 병렬 (Island 단위)
JobSolveVelocityConstraints ← 병렬 (Island 단위)
JobIntegrateVelocity       ← 병렬
JobSolvePositionConstraints ← 병렬 (Island 단위)
JobFindCCDContacts         ← 선택
JobResolveCCDContacts      ← 선택
```

의존성 그래프로 연결돼 있어서, 가능한 Job은 CPU 코어들이 병렬로 처리.

---

## Chaos vs Jolt 비교

| 항목 | Chaos (UE5) | Jolt (Wicked) |
|------|-------------|---------------|
| **알고리즘** | PBD (위치 기반) | 임펄스 기반 |
| **제약 해결** | 위치 직접 수정 | 임펄스로 속도 수정 |
| **에너지 보존** | 근사 (반복 수에 의존) | 더 정확 |
| **안정성** | 복잡한 제약에 유리 | 전통 방식, 검증됨 |
| **메모리** | SOA (타입별 분리) | 일반 구조체 |
| **Island** | 있음 | 있음 |
| **스레딩** | UE TaskGraph | 자체 JobSystem (어댑터 교체 가능) |
| **CCD** | 있음 | 있음 |
| **소프트바디** | 있음 | 있음 |
| **차량** | ChaosVehicles | VehicleConstraint |
| **파괴(Destruction)** | 내장 (주력 기능) | 없음 |
| **라이선스** | UE 종속 | MIT |
