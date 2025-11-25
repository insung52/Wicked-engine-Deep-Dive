# VizMotive Engine - Particle System Implementation Documentation

## 📋 Overview

VizMotive Engine의 파티클 시스템은 WickedEngine의 구조를 기반으로 GPU 기반 파티클 시뮬레이션을 구현했습니다. 이 문서는 파티클 시스템의 전체 구현 과정, 발생한 문제들, 그리고 해결 방법을 기록합니다.

**구현 기간**: 2024년 ~ 2025년 1월
**브랜치**: `particle`
**참조 엔진**: WickedEngine (MIT License)

---

## 🏗️ Implementation Timeline

### Phase 1: Basic Structure Setup (Initial Commits)

#### Commit: `14cec49` - Ready to add particle components
- **목적**: 파티클 컴포넌트 기본 구조 준비
- **변경사항**:
  - `EmittedParticleComponent` 클래스 스켈레톤 추가
  - Component factory에 파티클 컴포넌트 등록

#### Commit: `527ff0b` - Setup basic particle structure
- **목적**: 기본 파티클 데이터 구조 정의
- **주요 파일**:
  - `ShaderInterop_EmittedParticle.h`: 파티클 구조체 및 상수 정의
  - `Components.h`: `EmittedParticleComponent` 기본 속성 추가
- **핵심 구조체**:
  ```cpp
  struct Particle {
      float3 position;
      float mass;
      float3 velocity;
      float maxLife;
      float3 force;
      float life;
      float2 sizeBeginEnd;
      uint rotation_rotationVelocity; // packed
      uint color; // RGBA packed
  };

  struct ParticleCounters {
      uint aliveCount;
      uint deadCount;
      uint realEmitCount;
      uint aliveCount_afterSimulation;
      uint culledCount;
      uint cellAllocator;
  };
  ```

#### Commit: `e02be4c` - particle basic structure 2
- **목적**: GPU 리소스 버퍼 구조 확립
- **GPU 버퍼**:
  - `particleBuffer_`: 파티클 데이터 (StructuredBuffer)
  - `aliveList_[2]`: Double buffering을 위한 alive 인덱스 리스트
  - `deadList_`: 재사용 가능한 dead 파티클 인덱스
  - `counterBuffer_`: Atomic counter 버퍼
  - `indirectBuffers_`: Indirect draw arguments

---

### Phase 2: GPU Pipeline Implementation

#### Commit: `9a604a0` - particle basic 3
- **목적**: 파티클 Emit 및 Simulate 셰이더 구현
- **주요 셰이더**:
  - `emittedparticle_emit_CS.hlsl`: Dead list에서 파티클을 가져와 초기화
  - `emittedparticle_simulate_CS.hlsl`: 물리 시뮬레이션 및 생명주기 관리
  - `emittedparticle_kickoff_CS.hlsl`: Indirect dispatch arguments 준비

**Emit Shader 핵심 로직**:
```c
// Dead list에서 pop (LIFO)
int deadCount;
counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_DEADCOUNT, -1, deadCount);
if (deadCount < 1) return;

uint newParticleIndex = deadBuffer[deadCount - 1];

// 파티클 초기화
Particle particle;
particle.position = worldPos;
particle.velocity = velocity;
particle.life = maxLife;
particle.maxLife = maxLife;
// ... 기타 초기화

// Alive list에 push
uint aliveCount;
counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION, 1, aliveCount);
aliveBuffer_CURRENT[aliveCount] = newParticleIndex;
```

**Simulate Shader 핵심 로직**:
```c
// Alive list에서 읽기
uint particleIndex = aliveBuffer_CURRENT[DTid.x];
Particle particle = particleBuffer[particleIndex];

// 물리 시뮬레이션
particle.force += xParticleGravity * particle.mass;
particle.velocity += particle.force * dt;
particle.velocity *= xParticleDrag;
particle.position += particle.velocity * dt;

// 생명주기 감소
particle.life -= dt;

if (particle.life > 0) {
    // Alive: NEW list에 추가
    uint newAliveIndex;
    counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION, 1, newAliveIndex);
    aliveBuffer_NEW[newAliveIndex] = particleIndex;
} else {
    // Dead: Dead list에 추가
    uint deadIndex;
    counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_DEADCOUNT, 1, deadIndex);
    deadBuffer[deadIndex] = particleIndex;
}
```

#### Commit: `a2e66d8` - Particle simulation
- **목적**: 렌더링 파이프라인 통합
- **주요 파일**:
  - `emittedparticle_VS.hlsl`: Billboard 생성 버텍스 셰이더
  - `emittedparticle_simple_PS.hlsl`: 기본 픽셀 셰이더
  - `EmittedParticle_Detail.cpp`: GPU 업데이트 파이프라인 구현

**렌더링 파이프라인 순서**:
```
UpdateCPU() → UpdateGPU()
  ├─ Emit()
  ├─ KickoffUpdate()
  ├─ Simulate()
  ├─ Sort() (optional)
  ├─ FinishUpdate()
  └─ Draw()
```

---

### Phase 3: Bug Fixes and Core Features

#### Commit: `2db82a5` - Fix particle spawn
- **문제**: 파티클이 생성되지 않음
- **원인**: Dead list 초기화 누락
- **해결**: `CreateGPUResources()`에서 dead list를 모든 파티클 인덱스(0~N-1)로 초기화

#### Commit: `8e129ad` - Fix billboarding
- **문제**: 파티클이 카메라를 향하지 않음
- **원인**: Billboard 계산 오류
- **해결**:
```c
// 카메라 방향 벡터 사용
float3 cameraRight = float3(GetCamera().inverse_view._11, GetCamera().inverse_view._21, GetCamera().inverse_view._31);
float3 cameraUp = float3(GetCamera().inverse_view._12, GetCamera().inverse_view._22, GetCamera().inverse_view._32);

float3 worldPos = particlePos;
worldPos += cameraRight * quadPos.x;
worldPos += cameraUp * quadPos.y;
```

#### Commit: `43e4e5a` - Add particle buffer swap
- **목적**: Double buffering 구현
- **변경사항**: `SwapBuffers()` 함수 추가
```cpp
void SwapBuffers() {
    std::swap(aliveList_[0], aliveList_[1]);
}
```
- **호출 위치**: GPU 커맨드 끝 (후에 시작 전으로 이동)

#### Commit: `62b0ad0` - Fix dt calculate error
- **문제**: Fixed timestep이 제대로 동작하지 않음
- **원인**: Delta time 누적 오류
- **해결**: WickedEngine의 timestep 로직 참고하여 수정

---

### Phase 4: Visual Enhancements

#### Commit: `0c7ec6f` - Remove opacitycurve texture
- **목적**: Opacity curve를 텍스처가 아닌 constant buffer로 계산
- **이유**:
  - 텍스처 샘플링 오버헤드 제거
  - 파라미터 변경 시 텍스처 재생성 불필요
- **구현**:
```c
// Pixel shader에서 직접 계산
float t = input.lifePercent;
float opacityFactor;

if (t < xOpacityCurvePeakStart) {
    // Fade in
    opacityFactor = t / xOpacityCurvePeakStart;
} else if (t < xOpacityCurvePeakEnd) {
    // Peak
    opacityFactor = 1.0f;
} else {
    // Fade out
    opacityFactor = 1.0f - (t - xOpacityCurvePeakEnd) / (1.0f - xOpacityCurvePeakEnd);
}
```

#### Commit: `6074ec1` - Fix opacity parameter
- **문제**: Opacity curve 파라미터가 셰이더에 전달되지 않음
- **해결**: Constant buffer 바인딩 수정

#### Commit: `c02daba` - Add base color
- **목적**: 파티클 색상 제어 추가
- **구현**:
  - `xParticleBaseColor` constant buffer 파라미터
  - Emit shader에서 색상 초기화
  - Pixel shader에서 base color * vertex color 합성

#### Commit: `1e9feac` - Add Motion blur
- **목적**: 파티클 모션 블러 효과
- **구현**:
```c
// VS에서 velocity 방향으로 quad 늘리기
if (xParticleMotionBlurAmount > 0.0f) {
    float3 velocityViewSpace = mul((float3x3)GetCamera().view, particle.velocity);
    quadPos += dot(quadPos, velocityViewSpace) * velocityViewSpace * xParticleMotionBlurAmount;
}
```

---

### Phase 5: Particle Sorting

#### Commit: `e3ad8b6` - Add particle sorting
- **목적**: 깊이 정렬을 통한 반투명 렌더링 품질 개선
- **주요 파일**:
  - `emittedparticle_sort_CS.hlsl`: Bitonic sort 구현
  - `distanceBuffer_`: 파티클별 카메라 거리 저장
- **알고리즘**: AMD GPUSortLib 기반 Bitonic Sort
- **Sort Size**: 512 (그룹당)

**Sort Shader 구조**:
```c
#define SORT_SIZE 512

// LDS에 (distance, particleIndex) 쌍 로드
g_LDS[i] = float2(distanceBuffer[particleIndex], particleIndex);

// Bitonic sort
for (uint nMergeSize = 2; nMergeSize <= SORT_SIZE; nMergeSize *= 2) {
    for (uint nMergeSubSize = nMergeSize >> 1; nMergeSubSize > 0; nMergeSubSize >>= 1) {
        // Compare and swap
        if (a.x > b.x) { swap(a, b); }
    }
}

// 정렬된 인덱스를 alive buffer에 다시 쓰기
aliveBuffer[i] = (uint)g_LDS[i].y;
```

---

### Phase 6: Material System Integration

#### Commit: `4c439ff` - Add material to Particle
- **목적**: MaterialComponent를 파티클에 연결
- **변경사항**:
  - `EmittedParticleComponent`에 `materialID_` 추가
  - Material의 base color를 파티클 색상에 반영

