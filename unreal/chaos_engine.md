# Chaos 물리 엔진

UE5의 내장 물리 엔진. 소스: `D:\Epic games\UE_5.7\Engine\Source\Runtime\Experimental\Chaos\`

---

## Chaos가 뭔가

Epic이 UE5부터 직접 만든 물리 엔진. UE4까지는 NVIDIA의 PhysX를 가져다 썼는데, UE5에서 자체 개발로 교체했다.

**왜 새로 만들었나:**
- PhysX는 NVIDIA 소유 → 소스 수정 불가, 커스터마이징 한계
- 대규모 파괴(Destruction), 천(Cloth), 군중 시뮬레이션 등 새로운 요구사항
- 멀티코어 CPU 최대 활용을 위한 설계 필요
- Fortnite 같은 대형 게임에서 수천 개 물체를 동시에 다루려면 새 아키텍처가 필요했음

**한 줄 정의:** Epic이 게임 엔진에 최적화하여 처음부터 설계한 **위치 기반(PBD)** 물리 엔진.

---

## PhysX와 핵심 차이: 임펄스 방식 vs PBD 방식

물리 시뮬레이션의 핵심은 "두 물체가 충돌했을 때 어떻게 움직임을 바꿔주나"인데, 이 방식이 근본적으로 다름.

### PhysX 방식 — 임펄스(Impulse) 기반

```
충돌 감지 → 임펄스(순간적인 힘) 계산 → 속도(Velocity)를 바꿈 → 위치가 간접적으로 바뀜

[공이 바닥에 충돌]
  충돌 법선 계산 → 임펄스 = f(질량, 속도, 복원계수)
  속도 V += 임펄스 / 질량
  다음 프레임에 위치 X += V * dt
```

속도를 먼저 바꾸고, 그 속도로 위치가 바뀌는 **간접적** 방식.  
안정성이 dt(프레임 시간)에 민감 → 저FPS에서 물리가 불안정해질 수 있음.

### Chaos 방식 — PBD(Position-Based Dynamics)

```
충돌 감지 → 위치(Position)를 직접 조정 → 속도는 위치 변화로 역산

[공이 바닥에 충돌]
  예측 위치 P 계산 (현재 V로 이동)
  P가 바닥 아래면 → P를 바닥 위로 직접 끌어올림 (위치 보정)
  V_new = (P_new - P_old) / dt  ← 속도는 나중에 역산
```

**비교:**

| 항목 | PhysX (Impulse) | Chaos (PBD) |
|------|-----------------|-------------|
| 충돌 반응 | 속도 변경 | **위치 직접 조정** |
| 안정성 | dt에 민감 | dt에 덜 민감 |
| 제약 수렴 | 속도 기반이라 오버슈트 가능 | 위치 직접 조정이라 수렴 보장 용이 |
| 에너지 보존 | 정확함 | 근사치 (수렴 반복 수에 의존) |
| 직관성 | 물리 법칙에 충실 | 비물리적이지만 안정적 |

PBD는 "물리적으로 완벽한 정확도"보다 "게임에서 안정적이고 빠른 시뮬레이션"에 최적화된 방식. 옷감, 머리카락, 로프 같은 것도 같은 PBD 프레임워크로 처리할 수 있어서 통합이 깔끔함.

---

## 핵심 개념: Particle

Chaos에서 물리 세계의 모든 물체는 **Particle**이라는 단위로 표현됨.

```
Particle = 물리 시뮬레이션의 최소 단위 (UE Actor가 아님!)

UE Actor (BP_M1A2) 
  → UPrimitiveComponent (SkeletalMeshComponent)
      → FBodyInstance
          → Chaos Particle   ← 이게 Chaos가 직접 다루는 단위
```

### Particle 3가지 타입

```
FGeometryParticle (Static)
  - 움직이지 않는 물체 (지형, 건물, 고정 장애물)
  - 위치/회전만 있음. 속도 없음
  - 솔버가 이 위치는 바꾸지 않음

FKinematicGeometryParticle (Kinematic)  
  - 물리 시뮬레이션의 영향을 받지 않고, 코드로 직접 움직임
  - 속도는 있지만 힘/충돌에 반응 안 함
  - 예: 이동하는 플랫폼, 애니메이션으로 움직이는 캐릭터

FPBDRigidParticle (Dynamic)
  - 완전한 물리 시뮬레이션 (중력, 충돌 반응, 힘 등 모두 작용)
  - 탱크, 공, 파괴 파편 등
  - P(예측위치), V(속도), M(질량), I(관성) 등 풀 상태 보유
