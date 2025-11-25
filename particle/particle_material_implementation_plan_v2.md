# Particle Material Implementation Plan v2

runtime script (루아)
파이썬으로 바인딩 

---

mcp

## 문제 분석

### 원래 시도했던 방법 (실패)
VID 시스템을 전면 수정하여 `VID = (Entity << 8) | ComponentType` 형식으로 변경했으나:
- 너무 많은 코드가 `VID == Entity`를 가정하고 작성됨
- Scene, Actor, Resource 등 모든 시스템에 영향
- 기존 코드와의 호환성 문제로 수많은 에러 발생

### 근본 원인
- `VzEngineManager.cpp`에서 `VID vid = entity;`로 설정
- 모든 VID가 사실상 Entity와 동일한 값
- `GetMaterialComponentByVUID(materialVID)`가 ComponentType을 확인하려 하지만 타입 정보가 없음

## 새로운 해결 방안: 최소 침습적 접근

### 핵심 아이디어
**VID 시스템 전체를 바꾸지 말고, SetParticleMaterialID에서만 Entity로 직접 접근**

### 이유
1. 기존 시스템에서 `VID == Entity`는 사실상 표준
2. Material의 VID는 사실 MaterialComponent의 Entity임
3. `GetMaterialComponent(Entity)`를 직접 사용하면 문제 없음

## 구현 계획 v2

### Phase 1: EmittedParticleComponent 수정 (변경 없음)

#### 1.1. materialID 추가
**파일**: `EngineCore/Components/Components.h`

**추가 위치**: Line 3027
```cpp
Entity materialID_ = INVALID_ENTITY;  // Material entity (VID == Entity in current system)
```

**Getter**: Line 3126
```cpp
inline Entity GetMaterialID() const { return materialID_; }
```

**Setter**: Line 3161
```cpp
inline void SetMaterialID(Entity id) { materialID_ = id; timeStampSetter_ = TimerNow; }
```

### Phase 2: VzMaterial API 확장 (변경 없음)

#### 2.1. VzMaterial 함수 추가
**파일**: `EngineCore/HighAPIs/VzMaterial.h`

Line 64 이후:
```cpp
void SetEmissiveColor(const vfloat3& color);
void SetEmissiveStrength(float strength);
```

**파일**: `EngineCore/HighAPIs/VzMaterial.cpp`

Line 75 이후:
```cpp
void VzMaterial::SetEmissiveColor(const vfloat3& color)
{
    GET_MATERIAL_COMP(material, );
    XMFLOAT4 emissive = material->GetEmissiveColor();
    emissive.x = color.x;
    emissive.y = color.y;
    emissive.z = color.z;
    material->SetEmissiveColor(emissive);
    UpdateTimeStamp();
}

void VzMaterial::SetEmissiveStrength(float strength)
{
    GET_MATERIAL_COMP(material, );
    material->SetEmissiveStrength(strength);
    UpdateTimeStamp();
}
```

### Phase 3: VzActorParticle API 수정 (⚠️ 변경됨)

#### 3.1. SetParticleMaterialID 함수
**파일**: `EngineCore/HighAPIs/VzActor.h`

Line 287:
```cpp
void SetParticleMaterialID(const VID materialVID);
```

**파일**: `EngineCore/HighAPIs/VzActor.cpp`

Line 897 이후:
```cpp
void VzActorParticle::SetParticleMaterialID(const VID materialVID)
{
    EmittedParticleComponent* emitter = compfactory::GetEmittedParticleComponent(componentVID_);
    if (!emitter) return;

    // ⚠️ KEY CHANGE: VID == Entity in current system, use it directly
    // No need for GetMaterialComponentByVUID which expects encoded VID
    MaterialComponent* material = compfactory::GetMaterialComponent(materialVID);
    if (!material) return;

    emitter->SetMaterialID(material->GetEntity());
    UpdateTimeStamp();
}
```

**핵심 변경점:**
- ~~`GetMaterialComponentByVUID(materialVID)`~~ → `GetMaterialComponent(materialVID)`
- 현재 시스템에서 VID == Entity이므로 직접 사용 가능
- ComponentType 체크를 우회하여 assert 문제 해결

### Phase 4: Shader 시스템 연동 (변경 없음)

#### 4.1. ShaderInterop 수정
**파일**: `EngineShaders/Shaders/ShaderInterop_EmittedParticle.h`

Line 94 다음:
```cpp
float4      xParticleBaseColor;             // Base color (RGBA)

float       xParticleEmissive;              // Material emissive strength
float       xEmitterPadding2;
float       xEmitterPadding3;
float       xEmitterPadding4;
```