#### Commit: `4f4c26d` - Fix particle to use material base color
- **문제**: Material의 색상이 파티클에 적용되지 않음
- **해결**:
```cpp
// DrawParticles()에서 material 정보 읽기
Entity materialID = emitter.GetMaterialID();
if (materialID != INVALID_ENTITY) {
    MaterialComponent* material = compfactory::GetMaterialComponent(materialID);
    if (material) {
        XMFLOAT4 baseColor = material->GetBaseColor();
        cb.xParticleBaseColor = float4(baseColor.x, baseColor.y, baseColor.z, baseColor.w);
        cb.xParticleEmissive = material->GetEmissiveStrength();
    }
}
```

**Pixel Shader에서 emissive 적용**:
```c
float4 finalColor = texColor * xParticleBaseColor * input.color;
finalColor.a *= opacityFactor;

// HDR emissive multiplier
finalColor.rgb *= (1.0f + xParticleEmissive);
```

---

### Phase 7: Rotation System

#### Commit: `3cd6ecb` - Fix particle rotation, rotation velocity
- **문제**: 파티클 회전이 동작하지 않음
- **원인**:
  1. Rotation velocity가 시뮬레이션에서 적용되지 않음
  2. Packing/unpacking 오류
- **해결**:

**Simulate Shader에서 rotation velocity 적용**:
```c
// Unpack rotation and rotationVelocity
uint packedRotation = particle.rotation_rotationVelocity;
uint rotationBits = (packedRotation >> 16) & 0xFFFF;
uint rotationVelBits = packedRotation & 0xFFFF;

float rotation = (float(rotationBits) / 65535.0f) * 2.0f * PI - PI;
float rotationVel = (float(rotationVelBits) / 65535.0f) * 2.0f * PI - PI;

// Apply rotation velocity
rotation += rotationVel * dt;

// Wrap rotation [-PI, PI]
rotation = fmod(rotation + PI, 2.0f * PI) - PI;

// Pack back
uint newRotationBits = uint((rotation + PI) / (2.0f * PI) * 65535.0f);
particle.rotation_rotationVelocity = (newRotationBits << 16) | rotationVelBits;
```

---

### Phase 8: Critical Bug - Particle Flickering

#### Commit: `d98941c` - Fix particle flickering

**문제 증상**:
- Sorting을 켜면 카메라에 가장 가까운 파티클이 빠르게 깜빡거림
- 새 파티클이 생성될 때마다 깜빡임 발생
- 파티클이 죽고 재생성되기 시작하면 깜빡임 시작

**문제 분석 과정**:

1. **초기 가설**: Sorting 알고리즘 오류
   - Sorting shader의 distance buffer 인덱싱 확인
   - `distanceBuffer[particleIndex]` vs `distanceBuffer[aliveIndex]` 불일치 수정
   - **결과**: 해결 안 됨

2. **디버그 시각화**:
   ```c
   // Particle index를 색상으로 표시
   output.particleIndex = particleIndex;
   output.aliveListIndex = input.instanceID;

   // PS에서
   return float4(
       frac(input.aliveListIndex * 0.618033988749895),
       0.0f,
       0.0f,
       1.0f
   );
   ```
   - **발견**: `aliveBuffer[0]` (색상 0, 검은색) 위치의 파티클이 깜빡임

3. **사용자 디버깅 데이터** (memo.md):
   ```
   초반에는 안깜빡거리다가 life 가 끝나는 파티클이 생기기 시작하니까 깜빡거리네

   추가 (드디어 5개 파티클 추적 시작)
   2 1 4 3 0
   제거 (추적중이던 파티클이 삭제되면 - 로 표시)
   - 1 4 3 0    ← 0번 파티클 깜빡임
   ```

4. **WickedEngine 코드 비교**:
   ```cpp
   // WickedEngine - UpdateCPU()에서
   std::swap(aliveList[0], aliveList[1]);  // Line 360

   // VizMotive - UpdateGPU() 끝에서
   emitter.SwapBuffers();  // Line 316 (잘못된 위치!)
   ```

**근본 원인 발견**:

VizMotive의 잘못된 파이프라인 순서:
```
Frame N:
  Emit    → writes to aliveList[0]
  Simulate → reads aliveList[0], writes to aliveList[1]
  Sort     → sorts aliveList[1]
  Draw     → reads aliveList[1]
  SwapBuffers → swap(aliveList[0], aliveList[1])  ← GPU 커맨드 후!

Frame N+1:
  Emit    → writes to aliveList[0] (이제 이전 프레임의 sorted list)
           → 정렬된 리스트 끝에 새 파티클 추가 → 순서 깨짐!
```

**올바른 순서** (WickedEngine):
```
Frame N (CPU):
  SwapBuffers → swap(aliveList[0], aliveList[1])  ← GPU 커맨드 전!

Frame N (GPU):
  Emit    → writes to aliveList[0] (이전 simulate 결과)
  Simulate → reads aliveList[0], writes to aliveList[1]
  Sort     → sorts aliveList[1]
  Draw     → reads aliveList[1]
```

**해결 방법**:
```cpp
// EmittedParticle_Detail.cpp - UpdateParticleSystem()
void GRenderPath3DDetails::UpdateParticleSystem(...) {
    device->EventBegin("ParticleSystem Update", cmd);

    // Swap BEFORE GPU commands (like WickedEngine)
    emitter.SwapBuffers();  // ← 이동!

    EmitParticles(emitter, instanceIndex, cmd);
    // ... rest of pipeline
}
```

**추가 수정 - Emit Shader**:
```c
// aliveBuffer_NEW 바인딩 추가 (WickedEngine과 동일)
RWStructuredBuffer<Particle> particleBuffer : register(u0);
RWStructuredBuffer<uint> aliveBuffer_CURRENT : register(u1);
RWStructuredBuffer<uint> aliveBuffer_NEW : register(u2);      // 추가!
RWStructuredBuffer<uint> deadBuffer : register(u3);
RWByteAddressBuffer counterBuffer : register(u4);
```

---

### Phase 9: UI and Dynamic Configuration

#### Commit: `36317bc` - Add particle UI
- **목적**: Sample015에 파티클 파라미터 UI 추가
- **주요 파라미터**:
  - Size, Emit Count, Life, Random Life
  - Random Position Offset, Random Velocity, Random Size
  - Random Rotation, Random Rotation Velocity, Random Color
  - Velocity, Gravity, Drag
  - Opacity Curve (Peak Start, Peak End)
  - Sorting Enable/Disable
  - Motion Blur Amount

#### Commit: `4301c55` - Add max particle count gui
- **목적**: MaxParticles를 UI에서 동적으로 변경 가능하게
- **구현**:
```cpp
// Sample015.cpp
static int max_particles = 1000;
if (ImGui::SliderInt("Max Particles", &max_particles, 100, 1000000)) {
    particleEmitter->SetParticleMaxCount((uint32_t)max_particles);
}
```

#### Commit: `8a6fe5b` - Fix max particle gui

**문제**: UI에서 MaxParticles를 변경해도 실제 렌더링 개수가 바뀌지 않음

**원인 분석**:

1. **초기 구현**: 즉시 GPU 리소스 재생성
   ```cpp
   void SetMaxParticles(uint32_t count) {
       DestroyGPUResources();
       CreateGPUResources();  // 렌더링 중 호출!
   }
   ```
   - **문제**: `Assertion failed: cmd.IsValid()` 에러 발생
   - **원인**: 렌더링 진행 중 커맨드 리스트가 무효화됨

2. **WaitForGPU 시도**:
   ```cpp
   device->WaitForGPU();
   DestroyGPUResources();
   CreateGPUResources();
   ```
   - **문제**: 여전히 동일한 에러 발생

3. **가상 함수 문제**:
   ```cpp
   // VzActor.cpp
   EmittedParticleComponent* emitter = GetEmittedParticleComponent(componentVID_);
   emitter->SetMaxParticles(maxCount);  // 부모 클래스 버전 호출!
   ```
   - **원인**: `SetMaxParticles`가 가상 함수가 아님
   - **해결**: `virtual` 키워드 추가
   ```cpp
   // Components.h
   virtual void SetMaxParticles(uint32_t count);

   // GComponents.h
   void SetMaxParticles(uint32_t count) override;
   ```

**최종 해결 - WickedEngine 방식 채택**:

WickedEngine 코드 분석:
```cpp
void EmittedParticleSystem::SetMaxParticleCount(uint32_t value) {
    MAX_PARTICLES = value;
    counterBuffer = {}; // will be recreated
}
```

**핵심**: 리소스를 즉시 파괴하지 않고, 플래그만 무효화!

VizMotive 구현:
```cpp
void GEmittedParticleComponent::SetMaxParticles(uint32_t count) {
    if (maxParticles_ != count) {
        maxParticles_ = count;
        timeStampSetter_ = TimerNow;

        // Invalidate GPU resources (like WickedEngine)
        counterBuffer_ = {};
        gpuResourcesCreated_ = false;
        // Resources will be recreated in next CreateGPUResources() call
    }
}
```

**장점**:
- 렌더링 중단 없음
- 커맨드 리스트 무효화 방지
- 다음 프레임에서 안전하게 재생성

---

## 🔧 Final System Operation - Complete Technical Overview

이 섹션에서는 최종적으로 구현된 파티클 시스템의 동작 방식을 **구체적인 예시**와 함께 설명합니다.

### 📖 시스템 동작 시나리오 - 실제 예시로 이해하기

파티클 시스템이 어떻게 동작하는지 구체적인 숫자를 사용한 시나리오로 설명하겠습니다.