```

### Particle의 핵심 데이터 (FPBDRigidParticle)

`Public\Chaos\PBDRigidParticles.h`

```cpp
// 현재 위치/회전 (게임 스레드에서 읽히는 값)
TVector<T, 3>    X;   // Position
TRotation<T, 3>  R;   // Rotation

// PBD 예측 위치/회전 (솔버 내부에서 쓰는 임시 값)
TVector<T, 3>    P;   // Predicted Position  ← PBD의 핵심
TRotation<T, 3>  Q;   // Predicted Rotation

// 속도
TVector<T, 3>    V;   // Linear Velocity
TVector<T, 3>    W;   // Angular Velocity (Omega)

// 물리 속성
T                M;   // Mass (질량)
TMatrix<T, 3, 3> I;   // Inertia Tensor (관성 텐서)
```

X/R은 확정된 위치, P/Q는 솔버가 계산 중인 예측 위치. 솔버가 끝나면 P/Q → X/R로 복사됨.

---

## 메모리 구조: SOA (Structure of Arrays)

성능을 위해 Chaos는 **SOA** 방식으로 데이터를 저장.

```
일반적인 방식 (AOS, Array of Structures):
particle[0] = { x, y, z, rx, ry, rz, vx, vy, vz, mass, ... }
particle[1] = { x, y, z, rx, ry, rz, vx, vy, vz, mass, ... }
particle[2] = { x, y, z, rx, ry, rz, vx, vy, vz, mass, ... }

Chaos 방식 (SOA, Structure of Arrays):
positions[]  = { p0.x,p0.y,p0.z, p1.x,p1.y,p1.z, p2.x,p2.y,p2.z }
velocities[] = { v0.x,v0.y,v0.z, v1.x,v1.y,v1.z, v2.x,v2.y,v2.z }
masses[]     = { m0, m1, m2 }
```

**왜 SOA가 빠른가:**

모든 파티클의 위치를 업데이트할 때:
- AOS: 파티클마다 메모리 위치가 멀리 흩어져 있음 → 캐시 미스 빈번
- SOA: 위치 데이터가 메모리에 연속으로 나열 → 한 번에 캐시에 올라옴 + SIMD로 4~8개 동시 처리 가능

`Public\Chaos\PBDRigidsSOAs.h` — Static/Kinematic/Dynamic이 별도 배열로 분리:

```cpp
class FPBDRigidsSOAs
{
    TUniquePtr<FGeometryParticles>       StaticParticles;       // 정적 물체들
    TUniquePtr<FKinematicGeometryParticles> KinematicParticles; // 키네마틱 물체들
    TUniquePtr<FPBDRigidParticles>       DynamicParticles;      // 동적 물체들
    TUniquePtr<FPBDRigidParticles>       DynamicDisabledParticles; // 슬립 상태
    TUniquePtr<FPBDRigidClusteredParticles> ClusteredParticles; // 파괴 파편 클러스터
};
```

타입이 분리된 이유: Dynamic만 솔버에서 매 프레임 업데이트하고, Static은 건드리지 않아도 됨. 불필요한 처리를 타입 구분으로 원천 차단.

---

## Island: 병렬 처리의 핵심

솔버는 모든 물체를 한꺼번에 처리하지 않고, **연결된 물체 그룹(Island)** 단위로 나눔.

```
물체들:
  A -- B -- C    (A가 B를 건드리고, B가 C를 건드리는 경우)
  D -- E         (D,E는 위 그룹과 무관)
  F              (혼자 있는 물체, 슬립 가능)

→ Island 1: {A, B, C}  — 독립적으로 병렬 처리 가능
→ Island 2: {D, E}     — 독립적으로 병렬 처리 가능
→ Island 3: {F}        — 이동 없으면 Sleep 처리
```

**Island의 이점:**
- Island 사이에는 의존성 없음 → TaskGraph에서 병렬 실행
- 정지한 Island는 Sleep → 솔버에서 완전 제외 (비용 0)
- 새 충돌 발생 → 해당 Island만 Wake-up

탱크 30대 Chaos가 2.63ms밖에 안 걸린 이유: 탱크들이 서로 닿지 않으면 각자 Island → 병렬 처리.

---

## 매 프레임 시뮬레이션 루프

`Public\Chaos\PBDRigidsEvolution.h`

```
매 프레임 (AdvanceSolverBy 호출):

