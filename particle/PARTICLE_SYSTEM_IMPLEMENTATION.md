# VizMotive Engine - Particle System Implementation

## Overview
VizMotive Engine의 GPU 기반 파티클 시스템 구현 문서입니다. Compute Shader를 사용한 고성능 파티클 시뮬레이션을 제공합니다.

**Branch**: `Particle`
**Last Commit**: `7b4daa1 - Particle attribute set`
**Implementation Date**: 2025-11

---

## Architecture

### System Components

```
GPU Pipeline:
┌─────────────┐    ┌──────────────┐    ┌────────────┐    ┌──────────────┐
│   Kickoff   │ -> │     Emit     │ -> │  Simulate  │ -> │    Finish    │
│   (Setup)   │    │ (Spawn 새파티클)│   │  (물리연산)  │    │  (Cleanup)   │
└─────────────┘    └──────────────┘    └────────────┘    └──────────────┘
                                                                  │
                                                                  v
                                                          ┌──────────────┐
                                                          │   Render     │
                                                          │ (VS -> PS)   │
                                                          └──────────────┘
```

### File Structure

#### Core Components
- `EngineCore/Components/EmittedParticleComponent.cpp` - CPU 측 컴포넌트 관리
- `EngineCore/Components/GComponents.h` - GPU 리소스 관리
- `EngineShaders/ShaderEngine/EmittedParticle_Detail.cpp` - 파이프라인 통합

#### High-Level API
- `EngineCore/HighAPIs/VzActor.h` - VzActorParticle 클래스 정의
- `EngineCore/HighAPIs/VzActor.cpp` - 속성 설정 함수 구현

#### Shaders
- `ShaderInterop_EmittedParticle.h` - CPU/GPU 공유 데이터 구조
- `CS/emittedparticle_kickoffUpdate_CS.hlsl` - 초기화 및 인덱스 설정
- `CS/emittedparticle_emit_CS.hlsl` - 파티클 생성
- `CS/emittedparticle_simulate_CS.hlsl` - 물리 시뮬레이션
- `CS/emittedparticle_finishUpdate_CS.hlsl` - 렌더링 준비
- `VS/emittedparticle_VS.hlsl` - 정점 셰이더 (빌보딩)
- `PS/emittedparticle_simple_PS.hlsl` - 픽셀 셰이더 (텍스처 샘플링)

---

## Implementation History

### Commit Timeline

```
a2e66d8 - Particle simulation        (초기 시뮬레이션 로직)
0e6878a - Add sample15               (테스트 샘플 추가)
77cbe50 - Fix sample15               (샘플 수정)
40d0e09 - Store5 -> Store            (버퍼 쓰기 수정)
2db82a5 - Fix particle spawn         (생성 로직 수정)
8e129ad - Fix billboarding           (빌보드 수정)
af21beb - Update emittedparticle_VS  (VS 업데이트)
59d083d - Remove debug log           (디버그 로그 제거)
317d980 - Memory error fix           (메모리 오류 수정)
7b4daa1 - Particle attribute set     (속성 API 추가)
```

### Key Implementation Steps

1. **GPU Resource Setup**
   - Particle buffer (structured buffer)
   - Alive/Dead list buffers (double buffered)
   - Counter buffer (atomic operations)
   - Indirect argument buffers
   - Distance buffer (for sorting)

2. **Compute Shader Pipeline**
   - Kickoff: 간접 인수 준비, 버퍼 인덱스 설정
   - Emit: Dead list에서 인덱스 가져와 새 파티클 생성
   - Simulate: 물리 연산 (속도, 중력, 수명 등)
   - Finish: Alive list 업데이트, 렌더링 인수 준비

3. **Rendering**
   - Instanced drawing (DrawInstancedIndirect)
   - Billboard geometry (camera-facing quads)
   - Texture sampling with opacity curve

4. **High-Level API**
   - VzActorParticle 클래스
   - VzLight와 유사한 setter 패턴
   - 직관적인 속성 제어