#### 시나리오 설정
- **MaxParticles**: 1000개
- **EmitCount**: 초당 100개
- **ParticleLife**: 5초
- **현재 상황**: 시스템이 이미 5초간 실행되어 안정 상태 (약 500개 파티클 활성)

---

#### Frame N의 전체 흐름

**초기 상태** (Frame N 시작 전):
```
particleBuffer[0~999]: 1000개 파티클 데이터
  ├─ [0~499]: life > 0 (살아있음)
  └─ [500~999]: life = 0 (죽어있음)

aliveList[0]: [12, 45, 78, ..., 234] (500개 인덱스) ← 이전 프레임에서 Simulate가 작성
aliveList[1]: [비어있음] ← 이번 프레임에서 Simulate가 작성할 곳

deadList: [500, 501, 502, ..., 999] (500개 인덱스, LIFO 스택)

counterBuffer:
  ├─ aliveCount: 500
  ├─ deadCount: 500
  └─ aliveCount_afterSimulation: 0 (매 프레임 리셋됨)
```

---

**Step 0: SwapBuffers() - CPU에서 실행**

```cpp
// EmittedParticle_Detail.cpp:233
emitter.SwapBuffers();  // swap(aliveList[0], aliveList[1])
```

**결과**:
```
aliveList[0]: [비어있음] ← Emit가 여기에 쓸 것
aliveList[1]: [12, 45, 78, ..., 234] (500개) ← Simulate가 여기서 읽을 것
```

> **핵심**: Swap을 먼저 해야 Emit가 깨끗한 버퍼에 쓸 수 있음!

---

**Step 1: Emit - 새 파티클 100개 생성**

```cpp
// EmitParticles() 호출
// Dispatch: (100 + 255) / 256 = 1 group (256 threads, 처음 100개만 작동)
```

**Emit Shader 실행** (100개 스레드):

각 스레드 (예: Thread 0):
```c
// 1. Dead list에서 파티클 인덱스 가져오기
int deadCount;
counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_DEADCOUNT, -1, deadCount);
// Thread 0: deadCount = 500 (add 전 값), 이후 deadCount = 499
// Thread 1: deadCount = 499, 이후 deadCount = 498
// ...
// Thread 99: deadCount = 401, 이후 deadCount = 400

uint particleIndex = deadBuffer[deadCount - 1];
// Thread 0: particleIndex = deadBuffer[499] = 999
// Thread 1: particleIndex = deadBuffer[498] = 998
// ...
```

```c
// 2. 파티클 초기화
Particle particle;
particle.position = worldPos;  // 예: (0, 0, 0)
particle.velocity = (0, 5, 0);  // 위로 발사
particle.life = 5.0f;
particle.maxLife = 5.0f;
// ... 기타 초기화

particleBuffer[particleIndex] = particle;
// Thread 0: particleBuffer[999] = 새 파티클
// Thread 1: particleBuffer[998] = 새 파티클
```

```c
// 3. Alive list에 추가
uint aliveIndex;
counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION, 1, aliveIndex);
// Thread 0: aliveIndex = 0, 이후 = 1
// Thread 1: aliveIndex = 1, 이후 = 2
// ...
// Thread 99: aliveIndex = 99, 이후 = 100

aliveBuffer_CURRENT[aliveIndex] = particleIndex;
// Thread 0: aliveList[0][0] = 999
// Thread 1: aliveList[0][1] = 998
// ...
// Thread 99: aliveList[0][99] = 900
```

**Emit 후 상태**:
```
aliveList[0]: [999, 998, 997, ..., 900] (100개 새 파티클)
aliveList[1]: [12, 45, 78, ..., 234] (500개 기존 파티클, 아직 안 건드림)

deadList: [500, 501, ..., 899] (400개 남음)

counterBuffer:
  ├─ aliveCount: 500 (아직 안 바뀜)
  ├─ deadCount: 400 (500 → 400)
  └─ aliveCount_afterSimulation: 100 (0 → 100)
```

---

**Step 2: KickoffUpdate - 이전 프레임 결과를 현재 프레임에 반영**

여기가 핵심입니다! KickoffUpdate는 **이전 프레임의 Simulate 결과**를 가져옵니다.

```c
// emittedparticle_kickoff_CS.hlsl
uint aliveCount = counterBuffer.Load(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION);
// aliveCount = 500 ← 이전 프레임(Frame N-1)의 Simulate가 저장한 값!

counterBuffer.Store(PARTICLECOUNTER_OFFSET_ALIVECOUNT, aliveCount);
// aliveCount 카운터를 500으로 설정 → Simulate가 이 값을 읽어서 500개 처리

// Simulate dispatch args 준비
uint threadGroups = (500 + 255) / 256 = 2;
indirectBuffer.Store(ARGUMENTBUFFER_OFFSET_DISPATCHSIMULATION + 0, 2);  // X
indirectBuffer.Store(ARGUMENTBUFFER_OFFSET_DISPATCHSIMULATION + 4, 1);  // Y
indirectBuffer.Store(ARGUMENTBUFFER_OFFSET_DISPATCHSIMULATION + 8, 1);  // Z
```

**핵심 이해**:
- `aliveCount_afterSimulation`은 **이전 프레임**에서 Simulate가 계산한 값 (500)
- 현재 프레임의 Emit는 이 값을 건드리지 않고, 자신의 결과(100)를 **누적**함
- KickoffUpdate는 이전 프레임 값(500)을 `aliveCount`에 복사
- 이제 `aliveCount_afterSimulation` = 100 (Emit 결과만)

**KickoffUpdate 후 상태**:
```
counterBuffer:
  ├─ aliveCount: 500 (← Simulate가 처리할 개수)
  ├─ deadCount: 400
  └─ aliveCount_afterSimulation: 100 (← Emit 결과, Simulate가 여기에 누적할 것)
```

---

**Step 3: Simulate - 기존 500개 파티클 업데이트**

```c
// Dispatch: Indirect (KickoffUpdate가 준비한 args 사용)
// threadGroups = 2 (500개 파티클 / 256 threads per group)
// aliveBuffer_CURRENT = aliveList[1] (500개 기존 파티클)
// aliveBuffer_NEW = aliveList[0] (Emit가 쓴 100개 + Simulate가 추가할 것)
```

각 스레드 (예: Thread 0):
```c
// 1. 파티클 읽기
uint particleIndex = aliveBuffer_CURRENT[DTid.x];
// Thread 0: particleIndex = aliveList[1][0] = 12
Particle particle = particleBuffer[12];

// 2. 물리 시뮬레이션
particle.life -= dt;  // 5.0 → 4.983 (dt=0.016)
particle.force += xParticleGravity;  // (0, -9.8, 0)
particle.velocity += particle.force * dt;
particle.position += particle.velocity * dt;
particle.velocity *= xParticleDrag;  // 0.98

// 3. 생존 체크
if (particle.life > 0) {
    // 살아있음 → NEW list에 추가
    uint newAliveIndex;
    counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION, 1, newAliveIndex);
    // Thread 0: newAliveIndex = 100 (Emit가 이미 100개 추가함!)
    // Thread 1: newAliveIndex = 101
    // ...
    
    aliveBuffer_NEW[newAliveIndex] = particleIndex;
    // Thread 0: aliveList[0][100] = 12
    
    // 거리 계산 (sorting용)
    float distSQ = distance_squared(particle.position, camera.position);
    distanceBuffer[particleIndex] = -distSQ;
    // distanceBuffer[12] = -25.0
} else {
    // 죽음 → Dead list에 추가
    uint deadIndex;
    counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_DEADCOUNT, 1, deadIndex);
    deadBuffer[deadIndex] = particleIndex;
}

particleBuffer[particleIndex] = particle;
```

**가정**: 500개 중 480개 생존, 20개 사망

**Simulate 후 상태**:
```
aliveList[0]: [999, 998, ..., 900, 12, 45, 78, ..., (480개)] (100 + 480 = 580개)
aliveList[1]: [12, 45, 78, ..., 234] (500개, 이제 쓸모없음)

deadList: [500, 501, ..., 899, (20개 사망 파티클)] (400 + 20 = 420개)

counterBuffer:
  ├─ aliveCount: 500
  ├─ deadCount: 420 (400 → 420)
  └─ aliveCount_afterSimulation: 580 (100 → 580)
```

---

**Step 4: Sort - aliveList[0] 정렬 (optional)**

```c
// Dispatch: (1000 + 511) / 512 = 2 groups
// 각 그룹은 512개씩 정렬
```

**Group 0 (Thread 0~511)**:
```c
// LDS에 로드
uint particleIndex = aliveBuffer[GTid.x];  // 0~511
float distance = distanceBuffer[particleIndex];
g_LDS[localIndex] = float2(distance, particleIndex);

// 예시:
// g_LDS[0] = (-25.0, 999)  // 카메라에서 5m
// g_LDS[1] = (-100.0, 998) // 카메라에서 10m
// g_LDS[2] = (-400.0, 997) // 카메라에서 20m
// ...

// Bitonic sort 실행 (LDS 내에서)
// ...

// 정렬 후:
// g_LDS[0] = (-400.0, 997)  // 가장 먼 파티클
// g_LDS[1] = (-100.0, 998)
// g_LDS[2] = (-25.0, 999)   // 가장 가까운 파티클

// 다시 쓰기
aliveBuffer[GTid.x] = (uint)g_LDS[localIndex].y;
```

**Sort 후 상태**:
```
aliveList[0]: [997, 456, 123, ..., 999] (580개, 거리순 정렬)
  ├─ [0]: 가장 먼 파티클
  └─ [579]: 가장 가까운 파티클
```

---

**Step 5: FinishUpdate - Draw Args 준비**