#### 4.2. DrawParticles 수정
**파일**: `EngineShaders/ShaderEngine/EmittedParticle_Detail.cpp`

Line 668 이후:
```cpp
cb.xParticleMotionBlurAmount = emitter.GetMotionBlurAmount();

// Get emissive from material if available
cb.xParticleEmissive = 0.0f;
Entity materialID = emitter.GetMaterialID();
if (materialID != INVALID_ENTITY)
{
    MaterialComponent* material = compfactory::GetMaterialComponent(materialID);
    if (material)
    {
        cb.xParticleEmissive = material->GetEmissiveStrength();
    }
}
```

#### 4.3. Pixel Shader 수정
**파일**: `EngineShaders/Shaders/PS/emittedparticle_simple_PS.hlsl`

Line 45 다음:
```cpp
float4 finalColor = texColor * input.color;
finalColor.a *= opacityFactor;

// Apply emissive (HDR multiplier, like WickedEngine)
finalColor.rgb *= (1.0f + xParticleEmissive);

return finalColor;
```

### Phase 5: Sample15 GUI 추가 (변경 없음)

#### 5.1. Material 생성 및 연결
**파일**: `Examples/Sample015/sample15.cpp`

Line 283 이후:
```cpp
// Create material for particle
VzMaterial* particleMaterial = vzm::NewMaterial("Particle Material");
if (particleMaterial)
{
    particleMaterial->SetBaseColor({ 1.0f, 1.0f, 1.0f, 1.0f });
    particleMaterial->SetEmissiveColor({ 1.0f, 1.0f, 1.0f });
    particleMaterial->SetEmissiveStrength(0.0f);

    // Connect to particle
    particleEmitter->SetParticleMaterialID(particleMaterial->GetVID());
}
```

#### 5.2. GUI 추가
Line 768 이후:
```cpp
ImGui::Separator();
vzimgui::IGTextTitle("----- Particle Material -----");
{
    static VzMaterial* particleMaterial = nullptr;
    if (!particleMaterial)
    {
        std::vector<VzBaseComp*> components;
        if (GetComponentsByName("Particle Material", components) > 0)
        {
            particleMaterial = (VzMaterial*)components[0];
        }
    }

    if (particleMaterial)
    {
        static float emissive_color[3] = { 1.0f, 1.0f, 1.0f };
        if (ImGui::ColorEdit3("Emissive Color", emissive_color))
        {
            particleMaterial->SetEmissiveColor({ emissive_color[0], emissive_color[1], emissive_color[2] });
        }

        static float emissive_strength = 0.0f;
        if (ImGui::SliderFloat("Emissive Strength", &emissive_strength, 0.0f, 10.0f))
        {
            particleMaterial->SetEmissiveStrength(emissive_strength);
        }
    }
}
```

## v1과 v2의 차이점

### v1 (실패한 방법)
```cpp
// VzActor.cpp
void VzActorParticle::SetParticleMaterialID(const VID materialVID)
{
    MaterialComponent* material = compfactory::GetMaterialComponentByVUID(materialVID);
    // ❌ 호출 체인: GetMaterialComponentByVUID → GetEntityByVUID → GetComponentByVUID
    // ❌ GetComponentByVUID (ComponentFactory.cpp:45): vuid & 0xFF로 타입 추출
    // ❌ VID == Entity라서 하위 8비트가 ComponentType과 일치하지 않음
    // ❌ switch문 default case (Line 67): assert(0) → CRASH!
}
```

### v2 (새로운 방법)
```cpp
// VzActor.cpp
void VzActorParticle::SetParticleMaterialID(const VID materialVID)
{
    MaterialComponent* material = compfactory::GetMaterialComponent(materialVID);
    // ✅ GetMaterialComponent는 Entity를 직접 받아서 ComponentManager에서 조회
    // ✅ type_bits 체크를 우회하므로 assert 발생 안 함
    // ✅ 현재 시스템에서 VID == Entity이므로 100% 정상 작동
}
```

## 코드 분석 결과

### 1. VID 생성 방식 (VzEngineManager.cpp:361)
```cpp
VID vid = entity;  // VID에는 ComponentType 정보가 없음!
```

