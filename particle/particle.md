언리얼 엔진 나이아가라


ECS - Entity Component System

# wicked particle components 찾아보기

![alt text](image.png)

1. EmittedParticleSystem (wiEmittedParticle.h:19)

일반적인 파티클 이미터 시스템으로, 다양한 파티클 효과를 만들 수 있습니다:

주요 기능:
- 셰이더 타입: SOFT, SOFT_DISTORTION, SIMPLE, SOFT_LIGHTING
- 물리 시뮬레이션: SPH 유체 시뮬레이션 지원
- 파티클 속성: 크기, 수명, 회전, 모션블러, 질량, 중력, 속도, 드래그
- 스프라이트 시트: 프레임 애니메이션 지원 (framesX, framesY, frameRate)
- 충돌: 깊이 충돌 감지, 반발력(restitution) 설정
- 렌더링 옵션: 정렬, 프레임 블렌딩, 레이트레이싱 지원
- 불투명도 커브: 파티클 생명주기에 따른 불투명도 제어

플래그 옵션:
- DEBUG, PAUSED, SORTING, DEPTHCOLLISION, SPH_FLUIDSIMULATION
- HAS_VOLUME, FRAME_BLENDING, COLLIDERS_DISABLED, USE_RAIN_BLOCKER
- TAKE_COLOR_FROM_MESH

2. HairParticleSystem (wiHairParticle.h:19)

머리카락/털 같은 스트랜드 기반 파티클 시스템:

주요 기능:
- 스트랜드 시뮬레이션: strandCount, segmentCount, billboardCount로 구성
- 물리 속성: 길이, 강성(stiffness), 드래그, 중력, 랜덤성
- 렌더링: 빌보드 기반, 레이트레이싱 지원
- 최적화: 뷰 거리(viewDistance), 컬링 지원
- 텍스처 아틀라스: 여러 텍스처 영역 지원
- 카메라 벤딩: 카메라 방향에 따라 휘어지는 효과

플래그 옵션:
- REBUILD_BUFFERS, DIRTY, CAMERA_BEND_ENABLED

# VizMotive vs Wicked Engine: ECS 구조 및 파티클 시스템 비교

1. ECS (Entity-Component-System) 구조 비교

Wicked Engine (wiScene.h)

ComponentLibrary 기반 구조:
- wi::ecs::ComponentLibrary 사용
- ComponentManager<T> 템플릿으로 각 컴포넌트 타입 관리
- Scene 클래스가 모든 ComponentManager를 멤버로 보유

주요 컴포넌트들 (48개 line에 등록됨):
wi::ecs::ComponentManager<EmittedParticleSystem>& emitters;  // ← 파티클!
wi::ecs::ComponentManager<HairParticleSystem>& hairs;         // ← 헤어 파티클!
wi::ecs::ComponentManager<NameComponent>& names;
wi::ecs::ComponentManager<TransformComponent>& transforms;
wi::ecs::ComponentManager<MaterialComponent>& materials;
wi::ecs::ComponentManager<MeshComponent>& meshes;
wi::ecs::ComponentManager<ObjectComponent>& objects;
wi::ecs::ComponentManager<LightComponent>& lights;
wi::ecs::ComponentManager<CameraComponent>& cameras;
// ... 총 30개 이상의 컴포넌트

VizMotive Engine (ECS.h, Components.h)

유사한 ComponentManager 구조:
- vz::ecs::ComponentManager<T> 템플릿 사용
- ComponentManager_Interface 인터페이스 기반

주요 컴포넌트들 (현재 발견된 것):
GGeometryComponent
GMaterialComponent
GLightComponent
GProbeComponent
GRenderableComponent
GSpriteComponent
GSpriteFontComponent
// 파티클 컴포넌트는 발견되지 않음!

핵심 차이점:
- VizMotive에는 파티클 시스템이 아직 구현되지 않음
- Wicked는 Scene 레벨에서 모든 컴포넌트를 중앙 집중 관리
- VizMotive도 유사한 구조이지만 컴포넌트 종류가 적음

---
2. Wicked Engine 파티클 관련 컴포넌트

A. EmittedParticleSystem (wiEmittedParticle.h)

용도: GPU 기반 파티클 시스템 (일반적인 파티클 이펙트)