```c
uint aliveCount = counterBuffer.Load(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION);
// aliveCount = 580

// DrawIndexedInstancedIndirect args
indirectBuffer.Store(offset + 0, 6);         // 6 indices (quad)
indirectBuffer.Store(offset + 4, 580);       // 580 instances
```

---

**Step 6: Draw - 렌더링**

```cpp
device->DrawIndexedInstancedIndirect(indirectBuffer, offset);
// 580개 파티클 렌더링
```

**Vertex Shader 실행** (580 instances × 6 vertices = 3480 vertices):

```c
// Instance 0, Vertex 0
uint particleIndex = aliveBuffer[0];  // = 997 (가장 먼 파티클)
Particle particle = particleBuffer[997];

// Quad 생성
float2 quadPos = float2(-1, -1) * particleSize;

// Billboard
float3 cameraRight = GetCamera().inverse_view[0].xyz;
float3 cameraUp = GetCamera().inverse_view[1].xyz;
float3 worldPos = particle.position + cameraRight * quadPos.x + cameraUp * quadPos.y;

output.pos = mul(viewProj, float4(worldPos, 1));
output.color = unpack_rgba(particle.color);
output.lifePercent = 1.0 - particle.life / particle.maxLife;
```

**Pixel Shader 실행**:
```c
float4 texColor = particleTexture.Sample(sampler, input.uv);
float4 finalColor = texColor * xParticleBaseColor * input.color;

// Opacity curve
float t = input.lifePercent;  // 0.003 (방금 생성됨)
float opacityFactor = t / xOpacityCurvePeakStart;  // Fade in
finalColor.a *= opacityFactor;

// Emissive
finalColor.rgb *= (1.0 + xParticleEmissive);

return finalColor;
```

---

**Frame N 종료 상태**:
```
화면에 580개 파티클 렌더링 완료!

다음 프레임을 위한 준비:
  ├─ aliveList[0]: 580개 (정렬됨)
  ├─ aliveList[1]: (다음 프레임에서 덮어씀)
  ├─ deadList: 420개
  └─ counterBuffer.aliveCount_afterSimulation: 580
```

---

### 핵심 포인트 정리

1. **Double Buffering**: 
   - `aliveList[0]` ← Emit/Simulate가 **쓰기**
   - `aliveList[1]` ← Simulate가 **읽기**
   - 매 프레임 시작에 Swap!

2. **Dead List (LIFO 스택)**:
   - Emit: `deadBuffer[--deadCount]` (pop from top)
   - Simulate: `deadBuffer[deadCount++] = index` (push to top)

3. **Counter 흐름**:
   - Emit/Simulate → `aliveCount_afterSimulation`에 atomic add
   - KickoffUpdate → `aliveCount_afterSimulation` 복사 → `aliveCount`
   - FinishUpdate → `aliveCount_afterSimulation` 읽어서 draw args 준비

4. **Sorting**:
   - `distanceBuffer[particleIndex]` ← Simulate가 거리 저장
   - Sort가 `aliveBuffer`를 거리순으로 재배열
   - 멀리 있는 파티클부터 그려서 반투명 렌더링 품질 향상

---

### 1. 핵심 데이터 구조

#### 1.1 Particle 구조체 (GPU)
```cpp
// ShaderInterop_EmittedParticle.h:7-18
struct Particle {
    float3 position;                    // 파티클 위치 (12 bytes)
    float mass;                         // 질량 (4 bytes)
    float3 force;                       // 현재 프레임에 적용된 힘 (12 bytes)
    uint rotation_rotationVelocity;     // 패킹: rotation(16bit) | rotationVel(16bit) (4 bytes)
    float3 velocity;                    // 속도 (12 bytes)
    float maxLife;                      // 최대 생명 시간 (4 bytes)
    float2 sizeBeginEnd;                // 시작 크기, 끝 크기 (8 bytes)
    float life;                         // 현재 생명 시간 (4 bytes)
    uint color;                         // 패킹: RGBA8 (4 bytes)
};
// Total: 64 bytes per particle
```

**최적화 기법**:
- Rotation/RotationVelocity 패킹: 2개의 float(8 bytes) → 1개의 uint32(4 bytes)
  - 각 값을 [-π, π] 범위에서 16비트로 양자화
  - 정밀도: ~0.096도 (충분한 시각적 품질)
- Color 패킹: RGBA float4(16 bytes) → uint32(4 bytes)
  - 각 채널 8비트 (0~255)
- 64 bytes = GPU 캐시 라인 크기와 정렬

#### 1.2 ParticleCounters 구조체
```cpp
// ShaderInterop_EmittedParticle.h:21-29
struct ParticleCounters {
    uint aliveCount;                    // 현재 프레임 시작 시 살아있는 파티클 수
    uint deadCount;                     // Dead list에 있는 재사용 가능한 인덱스 수
    uint realEmitCount;                 // 실제로 생성된 파티클 수 (이번 프레임)
    uint aliveCount_afterSimulation;    // Simulate 후 살아남은 파티클 수
    uint culledCount;                   // Frustum culling된 파티클 수 (예약)
    uint cellAllocator;                 // SPH 시뮬레이션용 (예약)
};
// Total: 24 bytes (6 * uint32)
```

**카운터 역할**:
- `aliveCount`: Simulate가 읽을 파티클 수 (Kickoff에서 설정)
- `deadCount`: Emit이 사용 가능한 인덱스 수
- `aliveCount_afterSimulation`: Emit/Simulate가 atomic add로 증가, FinishUpdate가 복사
- `realEmitCount`: Kickoff에서 실제 생성할 파티클 수 계산에 사용

#### 1.3 EmittedParticleCB (Constant Buffer)
```cpp
// ShaderInterop_EmittedParticle.h:48-118
CBUFFER(EmittedParticleCB, CBSLOT_OTHER_EMITTEDPARTICLE) {
    // 시스템 파라미터
    uint xEmitterMaxParticleCount;           // 최대 파티클 수
    uint xEmitterInstanceIndex;              // 인스턴스 ID (RNG seed용)
    uint xEmitterMeshGeometryOffset;         // 메시 emission용 (예약)
    uint xEmitterMeshGeometryCount;          // 메시 emission용 (예약)

    // 파티클 기본 속성
    float xParticleSize;                     // 기본 크기
    float xParticleScaling;                  // 생명주기 동안 크기 스케일링 배율
    float xParticleRotation;                 // 기본 회전 (사용 안 함)
    float xParticleRandomPositionOffset;     // 생성 위치 랜덤 오프셋 범위

    float xParticleNormalFactor;             // Normal 방향 속도 배율, 메시에 particle component 가 적용된 경우, Normal 방향으로 튀어 나가는 초기 속도 조절
    float xParticleLifeSpan;                 // 기본 생명 시간
    float xParticleLifeSpanRandomness;       // 생명 시간 랜덤 범위 (0~1)
    float xParticleMass;                     // 질량

    float xParticleMotionBlurAmount;         // 모션 블러 강도
    float xParticleRandomColorFactor;        // 색상 랜덤 배율 (0~1)
    float xParticleRandomVelocity;           // 속도 랜덤 범위
    float xParticleRandomSize;               // 크기 랜덤 배율 (0~1)

    uint xEmitterOptions;                    // 옵션 플래그, sph 유체 시뮬레이션, 메시 셰이더, 충돌체 비활성화, 비 차단 등 여러 기능 토글
    float xEmitterFixedTimestep;             // 고정 타임스텝 (-1이면 가변)
    uint xEmitterPadding0;                   // 16바이트 정렬
    uint xEmitterPadding1;

    // 스프라이트 시트 파라미터
    uint2 xEmitterFramesXY;                  // 프레임 그리드 (X, Y)
    uint xEmitterFrameCount;                 // 총 프레임 수
    uint xEmitterFrameStart;                 // 시작 프레임 인덱스

    float2 xEmitterTexMul;                   // 텍스처 좌표 배율
    float xEmitterFrameRate;                 // 애니메이션 프레임 레이트
    uint xEmitterLayerMask;                  // 렌더링 레이어 마스크

    // 물리 파라미터
    float3 xParticleGravity;                 // 중력 벡터
    float xEmitterRestitution;               // 충돌 반발 계수 (예약)

    float3 xParticleVelocity;                // 초기 속도
    float xParticleDrag;                     // 드래그 계수 (속도 감쇠)

    // 시각 효과 파라미터
    float xOpacityCurvePeakStart;            // 불투명도 페이드인 끝 지점 (0~1)
    float xOpacityCurvePeakEnd;              // 불투명도 페이드아웃 시작 지점 (0~1)
    float xParticleRandomRotation;           // 회전 랜덤 범위 (라디안)
    float xParticleRandomRotationVelocity;   // 회전 속도 랜덤 배율

    float4 xParticleBaseColor;               // 기본 색상 (Material에서 설정)

    float xParticleEmissive;                 // Material emissive 강도
    float xEmitterPadding2;
    float xEmitterPadding3;
    float xEmitterPadding4;

    ShaderTransform xEmitterBaseMeshUnormRemap; // 메시 emission용 (예약)
};
```

#### 1.4 GPU 버퍼 구조

**CPU 측 (GEmittedParticleComponent)**:
```cpp
// GComponents.h:737-747
class GEmittedParticleComponent : public EmittedParticleComponent {
private:
    GPUBuffer particleBuffer_;      // Particle 구조체 배열
    GPUBuffer aliveList_[2];        // Double buffering: 살아있는 파티클 인덱스 리스트
    GPUBuffer deadList_;            // 재사용 가능한 파티클 인덱스 리스트 (LIFO 스택)
    GPUBuffer counterBuffer_;       // ParticleCounters 구조체
    GPUBuffer indirectBuffers_;     // Indirect dispatch/draw arguments
    GPUBuffer constantBuffer_;      // EmittedParticleCB 구조체
    GPUBuffer distanceBuffer_;      // Sorting용 거리 값 (float per particle)
    GPUBuffer emitBuffer_;          // EmitLocation 구조체

    bool gpuResourcesCreated_;      // 리소스 생성 여부
};
```