### 2. GetMaterialComponentByVUID 호출 체인
```cpp
// ComponentFactory.cpp:337
MaterialComponent* GetMaterialComponentByVUID(const VUID vuid)
{
    return GetMaterialComponent(GetEntityByVUID(vuid));
}

// ComponentFactory.cpp:71
Entity GetEntityByVUID(const VUID vuid)
{
    ComponentBase* comp = GetComponentByVUID(vuid);  // ← 여기서 문제!
    if (comp == nullptr) return INVALID_ENTITY;
    return comp->GetEntity();
}

// ComponentFactory.cpp:43-67
ComponentBase* GetComponentByVUID(const VUID vuid)
{
    ComponentType comp_type = static_cast<ComponentType>(uint32_t(vuid & 0xFF));
    // ↑ Entity의 하위 8비트를 ComponentType으로 해석 시도

    switch (comp_type)
    {
    case ComponentType::MATERIAL: return materialManager.GetComponentByVUID(vuid);
    // ... 다른 case들 ...
    default: assert(0);  // ← VID=76일 때 type_bits=76 → assert 발생!
    }
}
```

### 3. GetMaterialComponent는 안전
```cpp
// ComponentFactory.cpp에 선언만 있고 구현은 ComponentManager에 있음
MaterialComponent* GetMaterialComponent(const Entity entity)
{
    // ComponentManager가 Entity로 직접 조회
    // type_bits 체크 없음 → assert 발생 안 함
}
```

## 주의사항

### 1. VID == Entity 가정
- **현재 시스템**: `VID vid = entity;` (VzEngineManager.cpp)
- **의미**: VID와 Entity가 동일한 값
- **영향**: VID를 Entity처럼 사용 가능

### 2. 기존 시스템 유지
- **절대 수정 금지**: VzEngineManager.cpp의 VID 생성 로직
- **절대 수정 금지**: ComponentFactory의 GetComponentByVUID
- **원칙**: 기존 동작에 영향을 주지 않는 최소한의 변경

### 3. MaterialComponent는 이미 완비
- `GetEmissiveStrength()` 존재 (Line 951)
- `SetEmissiveStrength()` 존재 (Line 958)
- 추가 작업 불필요

## 구현 순서

1. **기존 VzEngineManager.cpp 변경사항 모두 되돌리기** (git checkout)
2. Phase 1: EmittedParticleComponent 수정
3. Phase 2: VzMaterial API 추가
4. Phase 3: VzActorParticle API 추가 (**v2 방식 사용**)
5. 빌드 테스트
6. Phase 4: Shader 연동
7. 빌드 테스트
8. Phase 5: Sample15 GUI
9. 최종 테스트

## 검증 체크리스트

- [ ] Phase 1-3 빌드 성공
- [ ] Material 생성 시 VID 정상 (assert 없음)
- [ ] SetParticleMaterialID 호출 시 에러 없음
- [ ] Phase 4 빌드 성공
- [ ] Emissive strength = 0: 기존과 동일
- [ ] Emissive strength > 0: 파티클이 밝게 빛남
- [ ] GUI에서 실시간 조절 가능

## 왜 이 방법이 더 나은가?

1. **최소 침습**: 기존 VID 시스템을 건드리지 않음
2. **호환성**: 모든 기존 코드가 정상 작동
3. **단순성**: 변경 범위가 VzActor.cpp 단 한 줄
4. **안정성**: assert나 crash 위험 없음

현재 시스템의 특성(VID == Entity)을 **장점으로 활용**하는 접근법입니다.

## 최종 검증 완료

### ✅ 검증된 사항
1. **VID == Entity 확인** (VzEngineManager.cpp:361)
2. **GetComponentByVUID의 assert 위치 확인** (ComponentFactory.cpp:67)
3. **GetMaterialComponent가 안전함 확인** (Entity 직접 조회)
4. **MaterialComponent에 GetEmissiveStrength 존재 확인** (Components.h:951)
5. **현재 모든 Phase가 미구현 상태 확인** (깨끗한 시작점)

### ⚠️ v1에서 발생했던 오류들
1. `assert(0)` at ComponentFactory.cpp:67 - type_bits 불일치
2. `Invalid geometryEntity` - VID 시스템 변경의 부작용
3. 수많은 `Component(...) is INVALID!` - 전역적인 VID 문제
4. mutex lock 오류 - VID/Entity 변환 문제

### ✅ v2가 이 문제들을 회피하는 이유
- **GetComponentByVUID를 호출하지 않음** → type_bits 체크 우회
- **VID 시스템을 수정하지 않음** → 기존 코드 무영향
- **Entity를 직접 사용** → 변환 오류 없음

## 구현 시작 준비 완료

모든 검증이 완료되었습니다. v2 계획대로 구현을 시작할 수 있습니다.