주요 특징:
- GPU 버퍼 기반 시뮬레이션
- particleBuffer: 파티클 데이터
- aliveList[2], deadList: Alive/Dead 관리
- distanceBuffer: 정렬용
- counterBuffer: Indirect draw용
- 물리 시뮬레이션
- SPH (Smoothed Particle Hydrodynamics) 유체 시뮬레이션 지원
- sphGridCells, sphParticleCells, densityBuffer
- 중력, 드래그, 반발력, 충돌 지원
- 렌더링 옵션
enum PARTICLESHADERTYPE {
    SOFT,              // 소프트 파티클
    SOFT_DISTORTION,   // 왜곡 효과
    SIMPLE,            // 심플
    SOFT_LIGHTING,     // 라이팅 적용
};
- 파티클 속성
- Size, Life, Velocity, Gravity
- Sprite sheet 애니메이션 (framesX, framesY, frameRate)
- Opacity curve 제어
- Motion blur
- 랜덤 팩터, 노말 팩터
- Raytracing 지원
- BLAS (Bottom-Level Acceleration Structure)
- primitiveBuffer

B. HairParticleSystem (wiHairParticle.h)

용도: 헤어/털/실 시뮬레이션

특징:
- 물리 기반 헤어 시뮬레이션
- 별도 컴포넌트로 분리됨

---
3. 주요 구조적 차이점

| 특징            | Wicked Engine                               | VizMotive Engine      |
|---------------|---------------------------------------------|-----------------------|
| ECS 구조        | ComponentLibrary + ComponentManager         | ComponentManager (유사) |
| 파티클 시스템       | ✅ EmittedParticleSystem✅ HairParticleSystem | ❌ 없음                  |
| 컴포넌트 수        | 30개 이상                                      | 약 7개 확인됨              |
| Scene 통합      | Scene 클래스에 모든 ComponentManager              | 분산된 구조                |
| Serialization | Archive 기반                                  | Archive 기반 (유사)       |
| EntityMapper  | 있음                                          | 있음                    |

---
4. 파티클 시스템 포팅 시 필요한 것들

필수 파일들:

Wicked Engine에서 가져와야 할 파일:
├── wiEmittedParticle.h/cpp          # 메인 파티클 시스템
├── ShaderInterop_EmittedParticle.h  # GPU-CPU 인터페이스
├── Shaders/
│   ├── emittedparticle_*.hlsl       # 파티클 시뮬레이션 셰이더
│   └── ...

VizMotive에 추가해야 할 것:

1. Component 추가:
- GEmittedParticleComponent 생성
- Scene에 ComponentManager<GEmittedParticleComponent> 등록
2. GPU 리소스 관리:
- Particle buffers (alive/dead lists)
- Indirect dispatch buffers
- SPH 관련 버퍼 (옵션)
3. 렌더링 파이프라인:
- Particle 전용 렌더 패스
- Sorting (distance-based)
- Soft particle depth collision
4. 시뮬레이션 업데이트:
- CPU: UpdateCPU() - Emission, burst
- GPU: Compute shader 기반 시뮬레이션
5. Raytracing 통합:
- Particle BLAS 생성
- TLAS에 파티클 인스턴스 추가

---
5. 권장 포팅 전략

Phase 1: 기본 구조

1. GEmittedParticleComponent 스텁 생성
2. Scene에 ComponentManager 등록
3. 기본 버퍼 생성 로직 포팅

Phase 2: GPU 시뮬레이션

1. Emit shader 포팅
2. Simulation shader 포팅
3. Indirect draw setup

Phase 3: 렌더링

1. 파티클 렌더링 셰이더
2. Soft particle 구현
3. Sorting 추가

Phase 4: 고급 기능

1. SPH 유체 시뮬레이션
2. Sprite sheet 애니메이션
3. Raytracing 통합

---

# WickedEngine의 System 개념

WickedEngine에서 "System"은 두 가지 의미가 있습니다:

1. ParticleSystem 클래스 (데이터 컴포넌트)

EmittedParticleSystem과 HairParticleSystem은 실제로는 Component입니다. 클래스 이름에 "System"이 붙어있지만, Scene에서
ComponentManager로 관리됩니다 (wiScene.h:48-49):

wi::ecs::ComponentManager<EmittedParticleSystem>& emitters = ...
wi::ecs::ComponentManager<HairParticleSystem>& hairs = ...

2. Scene에 등록된 모든 Component 종류 (wiScene.h:30-66)

WickedEngine은 ComponentManager를 통해 다양한 컴포넌트를 관리합니다:

기본 컴포넌트:
- NameComponent - 이름
- LayerComponent - 레이어 마스크
- TransformComponent - 위치/회전/스케일
- HierarchyComponent - 부모-자식 관계

렌더링 컴포넌트:
- MaterialComponent - 머티리얼 (PBR, Water, Cartoon 등)
- MeshComponent - 메시 데이터
- ObjectComponent - 렌더링 오브젝트
- ImpostorComponent - 임포스터

물리 컴포넌트:
- RigidBodyPhysicsComponent - 강체 물리
- SoftBodyPhysicsComponent - 소프트바디 물리
- ColliderComponent - 충돌체
- PhysicsConstraintComponent - 물리 제약