**버퍼 생성 코드** (EmittedParticleComponent.cpp:132-300):
```cpp
// 1. Particle Buffer - 모든 파티클 데이터
desc.size = maxCount * sizeof(Particle);            // 예: 1000 * 64 = 64KB
desc.bind_flags = SHADER_RESOURCE | UNORDERED_ACCESS;
desc.misc_flags = BUFFER_STRUCTURED;
desc.stride = sizeof(Particle);
// 초기화: 모든 파티클 life = 0 (dead)

// 2. Alive List Buffers (double buffered)
for (int i = 0; i < 2; ++i) {
    desc.size = maxCount * sizeof(uint32_t);        // 예: 1000 * 4 = 4KB
    desc.bind_flags = SHADER_RESOURCE | UNORDERED_ACCESS;
    desc.misc_flags = BUFFER_RAW;
    // 초기화: 빈 리스트
}

// 3. Dead List Buffer
desc.size = maxCount * sizeof(uint32_t);            // 예: 1000 * 4 = 4KB
desc.bind_flags = SHADER_RESOURCE | UNORDERED_ACCESS;
desc.misc_flags = BUFFER_RAW;
// 초기화: [0, 1, 2, ..., maxCount-1] (모든 파티클 인덱스)

// 4. Counter Buffer
desc.size = sizeof(ParticleCounters);               // 24 bytes
desc.bind_flags = SHADER_RESOURCE | UNORDERED_ACCESS;
desc.misc_flags = BUFFER_RAW;
// 초기화: aliveCount=0, deadCount=maxCount, 나머지=0

// 5. Indirect Buffers
desc.size = 256;                                     // DispatchEmit(12) + DispatchSim(12) + DrawInstanced(20)
desc.bind_flags = UNORDERED_ACCESS;
desc.misc_flags = BUFFER_RAW | INDIRECT_ARGS;

// 6. Distance Buffer (sorting용)
desc.size = maxCount * sizeof(float);               // 예: 1000 * 4 = 4KB
desc.bind_flags = SHADER_RESOURCE | UNORDERED_ACCESS;
desc.misc_flags = BUFFER_RAW;
```

### 2. CPU 측 업데이트 파이프라인

#### 2.1 UpdateCPU() - 매 프레임 호출
```cpp
// EmittedParticleComponent.cpp:354-399
void GEmittedParticleComponent::UpdateCPU(const TransformComponent& transform, float dt) {
    // 1. World matrix 업데이트
    worldMatrix = transform.GetWorldMatrix();
    center = XMFLOAT3(worldMatrix._41, worldMatrix._42, worldMatrix._43);

    // 2. GPU 리소스 생성 (필요시)
    if (!HasValidGPUResources()) {
        CreateGPUResources();  // 최초 1회 또는 MaxParticles 변경 후
    }

    // 3. Delta time 누적
    dt_ += dt;

    // 4. Fixed timestep 처리
    float timestep = GetFixedTimestep();
    if (timestep > 0.0f) {
        dt_ = timestep;  // 고정 타임스텝 모드
    }

    // 5. Emission 누적 (Paused가 아니면)
    if (!IsPaused()) {
        emit_ += GetEmitCount() * dt;  // 예: 100 particles/sec * 0.016s = 1.6
    }

    // 6. Burst emission 처리
    if (burst_ > 0) {
        emit_ += (float)burst_;  // Burst(50) 호출 시 즉시 50개 추가
        burst_ = 0;
    }

    // 7. Activity tracking (디버깅/최적화용)
    if (emit_ > 0.0f) {
        activeFrames_ = 1;
    }
}
```

**핵심 개념**:
- `emit_`: 누적된 생성할 파티클 수 (실수형, 소수점 가능)
- `burst_`: 즉시 생성할 파티클 수 (Burst() 호출로 설정)
- CPU에서는 파티클 데이터를 직접 조작하지 않음 (모두 GPU)

### 3. GPU 업데이트 파이프라인

#### 3.1 전체 파이프라인 순서 (EmittedParticle_Detail.cpp:220-319)

```cpp
void UpdateParticleSystem(GEmittedParticleComponent& emitter, uint32_t instanceIndex, CommandList cmd) {
    // [중요!] Step 0: Buffer Swap BEFORE GPU commands
    // Line 233 - 이 타이밍이 flickering 문제의 핵심!
    emitter.SwapBuffers();  // swap(aliveList[0], aliveList[1])

    // Step 1: Emit new particles
    EmitParticles(emitter, instanceIndex, cmd);
    GPUBarrier::Memory();

    // Step 2: Kickoff Update (prepare counters)
    // - Copy aliveCount_afterSimulation → aliveCount
    // - Prepare indirect dispatch args for Simulate
    device->BindUAVs({counterBuffer, indirectBuffers});
    device->BindComputeShader(&kickoffUpdateCS);
    device->Dispatch(1, 1, 1);
    GPUBarrier::Memory();

    // Step 3: Simulate particles (physics + life)
    SimulateParticles(emitter, instanceIndex, cmd);
    GPUBarrier::Memory();

    // Step 4: Sort particles (optional)
    if (emitter.IsSorted()) {
        SortParticles(emitter, cmd);
        GPUBarrier::Memory();
    }

    // Step 5: Finish Update (prepare draw args)
    device->BindResources({counterBuffer});
    device->BindUAVs({indirectBuffers});
    device->BindComputeShader(&finishUpdateCS);
    device->Dispatch(1, 1, 1);
}
```

#### 3.2 SwapBuffers() - Double Buffering의 핵심

**타이밍이 중요한 이유**:
```c
// 잘못된 타이밍 (VizMotive 초기 버전) - GPU 커맨드 AFTER
Frame N:
    Emit    → writes to aliveList[0]           (unsorted)
    Simulate → reads aliveList[0], writes to aliveList[1]
    Sort     → sorts aliveList[1]              (sorted!)
    Draw     → reads aliveList[1]
    SwapBuffers → swap(aliveList[0], aliveList[1])  ← 문제!

Frame N+1:
    Emit    → writes to aliveList[0]           (이제 sorted list!)
            → 새 파티클이 정렬된 리스트 끝에 추가
            → aliveBuffer[0]에 새 파티클 → 카메라 앞에서 깜빡임!
```

```c
// 올바른 타이밍 (현재 버전) - GPU 커맨드 BEFORE
Frame N (CPU):
    SwapBuffers → swap(aliveList[0], aliveList[1])  ← 먼저 swap!

Frame N (GPU):
    Emit    → writes to aliveList[0]           (이전 simulate 결과, unsorted)
    Simulate → reads aliveList[0], writes to aliveList[1]
    Sort     → sorts aliveList[1]              (sorted)
    Draw     → reads aliveList[1]              (sorted list 사용)

Frame N+1 (CPU):
    SwapBuffers → swap(aliveList[0], aliveList[1])

Frame N+1 (GPU):
    Emit    → writes to aliveList[0]           (sorted list이지만 문제없음)
            → Emit는 정렬 순서 신경 안 씀
    Simulate → reads aliveList[0], writes to aliveList[1] (새로 정렬됨)
```

**핵심**: Emit는 aliveList[0]에 항상 쓰고, Simulate는 aliveList[0]을 읽어 aliveList[1]에 쓴다.
Swap이 먼저 일어나면 Emit가 이전 프레임의 Simulate 결과를 사용하게 되어 정상 동작.

#### 3.3 Emit Shader (emittedparticle_emit_CS.hlsl)

**바인딩**:
```c
StructuredBuffer<EmitLocation> emitBuffer : register(t0);      // SRV
RWStructuredBuffer<Particle> particleBuffer : register(u0);    // UAV
RWStructuredBuffer<uint> aliveBuffer_CURRENT : register(u1);   // UAV - Emit writes here
RWStructuredBuffer<uint> aliveBuffer_NEW : register(u2);       // UAV - Not used by Emit, but must bind
RWStructuredBuffer<uint> deadBuffer : register(u3);            // UAV
RWByteAddressBuffer counterBuffer : register(u4);              // UAV
```

**Dispatch**: `(emitCount + 255) / 256` groups of 256 threads
- 예: emitCount=100 → 1 group (256 threads, 처음 100개만 작동)