---

## API Usage

### Basic Setup (Sample15)

```cpp
#include "HighAPIs/VzActor.h"

// 파티클 이미터 생성
VzActorParticle* particleEmitter = vzm::NewActorParticle("Particle Emitter");
particleEmitter->SetPosition({ 0.f, 1.f, 0.f });
particleEmitter->SetVisibleLayerMask(0xF, true);
scene->AppendChild(particleEmitter);

// 초기 버스트
particleEmitter->Burst(100);
```

### Property Configuration

```cpp
// 기본 속성
particleEmitter->SetParticleSize(1.5f);                      // 크기
particleEmitter->SetParticleScaleX(1.2f);                    // X축 스케일
particleEmitter->SetParticleScaleY(1.2f);                    // Y축 스케일

// 발사 설정
particleEmitter->SetParticleVelocity({ 0.f, 3.0f, 0.f });   // 초기 속도
particleEmitter->SetParticleRandomFactor(1.5f);              // 속도 랜덤성

// 물리 설정
particleEmitter->SetParticleGravity({ 0.f, -9.8f, 0.f });   // 중력
particleEmitter->SetParticleDrag(0.98f);                     // 공기 저항 (0~1)
particleEmitter->SetParticleMass(1.0f);                      // 질량

// 수명 설정
particleEmitter->SetParticleLife(4.0f);                      // 기본 수명 (초)
particleEmitter->SetParticleRandomLife(2.0f);                // 수명 랜덤 범위 (±초)

// 발생 설정
particleEmitter->SetParticleEmitCount(30.0f);                // 초당 발생 개수
particleEmitter->SetParticleMaxCount(3000);                  // 최대 파티클 수

// 시각 효과
particleEmitter->SetParticleRandomColor(0.3f);               // 색상 랜덤성 (0~1)
particleEmitter->SetParticleRotation(0.5f);                  // 회전 속도
particleEmitter->SetParticleOpacityCurve(0.1f, 0.9f);        // 페이드 (시작%, 끝%)
particleEmitter->SetParticleSorted(true);                    // 깊이 정렬 활성화
```

### Available Methods

#### Emission Control
```cpp
void Burst(int num);                           // 한번에 N개 방출
void Burst(int num, const vfloat3& position); // 특정 위치에서 방출
void Restart();                                // 시스템 재시작
```

#### Property Setters
```cpp
void SetParticleSize(float size);              // 기본 크기
void SetParticleScaleX(float scale);           // X축 스케일
void SetParticleScaleY(float scale);           // Y축 스케일
void SetParticleVelocity(const vfloat3& v);   // 초기 속도
void SetParticleRandomFactor(float factor);    // 속도 랜덤성
void SetParticleGravity(const vfloat3& g);    // 중력
void SetParticleDrag(float drag);              // 공기 저항
void SetParticleMass(float mass);              // 질량
void SetParticleLife(float life);              // 수명
void SetParticleRandomLife(float randomLife);  // 수명 랜덤
void SetParticleEmitCount(float count);        // 발생 속도
void SetParticleMaxCount(uint32_t maxCount);   // 최대 개수
void SetParticleRandomColor(float random);     // 색상 랜덤
void SetParticleRotation(float rotation);      // 회전
void SetParticleOpacityCurve(float start, float end); // 투명도 커브
void SetParticleSorted(bool sorted);           // 정렬 여부
```

---

## Current Status

### ✅ Working Features

1. **GPU Pipeline**
   - ✅ Compute shader 4단계 파이프라인 작동
   - ✅ Indirect drawing 정상 작동
   - ✅ Double buffered alive list
   - ✅ Dead list 재사용

2. **Rendering**
   - ✅ 빌보드 렌더링 (카메라 facing)
   - ✅ 텍스처 샘플링
   - ✅ 인스턴싱 렌더링