애니메이션/스켈레톤:
- ArmatureComponent - 스켈레톤
- AnimationComponent - 애니메이션
- AnimationDataComponent - 애니메이션 데이터
- InverseKinematicsComponent - IK
- HumanoidComponent - 휴머노이드

라이팅/환경:
- LightComponent - 라이트
- CameraComponent - 카메라
- EnvironmentProbeComponent - 환경 프로브
- WeatherComponent - 날씨

파티클:
- EmittedParticleSystem - 일반 파티클 이미터
- HairParticleSystem - 헤어/스트랜드 파티클

기타:
- ForceFieldComponent - 포스필드
- DecalComponent - 데칼
- SoundComponent - 사운드
- VideoComponent - 비디오
- SpringComponent - 스프링
- ScriptComponent - 스크립트
- ExpressionComponent - 표정
- CharacterComponent - 캐릭터
- SplineComponent - 스플라인
- MetadataComponent - 메타데이터
- wi::terrain::Terrain - 지형
- wi::Sprite - 스프라이트
- wi::SpriteFont - 폰트
- wi::VoxelGrid - 복셀 그리드

총 34개의 컴포넌트 타입이 있습니다.

VizMotive와 마찬가지로 ECS 패턴을 사용하지만, WickedEngine은 파티클 컴포넌트의 클래스 이름에 "System"을 붙였을 뿐, 실제로는 모두
Component입니다.

파티클의 멤버 파라미터

---

# ECS의 Component와 System 개념

Component (컴포넌트): 순수한 데이터
```c
// Component = 데이터만 담는 구조체
struct TransformComponent {
    XMFLOAT3 position;
    XMFLOAT4 rotation;
    XMFLOAT3 scale;
    XMFLOAT4X4 worldMatrix;
    // 메서드가 있을 수도 있지만, 주로 데이터 중심
};

struct VelocityComponent {
    XMFLOAT3 velocity;
    XMFLOAT3 acceleration;
};
```

System (시스템): 컴포넌트를 처리하는 로직
```c
// System = 컴포넌트들을 순회하며 업데이트하는 함수
void PhysicsSystem(ComponentManager<TransformComponent>& transforms,
                    ComponentManager<VelocityComponent>& velocities,
                    float dt) {
    // 모든 엔티티를 순회하며 물리 시뮬레이션
    for (int i = 0; i < entities.size(); i++) {
        transforms[i].position += velocities[i].velocity * dt;
        velocities[i].velocity += velocities[i].acceleration * dt;
    }
}
```
ECS의 핵심 철학:

- Component: "무엇을 가지고 있는가?" (데이터)
- System: "무엇을 하는가?" (로직/행동)
- Entity: 컴포넌트들의 컨테이너 (ID만 가진 빈 껍데기)

# 왜 system?

1. 역사적 관습

보통 particle system 이라는 용어 및 이름을 자주 사용함

2. 의미론적 차이

"Particle System"이라는 이름이 의미하는 것:
- 단순한 "위치" 컴포넌트가 아니라
- 수천 개의 파티클을 관리하는 복잡한 하위 시스템
- Emitter, Simulation, Collider, Renderer 등 여러 하위 기능을 포함

```c
class EmittedParticleSystem {
    GPUBuffer particleBuffer;      // 파티클 데이터
    GPUBuffer aliveList[2];         // 살아있는 파티클 목록
    GPUBuffer deadList;             // 죽은 파티클 목록
    GPUBuffer distanceBuffer;       // 정렬용
    GPUBuffer sphGridCells;         // SPH 유체 시뮬레이션
    GPUBuffer densityBuffer;        // 밀도 계산

    // 이것은 단순한 데이터가 아니라 "시스템"처럼 복잡함!
};
```

  | ECS 요소    | WickedEngine                                                    |
  |-----------|-----------------------------------------------------------------|
  | Entity    | wi::ecs::Entity (고유 ID)                                         |
  | Component | EmittedParticleSystem, HairParticleSystem, TransformComponent 등 |
  | System    | RunParticleUpdateSystem(), RunTransformUpdateSystem() 등         |

Components.h 에 새로운 구조체(EmittedParticleComponent) 추가 (encapsule 확실하게 wicked 랑은 다르게)

해당 구조체를 상속 받는 GEmittedParticleComponent 를 GComponents.h 에 추가

	// 이런 함수는 셰이더 엔진에서 구현되어야함. component 내부 말고
	void EmittedParticleSystem::Draw(const MaterialComponent& material, CommandList cmd, const PARTICLESHADERTYPE* shadertype_override) const

    particlesystem_detail.cpp 같은 코드로 구현할 예정.
    이런 RenderPath2D_Detail, ShadowMap_Detail.cpp같은 코드의 헤더는 보통 어디서 구현되는지 찾아보기.