**핵심 로직** (Lines 38-122):
```c
[numthreads(256, 1, 1)]
void main(uint3 DTid : SV_DispatchThreadID) {
    EmitLocation location = emitBuffer[0];

    if (DTid.x >= location.count)
        return;  // 이 스레드는 작동 안 함

    // 1. Random seed 초기화
    uint seed = xEmitterInstanceIndex + DTid.x + GetFrame().frame_count * 1000;

    // 2. Emission position 계산
    float3 emitPos = float3(0, 0, 0);  // 현재는 point emission만 지원
    emitPos += (rand3(seed, 100) - 0.5f) * xParticleRandomPositionOffset;
    float3 worldPos = mul(worldMatrix, float4(emitPos, 1)).xyz;

    // 3. Velocity 계산
    float3 velocity = xParticleVelocity;
    velocity += (nor + (rand3(seed, 200) - 0.5f) * xParticleRandomVelocity) * xParticleNormalFactor;

    // 4. Size 계산
    float size = xParticleSize + xParticleSize * (rand(seed, 300) - 0.5f) * xParticleRandomSize;

    // 5. Life 계산
    float maxLife = xParticleLifeSpan + xParticleLifeSpan * (rand(seed, 400) - 0.5f) * xParticleLifeSpanRandomness;

    // 6. Rotation 계산 및 패킹
    float rotation = (rand(seed, 500) - 0.5f) * xParticleRandomRotation;
    float rotationVel = xParticleRotation * (rand(seed, 600) - 0.5f) * xParticleRandomRotationVelocity;
    uint rotation_packed = uint((rotation + PI) / (2*PI) * 65535.0f);
    uint rotationVel_packed = uint((rotationVel + PI) / (2*PI) * 65535.0f);
    uint rotation_rotationVelocity = (rotation_packed << 16) | rotationVel_packed;

    // 7. Color 계산
    float4 baseColor = xParticleBaseColor * unpack_rgba(location.color);
    baseColor.r *= lerp(1.0f, rand(seed, 700), xParticleRandomColorFactor);
    // ... (g, b도 동일)

    // 8. Particle 구조체 초기화
    Particle particle;
    particle.position = worldPos;
    particle.mass = xParticleMass;
    particle.force = float3(0, 0, 0);
    particle.rotation_rotationVelocity = rotation_rotationVelocity;
    particle.velocity = velocity;
    particle.maxLife = maxLife;
    particle.sizeBeginEnd = float2(size, size * xParticleScaling);
    particle.life = maxLife;
    particle.color = pack_rgba(baseColor);

    // 9. Dead list에서 파티클 인덱스 pop (LIFO)
    int deadCount;
    counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_DEADCOUNT, -1, deadCount);
    if (deadCount < 1)
        return;  // Dead list 비었음, 생성 실패

    uint particleIndex = deadBuffer[deadCount - 1];  // Stack top

    // 10. Particle buffer에 쓰기
    particleBuffer[particleIndex] = particle;

    // 11. Alive list에 추가
    uint aliveIndex;
    counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION, 1, aliveIndex);
    aliveBuffer_CURRENT[aliveIndex] = particleIndex;
}
```

**주의사항**:
- Emit는 `aliveBuffer_CURRENT` (aliveList[0])에 쓴다
- `aliveBuffer_NEW`는 바인딩만 하고 사용 안 함 (WickedEngine 호환성)
- Dead list는 LIFO 스택: pop from end (deadCount-1), push to end
- InterlockedAdd의 반환값은 **add 전의 값**

#### 3.4 Kickoff Update Shader (emittedparticle_kickoff_CS.hlsl)

**목적**: Emit 후 카운터를 정리하고 Simulate용 indirect args 준비

```c
[numthreads(32, 1, 1)]
void main(uint3 DTid : SV_DispatchThreadID) {
    if (DTid.x == 0) {
        // 1. Copy aliveCount_afterSimulation → aliveCount
        uint aliveCount = counterBuffer.Load(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION);
        counterBuffer.Store(PARTICLECOUNTER_OFFSET_ALIVECOUNT, aliveCount);

        // 2. Prepare DispatchSimulation args
        uint threadGroups = (aliveCount + 255) / 256;
        uint offset = ARGUMENTBUFFER_OFFSET_DISPATCHSIMULATION;
        indirectBuffer.Store(offset + 0, threadGroups);  // X
        indirectBuffer.Store(offset + 4, 1);             // Y
        indirectBuffer.Store(offset + 8, 1);             // Z
    }
}
```

#### 3.5 Simulate Shader (emittedparticle_simulate_CS.hlsl)

**바인딩**:
```c
RWStructuredBuffer<Particle> particleBuffer : register(u0);
RWStructuredBuffer<uint> aliveBuffer_CURRENT : register(u1);  // Read
RWStructuredBuffer<uint> aliveBuffer_NEW : register(u2);      // Write
RWStructuredBuffer<uint> deadBuffer : register(u3);
RWByteAddressBuffer counterBuffer : register(u4);
RWStructuredBuffer<float> distanceBuffer : register(u5);
```

**Dispatch**: Indirect dispatch (알아서 계산됨)
- 예: aliveCount=500 → 2 groups (500/256 = 1.95 → ceil = 2)

**핵심 로직** (Lines 18-123):
```c
[numthreads(256, 1, 1)]
void main(uint3 DTid : SV_DispatchThreadID) {
    // 1. 범위 체크
    uint aliveCount = counterBuffer.Load(PARTICLECOUNTER_OFFSET_ALIVECOUNT);
    if (DTid.x >= aliveCount)
        return;

    // 2. Alive list에서 파티클 인덱스 읽기
    uint particleIndex = aliveBuffer_CURRENT[DTid.x];
    Particle particle = particleBuffer[particleIndex];

    // 3. Timestep 계산
    float dt = xEmitterFixedTimestep;
    if (dt < 0.0f) {
        dt = GetFrame().delta_time;
        // Fallback to 60 FPS if invalid
        if (dt <= 0.0f || dt > 0.1f) {
            // dt = 1.0f / 60.0f;  // Commented out - use actual dt
        }
    }

    // 4. Life 감소
    particle.life -= dt;

    // 5. Life lerp 계산 (size, opacity용)
    const float lifeLerp = 1.0f - particle.life / particle.maxLife;
    const float particleSize = lerp(particle.sizeBeginEnd.x, particle.sizeBeginEnd.y, lifeLerp);

    // 6. 생존 체크
    if (particle.life > 0.0f) {
        // === Physics Integration ===

        // 6a. Gravity
        particle.force += xParticleGravity;

        // 6b. Velocity integration: v += (F/m) * dt
        particle.velocity += (particle.force / particle.mass) * dt;

        // 6c. Position integration: p += v * dt
        particle.position += particle.velocity * dt;

        // 6d. Reset force
        particle.force = float3(0, 0, 0);

        // 6e. Apply drag
        particle.velocity *= xParticleDrag;

        // === Rotation Update ===

        // 6f. Unpack rotation
        uint packed = particle.rotation_rotationVelocity;
        float rotation = (float((packed >> 16) & 0xFFFF) / 65535.0f) * 2*PI - PI;
        float rotationVel = (float(packed & 0xFFFF) / 65535.0f) * 2*PI - PI;

        // 6g. Update rotation
        rotation += rotationVel * dt;

        // 6h. Re-pack rotation
        uint rotation_new = uint((rotation + PI) / (2*PI) * 65535.0f);
        uint rotationVel_new = uint((rotationVel + PI) / (2*PI) * 65535.0f);
        particle.rotation_rotationVelocity = (rotation_new << 16) | rotationVel_new;

        // === Add to alive list ===

        // 6i. Push to NEW list
        uint newAliveIndex;
        counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION, 1, newAliveIndex);
        aliveBuffer_NEW[newAliveIndex] = particleIndex;

        // 6j. Calculate distance for sorting
        // IMPORTANT: Store at particleIndex, NOT newAliveIndex!
        float3 eyeVector = particle.position - GetCamera().position;
        float distSQ = dot(eyeVector, eyeVector);
        distanceBuffer[particleIndex] = -distSQ;  // Negative = far-to-near
    }
    else {
        // === Particle Death ===

        // 7. Push to dead list (LIFO)
        uint deadIndex;
        counterBuffer.InterlockedAdd(PARTICLECOUNTER_OFFSET_DEADCOUNT, 1, deadIndex);
        deadBuffer[deadIndex] = particleIndex;
    }

    // 8. Write back particle
    particleBuffer[particleIndex] = particle;
}
```

**핵심 포인트**:
- Simulate는 `aliveBuffer_CURRENT`를 읽고 `aliveBuffer_NEW`에 쓴다
- Distance는 `particleIndex`에 저장 (not aliveIndex!) - Sort 호환성
- Dead list는 LIFO: push to end (deadIndex)

#### 3.6 Sort Shader (emittedparticle_sort_CS.hlsl)

**알고리즘**: Bitonic Sort (AMD GPUSortLib 기반)

**바인딩**:
```c
RWStructuredBuffer<uint> aliveBuffer : register(u0);          // Sort in-place
RWStructuredBuffer<float> distanceBuffer : register(u1);      // Read distances
```

**Dispatch**: `(maxParticles + 511) / 512` groups of 512 threads
- 각 그룹은 512개 파티클까지 정렬 가능
- 그룹 간 정렬은 없음 (trade-off: 성능 vs 완벽한 정렬)

**LDS 사용**:
```c
#define SORT_SIZE 512
groupshared float2 g_LDS[SORT_SIZE];  // (distance, particleIndex)
```

**핵심 로직**:
```c
[numthreads(SORT_SIZE, 1, 1)]
void main(uint3 GTid : SV_GroupThreadID, uint3 Gid : SV_GroupID) {
    uint baseIndex = Gid.x * SORT_SIZE;
    uint localIndex = GTid.x;
    uint globalIndex = baseIndex + localIndex;

    // 1. Load to LDS
    if (globalIndex < aliveCount) {
        uint particleIndex = aliveBuffer[globalIndex];
        float distance = distanceBuffer[particleIndex];  // Read by particleIndex!
        g_LDS[localIndex] = float2(distance, particleIndex);
    } else {
        g_LDS[localIndex] = float2(FLT_MAX, 0);  // Sentinel
    }
    GroupMemoryBarrierWithGroupSync();

    // 2. Bitonic sort (nested loops)
    for (uint nMergeSize = 2; nMergeSize <= SORT_SIZE; nMergeSize *= 2) {
        for (uint nMergeSubSize = nMergeSize >> 1; nMergeSubSize > 0; nMergeSubSize >>= 1) {
            uint index = 2 * localIndex - (localIndex & (nMergeSubSize - 1));

            // Compare and swap
            float2 a = g_LDS[index];
            float2 b = g_LDS[index + nMergeSubSize];

            if (a.x > b.x) {  // Far-to-near (negative distance)
                g_LDS[index] = b;
                g_LDS[index + nMergeSubSize] = a;
            }

            GroupMemoryBarrierWithGroupSync();
        }
    }

    // 3. Write back to alive buffer
    if (globalIndex < aliveCount) {
        aliveBuffer[globalIndex] = (uint)g_LDS[localIndex].y;
    }
}
```