3. **Properties (Confirmed Working)**
   - ✅ **Size** - 파티클 크기 변경 확인됨
   - ✅ **ScaleX / ScaleY** - 비율 조정 확인됨
   - ✅ **RandomColor** - 색상 변화 확인됨
   - ✅ **MaxCount** - 최대 개수 제한 작동
   - ✅ **EmitCount** - 발생 속도 제어 작동

4. **High-Level API**
   - ✅ VzActorParticle 클래스
   - ✅ 16개 setter 메서드
   - ✅ VzLight 스타일 API

### ⚠️ Issues / Not Working

1. **Physics Properties (작동 안 함)**
   - ❌ **Life / RandomLife** - 수명 제어 미작동
   - ❌ **Velocity** - 초기 속도 미적용
   - ❌ **Gravity** - 중력 효과 없음
   - ❌ **Drag** - 공기 저항 미작동
   - ❌ **Mass** - 질량 효과 없음

2. **Visual Effects (작동 안 함)**
   - ❌ **OpacityCurve** - 페이드 인/아웃 미작동
   - ❌ **Rotation** - 회전 효과 없음

3. **Other Issues**
   - ⚠️ **Exit Crash** - 프로그램 종료 시 D3D12MemAlloc.cpp:6875 크래시 (재현 가능)
   - ⚠️ **Sorted** - 정렬 기능 미확인

### 🔍 Suspected Issues

#### 1. Constant Buffer Update
```cpp
// EmittedParticle_Detail.cpp - UpdateGPU() 함수
// 문제: CB 업데이트 타이밍이나 값 전달 이슈 가능성
```

#### 2. Shader Parameter Binding
```hlsl
// emittedparticle_simulate_CS.hlsl
// velocity, gravity, drag 등의 파라미터가 제대로 바인딩되지 않을 가능성
```

#### 3. Emit Shader Logic
```hlsl
// emittedparticle_emit_CS.hlsl
// 초기 속도와 수명 설정이 제대로 적용되지 않을 가능성
```

---

## GPU Resources

### Buffer Layout

```cpp
// Particle Buffer (Structured Buffer)
struct Particle {
    float3 position;
    float  life;           // 현재 수명
    float3 velocity;
    float  maxLife;        // 최대 수명
    float  size;
    float  rotation;
    float  mass;
    uint   color;          // RGBA packed
};

// Counter Buffer (Raw Buffer)
struct ParticleCounters {
    uint aliveCount;                    // 현재 살아있는 파티클 수
    uint deadCount;                     // 현재 죽은 파티클 수
    uint realEmitCount;                 // 실제 발생한 파티클 수
    uint aliveCount_afterSimulation;    // 시뮬레이션 후 살아있는 수
    uint culledCount;                   // 컬링된 파티클 수
    uint cellAllocator;                 // (예약)
};

// Alive List [2] - Double buffered
// Dead List
// Distance Buffer - For sorting
// Indirect Args Buffer - For DrawInstancedIndirect
```

---

## Shader Pipeline Details

### 1. Kickoff Compute Shader
```hlsl
// 목적: 간접 인수 준비, 버퍼 인덱스 설정
// 입력: counterBuffer
// 출력: indirectBuffers (dispatch/draw args)
```

### 2. Emit Compute Shader
```hlsl
// 목적: 새 파티클 생성
// 입력: deadList, counterBuffer, emitter properties
// 출력: particleBuffer (초기화), aliveList (추가)
// 주요 로직:
//   - deadList에서 사용 가능한 인덱스 가져오기
//   - 초기 위치, 속도, 수명 설정
//   - aliveList에 추가
```

### 3. Simulate Compute Shader
```hlsl
// 목적: 물리 시뮬레이션
// 입력: particleBuffer, aliveList
// 출력: particleBuffer (업데이트)
// 주요 로직:
//   - 속도에 중력 적용
//   - 속도에 drag 적용
//   - 위치 업데이트 (position += velocity * dt)
//   - 수명 감소 (life -= dt)
//   - 죽은 파티클 처리
```