① Force 적용
   중력, 외부 힘(바람, 폭발) → 각 Dynamic Particle의 V(속도) 업데이트

② 예측 위치 계산 (Predict)
   P = X + V * dt  (이 위치로 이동한다고 가정)
   Q = R + W * dt

③ 충돌 감지
   Broad Phase: 빠른 AABB로 충돌 후보 추려냄
   Narrow Phase: 정밀 형상으로 접촉점(Contact Manifold) 계산

④ 제약 해결 (Constraint Solving) — 반복 N회
   for (iter = 0; iter < NumIterations; iter++):
     충돌 제약: P를 겹치지 않도록 직접 조정 (PBD 핵심)
     관절 제약: 관절 한계 내로 P, Q 조정
     커스텀 제약: 차량 서스펜션 등

⑤ 속도 역산 (Velocity Update)
   V = (P - X) / dt   ← 속도는 위치 변화에서 역산
   W = (Q - R) / dt

⑥ 위치 확정
   X = P
   R = Q

⑦ Dirty Particle 마킹
   변경된 Particle을 DirtyList에 추가 → SyncBodies에서 게임 스레드로 복사
```

---

## 모듈 구성

Chaos는 하나의 모듈이 아니라 여러 모듈로 나뉨:

```
ChaosCore
  ─ 수학 라이브러리 (벡터, 행렬, 쿼터니언)
  ─ 기본 타입 정의
  └─ 의존성 없음 (엔진 없이도 사용 가능)

Chaos (물리 엔진 본체)
  ─ Particle, SOA, Evolution, Solver
  ─ Collision (Broad/Narrow Phase)
  ─ Constraints (PBD 제약들)
  ─ Island Manager
  ─ CCD (Continuous Collision Detection)
  └─ 의존: ChaosCore, IntelISPC(SIMD), Eigen(선형대수)

ChaosSolverEngine (UE 통합 레이어)
  ─ FSolverProxy: Chaos Particle ↔ UE Actor 연결
  ─ FPhysicsInterface: UE 코드가 Chaos 부르는 창구
  └─ 의존: Chaos, Engine, RHI

ChaosVehicles (차량 특화)
  ─ FChaosVehicleManagerAsyncCallback
  ─ 타이어, 서스펜션, 엔진 시뮬레이션
  └─ 의존: Chaos, ChaosSolverEngine

GeometryCollectionEngine (파괴 시스템)
  ─ TPBDRigidClusteredParticles
  ─ Voronoi 기반 파괴 셀 생성
  └─ 의존: Chaos, Voronoi

FieldSystem (영역 기반 힘)
  ─ 폭발 범위, 바람 영역 등
  └─ 의존: Chaos
```

---

## 이전 분석과 연결

이전 디버깅/분석에서 나온 용어들이 어디에 해당하는지:

| 이전 분석 결과 | Chaos 구조 |
|---------------|-----------|
| `STAT_ChaosTick 2.63ms` | `FPhysicsSolverAdvanceTask` → `AdvanceSolverBy()` → 전체 시뮬레이션 루프 1회 |
| `FChaosVehicleManagerAsyncCallback::OnPreSimulate_Internal` | 위 루프의 ① Force 적용 단계에서 호출되는 차량 콜백 |
| `SyncBodies()` → `PullPhysicsStateForEachDirtyProxy` | ⑦ Dirty 마킹된 Particle을 게임 스레드로 복사 |
| `FPBDRigidParticles` | 탱크 30대가 각각 이 타입의 Particle로 등록됨 |
| `Island` | 탱크들이 서로 분리된 Island → 2.63ms에 병렬 처리 가능 |
| `ProcessUntilTasksComplete 78ms` | Chaos 솔버 2.63ms가 끝난 후 게임 스레드가 BP 틱 처리하는 구간 (물리랑 무관) |

---

## 요약

```
Chaos = Epic이 UE5를 위해 처음부터 만든 PBD 기반 물리 엔진

핵심 설계 원칙:
  PBD   — 위치를 직접 조정, 안정적이고 복잡한 제약 처리에 유리
  SOA   — 데이터를 타입별로 분리, SIMD 최적화
  Island — 연결된 물체 그룹 단위 병렬 처리
  비동기 — 물리 스레드와 게임 스레드 분리, TaskGraph 사용

PhysX 대비:
  유리: 커스터마이징 가능, 파괴/천/유체 통합, 멀티코어 확장성
  불리: 에너지 보존 정확도 낮음, 아직 성숙도 낮음 (UE4 PhysX 대비 버그 많음)
```