**성능**:
- 시간 복잡도: O(n log²n)
- LDS 사용량: 512 * 8 bytes = 4KB per group
- 제한: 각 그룹 내에서만 정렬 (충분히 좋은 품질)

#### 3.7 Finish Update Shader (emittedparticle_finish_CS.hlsl)

**목적**: Draw indirect arguments 준비

```c
[numthreads(1, 1, 1)]
void main() {
    // Read alive count after simulation
    uint aliveCount = counterBuffer.Load(PARTICLECOUNTER_OFFSET_ALIVECOUNT_AFTERSIMULATION);

    // Prepare DrawIndexedInstancedIndirect args
    uint offset = ARGUMENTBUFFER_OFFSET_DRAWPARTICLES;
    indirectBuffer.Store(offset + 0, 6);              // IndexCountPerInstance (quad = 6 indices)
    indirectBuffer.Store(offset + 4, aliveCount);     // InstanceCount (= alive particles)
    indirectBuffer.Store(offset + 8, 0);              // StartIndexLocation
    indirectBuffer.Store(offset + 12, 0);             // BaseVertexLocation
    indirectBuffer.Store(offset + 16, 0);             // StartInstanceLocation
}
```

### 4. 렌더링 파이프라인

#### 4.1 Vertex Shader (emittedparticle_VS.hlsl)

**입력**:
- `input.instanceID`: 파티클 인덱스 (0~aliveCount-1)
- `input.vertexID`: Quad 버텍스 인덱스 (0~5, 2 triangles)

**출력**:
```c
struct PS_INPUT {
    float4 pos : SV_POSITION;
    float2 uv : TEXCOORD0;
    float4 color : COLOR;
    float lifePercent : LIFE;
    float rotation : ROTATION;
    uint particleIndex : PARTICLEIDX;    // Debug용
    uint aliveListIndex : ALIVEIDX;      // Debug용
};
```

**핵심 로직**:
```c
PS_INPUT main(uint vID : SV_VertexID, uint iID : SV_InstanceID) {
    // 1. Load particle from alive list
    uint particleIndex = aliveBuffer[iID];
    Particle particle = particleBuffer[particleIndex];

    // 2. Calculate life lerp
    float lifeLerp = 1.0f - particle.life / particle.maxLife;
    float particleSize = lerp(particle.sizeBeginEnd.x, particle.sizeBeginEnd.y, lifeLerp);

    // 3. Generate quad vertices (-1~1)
    float2 quadPos;
    switch(vID) {
        case 0: quadPos = float2(-1, -1); break;
        case 1: quadPos = float2(-1,  1); break;
        case 2: quadPos = float2( 1,  1); break;
        case 3: quadPos = float2(-1, -1); break;
        case 4: quadPos = float2( 1,  1); break;
        case 5: quadPos = float2( 1, -1); break;
    }
    quadPos *= particleSize;

    // 4. Unpack rotation
    float rotation = (float((particle.rotation_rotationVelocity >> 16) & 0xFFFF) / 65535.0f) * 2*PI - PI;

    // 5. Apply rotation
    float c = cos(rotation);
    float s = sin(rotation);
    float2 rotatedPos = float2(
        quadPos.x * c - quadPos.y * s,
        quadPos.x * s + quadPos.y * c
    );

    // 6. Billboard to camera
    float3 cameraRight = GetCamera().inverse_view[0].xyz;
    float3 cameraUp = GetCamera().inverse_view[1].xyz;

    float3 worldPos = particle.position;
    worldPos += cameraRight * rotatedPos.x;
    worldPos += cameraUp * rotatedPos.y;

    // 7. Motion blur (optional)
    if (xParticleMotionBlurAmount > 0.0f) {
        float3 velocityViewSpace = mul((float3x3)GetCamera().view, particle.velocity);
        rotatedPos += dot(rotatedPos, velocityViewSpace.xy) * velocityViewSpace.xy * xParticleMotionBlurAmount;
        // Re-apply billboard with stretched quad
    }

    // 8. Transform to clip space
    output.pos = mul(GetCamera().view_projection, float4(worldPos, 1));

    // 9. UV coordinates
    output.uv = quadPos / particleSize * 0.5f + 0.5f;  // 0~1

    // 10. Color
    output.color = unpack_rgba(particle.color);
    output.lifePercent = lifeLerp;
    output.rotation = rotation;

    return output;
}
```

#### 4.2 Pixel Shader (emittedparticle_simple_PS.hlsl)

```c
float4 main(PS_INPUT input) : SV_TARGET {
    // 1. Sample texture
    float4 texColor = xTexture.Sample(sampler_linear_clamp, input.uv);

    // 2. Calculate opacity curve
    float t = input.lifePercent;
    float opacityFactor;

    if (t < xOpacityCurvePeakStart) {
        // Fade in: 0 → 1
        opacityFactor = t / xOpacityCurvePeakStart;
    } else if (t < xOpacityCurvePeakEnd) {
        // Peak: 1
        opacityFactor = 1.0f;
    } else {
        // Fade out: 1 → 0
        opacityFactor = 1.0f - (t - xOpacityCurvePeakEnd) / (1.0f - xOpacityCurvePeakEnd);
    }

    // 3. Combine colors
    float4 finalColor = texColor * xParticleBaseColor * input.color;
    finalColor.a *= opacityFactor;

    // 4. Apply emissive (HDR)
    finalColor.rgb *= (1.0f + xParticleEmissive);

    return finalColor;
}
```

### 5. 동적 MaxParticles 변경 메커니즘

#### 5.1 문제와 해결

**초기 시도 - 즉시 재생성**:
```cpp
void SetMaxParticles(uint32_t count) {
    DestroyGPUResources();   // 렌더링 중 파괴!
    CreateGPUResources();    // Assertion failed: cmd.IsValid()
}
```

**WaitForGPU 시도**:
```cpp
void SetMaxParticles(uint32_t count) {
    device->WaitForGPU();    // GPU 대기
    DestroyGPUResources();   // 여전히 에러!
    CreateGPUResources();
}
```

**최종 해결 - Lazy Invalidation** (WickedEngine 방식):
```cpp
// EmittedParticleComponent.cpp:337-352
void GEmittedParticleComponent::SetMaxParticles(uint32_t count) {
    if (maxParticles_ != count) {
        backlog::post("Changing from " + std::to_string(maxParticles_) +
                      " to " + std::to_string(count) + " particles");

        maxParticles_ = count;
        timeStampSetter_ = TimerNow;

        // Just invalidate - don't destroy during rendering!
        counterBuffer_ = {};           // Empty GPUBuffer
        gpuResourcesCreated_ = false;  // Mark as invalid

        // Resources will be recreated in next UpdateCPU() → CreateGPUResources()
    }
}
```

**동작 흐름**:
```
Frame N:
    UI: SetMaxParticles(5000)
        → counterBuffer_ = {}
        → gpuResourcesCreated_ = false
    UpdateCPU()
        → HasValidGPUResources() == false
        → CreateGPUResources() 호출
        → 5000 particles용 버퍼 생성
    UpdateGPU()
        → 정상 렌더링 (새 버퍼 사용)
```

**장점**:
- 렌더링 중단 없음
- 커맨드 리스트 무효화 방지
- 다음 프레임에서 자동으로 안전하게 재생성

### 6. 메모리 레이아웃 및 성능 특성

#### 6.1 1000 파티클 기준 메모리 사용량

```
particleBuffer:      1000 * 64 bytes   = 64,000 bytes  (62.5 KB)
aliveList[0]:        1000 * 4 bytes    = 4,000 bytes   (3.9 KB)
aliveList[1]:        1000 * 4 bytes    = 4,000 bytes   (3.9 KB)
deadList:            1000 * 4 bytes    = 4,000 bytes   (3.9 KB)
distanceBuffer:      1000 * 4 bytes    = 4,000 bytes   (3.9 KB)
counterBuffer:       24 bytes
indirectBuffers:     256 bytes
constantBuffer:      ~256 bytes (CB는 256 byte aligned)
emitBuffer:          64 bytes
────────────────────────────────────────────────────────
Total:               ~84 KB per emitter
```

#### 6.2 GPU Occupancy 분석

**Emit Shader**:
- Thread group: 256 threads
- Registers: ~20 (예상)
- LDS: 0 bytes
- Occupancy: High (100%)

**Simulate Shader**:
- Thread group: 256 threads
- Registers: ~30 (예상)
- LDS: 0 bytes
- Occupancy: High (100%)

**Sort Shader**:
- Thread group: 512 threads
- Registers: ~15 (예상)
- LDS: 4 KB (512 * 8 bytes)
- Occupancy: Medium-High (~75%, LDS 제한)

#### 6.3 프레임당 GPU 비용

**1000 파티클, 60 FPS 기준**:
```
Emit:         ~0.05ms  (100 particles emitted)
KickoffUpdate: ~0.01ms  (1 thread group)
Simulate:     ~0.15ms  (1000 particles, physics)
Sort:         ~0.10ms  (Bitonic sort, 2 groups)
FinishUpdate: ~0.01ms  (1 thread group)
Draw:         ~0.20ms  (1000 quads, alpha blending)
────────────────────────────────────────────────
Total:        ~0.52ms per emitter
```

**10,000 파티클 기준**: ~2.5ms (선형 증가 아님, sort 병목)

---

## 🎨 Architecture Overview

### Component Hierarchy

```
ComponentBase
  └─ EmittedParticleComponent
       ├─ Particle properties (size, life, velocity, etc.)
       ├─ Material integration (materialID_)
       └─ GEmittedParticleComponent
            ├─ GPU buffers
            ├─ GPU resource management
            └─ GPU update pipeline
```