### 4. Finish Compute Shader
```hlsl
// 목적: 렌더링 준비
// 입력: particleBuffer, aliveList
// 출력: aliveList (정리), deadList (추가)
// 주요 로직:
//   - 죽은 파티클을 deadList로 이동
//   - 살아있는 파티클만 aliveList에 유지
//   - 거리 계산 (정렬용)
```

### 5. Vertex Shader
```hlsl
// 목적: 빌보드 지오메트리 생성
// 입력: particleBuffer, aliveList, vertexID, instanceID
// 출력: 스크린 공간 위치, UV
// 주요 로직:
//   - 인스턴스 ID로 파티클 데이터 가져오기
//   - 카메라 방향 벡터 계산
//   - 쿼드 정점 생성 (4개 정점)
```

### 6. Pixel Shader
```hlsl
// 목적: 텍스처 샘플링 및 색상 출력
// 입력: UV, color
// 출력: final color
// 주요 로직:
//   - 텍스처 샘플링
//   - 파티클 색상 적용
//   - 투명도 커브 적용 (life 기반)
```

---

## Known Issues & TODO

### High Priority
- [ ] **물리 속성 미작동 문제 해결** (velocity, gravity, drag, mass)
- [ ] **수명 제어 미작동 문제 해결** (life, randomLife)
- [ ] **투명도 커브 미작동 문제 해결** (opacityCurve)
- [ ] **종료 시 크래시 수정** (D3D12MemAlloc.cpp:6875)

### Medium Priority
- [ ] 파티클 정렬 기능 확인 및 테스트
- [ ] 회전 효과 구현 확인
- [ ] Burst with position 구현 완성

### Low Priority
- [ ] 파티클 충돌 처리
- [ ] 파티클 컬링 최적화
- [ ] LOD 시스템

---

## Debug Tips

### 1. Shader Debugging
```cpp
// EmittedParticle_Detail.cpp - Initialize()
// 셰이더 컴파일 실패 시 backlog 확인
```

### 2. GPU Resource Validation
```cpp
// EmittedParticleComponent.cpp - CreateGPUResources()
// 각 버퍼 생성 성공 여부 확인
```

### 3. Property Update Verification
```cpp
// VzActor.cpp - SetParticle* 함수들
// EmittedParticleComponent::Set* 호출 확인
// UpdateTimeStamp() 호출 확인
```

### 4. Constant Buffer Check
```cpp
// EmittedParticle_Detail.cpp - UpdateGPU()
// CB 데이터 확인 (velocity, gravity, life 등)
```

---

## Performance Notes

- **Max Particles**: 기본 1000개, 설정 가능
- **Update Frequency**: 매 프레임
- **GPU Memory**: ~2MB per 1000 particles (depends on buffer sizes)
- **Draw Calls**: 1 draw call per emitter (indirect drawing)

---

## References

### External Resources
- [GPU Gems 3 - Chapter 23: Particle Simulation](https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-23-high-speed-particle-rendering-gpu)
- [DirectCompute Particle Simulation](https://developer.nvidia.com/gpugems/gpugems3/part-v-physics-simulation/chapter-29-real-time-rigid-body-simulation-gpus)

### Internal Documentation
- `EngineShaders/Shaders/ShaderInterop_EmittedParticle.h` - 데이터 구조 정의
- `Examples/Sample015/sample15.cpp` - 사용 예제

---

## Change Log

### 2025-11-12
- VzActorParticle API 추가 (16 setter methods)
- Sample15 파티클 속성 설정 예제 추가
- 디버그 로그 제거
- 메모리 오류 수정 (destructor에 WaitForGPU 추가)

### Previous Changes
- 초기 구현 완료
- 빌보딩 수정
- 파티클 생성 로직 수정
- VS 업데이트

---

**Last Updated**: 2025-11-12
**Author**: Claude Code
**Status**: 🟡 Partially Working (size, scale, randomColor working / physics not working)