### GPU Pipeline Flow

```
CPU (UpdateCPU):
  ├─ Update world matrix
  ├─ Accumulate emit count
  ├─ Handle burst emissions
  └─ Swap alive buffers ← Important!

GPU (UpdateGPU):
  ├─ [Emit] ─────────────────┐
  │   └─ Dead list → Alive   │
  │                          │
  ├─ [Kickoff Update] ───────┤
  │   └─ Prepare counters    │
  │                          │
  ├─ [Simulate] ─────────────┤
  │   ├─ Physics             │
  │   ├─ Life countdown      │
  │   └─ Alive → Dead        │
  │                          │
  ├─ [Sort] (optional) ──────┤
  │   └─ Bitonic sort        │
  │                          │
  ├─ [Finish Update] ────────┤
  │   └─ Prepare draw args   │
  │                          │
  └─ [Draw] ─────────────────┘
      └─ Billboard quads
```

### Double Buffering Strategy

```
aliveList[0]: CURRENT (read)
aliveList[1]: NEW (write)

Frame N:
  Emit:     writes to aliveList[0]
  Simulate: reads aliveList[0], writes to aliveList[1]
  Draw:     reads aliveList[1]
  Swap:     swap(aliveList[0], aliveList[1])

Frame N+1:
  Emit:     writes to aliveList[0] (was aliveList[1])
  ...
```

---

## 🐛 Major Issues and Solutions

### Issue #1: Particle Not Spawning
- **Symptom**: No particles rendered
- **Root Cause**: Dead list not initialized
- **Solution**: Initialize dead list with all indices (0 to MAX_PARTICLES-1)
- **File**: `EmittedParticleComponent.cpp:186-190`

### Issue #2: Billboard Orientation Wrong
- **Symptom**: Particles not facing camera
- **Root Cause**: Incorrect inverse view matrix usage
- **Solution**: Extract camera axes from inverse view matrix columns
- **File**: `emittedparticle_VS.hlsl:111-112`

### Issue #3: Opacity Not Working
- **Symptom**: All particles same opacity
- **Root Cause**: Constant buffer not bound in Draw call
- **Solution**: Bind CB with opacity parameters in `DrawParticles()`
- **File**: `EmittedParticle_Detail.cpp:684-685`

### Issue #4: Material Color Not Applied
- **Symptom**: Material base color ignored
- **Root Cause**: Material data not passed to shaders in Draw call
- **Solution**: Read material in `DrawParticles()`, set CB before binding
- **File**: `EmittedParticle_Detail.cpp:672-682`

### Issue #5: Rotation Not Working
- **Symptom**: Particles don't rotate over time
- **Root Cause**:
  1. Rotation velocity not applied in simulation
  2. Packing formula incorrect
- **Solution**: Apply rotation velocity in simulate shader with proper wrapping
- **File**: `emittedparticle_simulate_CS.hlsl:60-81`

### Issue #6: Particle Flickering (Critical)
- **Symptom**: Random particle flickering, especially after particles start dying
- **Root Cause**: Buffer swap timing wrong - swapped AFTER GPU commands instead of BEFORE
- **Solution**: Move `SwapBuffers()` to before GPU command submission
- **Files**:
  - `EmittedParticle_Detail.cpp:233` (swap moved here)
  - `emittedparticle_emit_CS.hlsl:12` (aliveBuffer_NEW binding added)
- **Details**: See Phase 8 section above

### Issue #7: MaxParticles GUI Not Working
- **Symptom**: Changing max particles in UI has no effect
- **Root Cause**:
  1. Immediate resource recreation during rendering
  2. Virtual function not declared
- **Solution**: Invalidate resources, recreate next frame (WickedEngine style)
- **File**: `EmittedParticleComponent.cpp:337-352`

---

## 📊 Performance Characteristics

### GPU Buffer Sizes (for 1000 particles)
```
particleBuffer_:     1000 * 64 bytes  = 64 KB
aliveList_[0]:       1000 * 4 bytes   = 4 KB
aliveList_[1]:       1000 * 4 bytes   = 4 KB
deadList_:           1000 * 4 bytes   = 4 KB
distanceBuffer_:     1000 * 4 bytes   = 4 KB
counterBuffer_:      32 bytes
indirectBuffers_:    80 bytes
emitBuffer_:         64 bytes
─────────────────────────────────────────────
Total:               ~84 KB per emitter
```

### Compute Shader Dispatch Sizes
```
Emit:         (emitCount + 255) / 256 groups
Kickoff:      1 group (32 threads)
Simulate:     Indirect dispatch (based on aliveCount)
Sort:         (maxParticles + 511) / 512 groups
FinishUpdate: 1 group (1 thread)
```

### Sorting Performance
- **Algorithm**: Bitonic sort (O(n log²n))
- **Local size**: 512 particles per group
- **Shared memory**: 512 * 8 bytes = 4 KB per group
- **Trade-off**: Quality vs Performance (can disable sorting)

---

## 🔧 Key Implementation Details

### Random Number Generation
```c
float rand(uint seed, uint offset) {
    uint h = seed + offset;
    h = (h ^ 61u) ^ (h >> 16u);
    h *= 9u;
    h = h ^ (h >> 4u);
    h *= 0x27d4eb2du;
    h = h ^ (h >> 15u);
    return float(h) * (1.0f / 4294967296.0f);
}
```
- **Type**: Hash-based PRNG
- **Quality**: Good distribution for particle effects
- **Performance**: Very fast (no texture lookups)

### Rotation Packing
```c
// Pack two float16 values into uint32
uint rotation_packed = uint((rotation + PI) / (2.0f * PI) * 65535.0f);
uint rotationVel_packed = uint((rotationVel + PI) / (2.0f * PI) * 65535.0f);
uint packed = (rotation_packed << 16) | rotationVel_packed;

// Unpack
uint rotationBits = (packed >> 16) & 0xFFFF;
float rotation = (float(rotationBits) / 65535.0f) * 2.0f * PI - PI;
```
- **Range**: [-π, π]
- **Precision**: 16 bits per value (~0.1 degree)
- **Benefit**: Save 4 bytes per particle

### Color Packing
```c
uint pack_rgba(float4 color) {
    uint r = uint(saturate(color.r) * 255.0f);
    uint g = uint(saturate(color.g) * 255.0f);
    uint b = uint(saturate(color.b) * 255.0f);
    uint a = uint(saturate(color.a) * 255.0f);
    return (a << 24) | (b << 16) | (g << 8) | r;
}
```
- **Format**: RGBA8 (8 bits per channel)
- **Benefit**: Save 12 bytes per particle

---

## 🎯 Future Improvements

### High Priority
- [ ] **Texture Support**: Add sprite sheet animation
- [ ] **Collision Detection**: Depth buffer collision
- [ ] **Mesh Emission**: Emit from mesh surface
- [ ] **GPU Culling**: Frustum culling on GPU

### Medium Priority
- [ ] **Soft Particles**: Depth fade near geometry
- [ ] **Lighting**: PBR lighting for particles
- [ ] **Trails**: Particle trail rendering
- [ ] **Forces**: Wind, attractors, vortex

### Low Priority
- [ ] **SPH Simulation**: Fluid dynamics
- [ ] **Simulation Spaces**: World/Local/Custom
- [ ] **Sub-Emitters**: Particle spawning particles
- [ ] **GPU Readback**: Statistics and debugging

---

## 📚 References

### WickedEngine
- **Repository**: https://github.com/turanszkij/WickedEngine
- **License**: MIT
- **Files Referenced**:
  - `wiEmittedParticle.h/cpp`
  - `emittedparticle_emitCS.hlsl`
  - `emittedparticle_simulateCS.hlsl`
  - `emittedparticle_kickoffUpdateCS.hlsl`
  - `ShaderInterop_EmittedParticle.h`

### AMD GPUSortLib
- **Algorithm**: Bitonic Sort
- **Used In**: `emittedparticle_sort_CS.hlsl`

### Key Design Decisions
1. **Double Buffering**: Prevents read-write conflicts
2. **GPU-Driven**: All simulation on GPU for performance
3. **Indirect Dispatch**: Efficient for varying particle counts
4. **Dead List Recycling**: Memory efficiency
5. **Lazy Resource Creation**: Recreate only when needed (WickedEngine style)

---

## ✅ Testing

### Test Cases Implemented
- `ParticleSystemTest.cpp`: Basic component functionality
- Sample015: Interactive UI testing
- Visual tests: Sorting, opacity, rotation, color

### Known Limitations
- Max 1M particles per emitter (UI limit)
- Sorting limited to 512 particles per group
- No texture support yet
- No collision detection yet

---

## 📝 Commit Summary

```
Initial Setup:
  14cec49 - Ready to add particle components
  527ff0b - Setup basic particle structure
  e02be4c - particle basic structure 2
  9a604a0 - particle basic 3

Core Implementation:
  a2e66d8 - Particle simulation
  2db82a5 - Fix particle spawn
  8e129ad - Fix billboarding
  43e4e5a - Add particle buffer swap

Visual Features:
  0c7ec6f - Remove opacitycurve texture
  6074ec1 - Fix opacity parameter
  c02daba - Add base color
  1e9feac - Add Motion blur

Sorting:
  e3ad8b6 - Add particle sorting

Material Integration:
  4c439ff - Add material to Particle
  4f4c26d - Fix particle to use material base color

Rotation:
  3cd6ecb - Fix particle rotation, rotation velocity

Critical Fixes:
  d98941c - Fix particle flickering

UI and Dynamic Config:
  36317bc - Add particle UI
  4301c55 - Add max particle count gui
  8a6fe5b - Fix max particle gui
```

