# Part 2: 그래픽스 파이프라인

Part 1에서 개별 개념들을 배웠다.
이제 이것들이 어떻게 **순서대로 연결**되어 화면에 그림이 나오는지 알아보자.

---

## 2.1 파이프라인 전체 흐름도

### 그래픽스 파이프라인이란?

입력(버텍스 데이터) → 출력(화면 픽셀)까지의 **정해진 처리 단계**.

```
┌─────────────────────────────────────────────────────────────┐
│                    Graphics Pipeline                         │
│                                                              │
│  ┌──────────────┐                                           │
│  │ Input Data   │  버텍스 버퍼, 인덱스 버퍼                   │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Input        │  버텍스를 삼각형으로 조립                    │
│  │ Assembler    │  (고정 기능)                               │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Vertex       │  정점 변환 (MVP), 조명 등                   │
│  │ Shader       │  ★ 프로그래밍 가능                         │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Hull Shader  │  테셀레이션 제어                           │
│  │ Tessellator  │  (선택적)                                  │
│  │ Domain Shader│                                           │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Geometry     │  프리미티브 생성/삭제                       │
│  │ Shader       │  (선택적) ★ 프로그래밍 가능                │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Stream       │  버텍스를 버퍼로 출력                       │
│  │ Output       │  (선택적)                                  │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Rasterizer   │  삼각형 → 픽셀(프래그먼트)                  │
│  │              │  (고정 기능)                               │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Pixel        │  각 픽셀의 색상 결정                        │
│  │ Shader       │  ★ 프로그래밍 가능                         │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Output       │  깊이 테스트, 블렌딩                        │
│  │ Merger       │  (고정 기능, 설정 가능)                     │
│  └──────┬───────┘                                           │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Render       │  최종 결과 저장                            │
│  │ Target       │                                           │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

### 고정 기능 vs 프로그래밍 가능

**고정 기능 (Fixed Function)**:
- 동작 방식이 정해져 있음
- 파라미터로 설정만 가능
- 예: Input Assembler, Rasterizer, Output Merger

**프로그래밍 가능 (Programmable)**:
- 셰이더 코드로 동작 정의
- 자유롭게 로직 작성 가능
- 예: Vertex Shader, Pixel Shader, Compute Shader

### 왜 "파이프라인"인가?

공장 조립 라인처럼 **동시에 여러 단계가 진행**된다.

```
시간 →
삼각형1: [IA][VS][RS][PS][OM]
삼각형2:    [IA][VS][RS][PS][OM]
삼각형3:       [IA][VS][RS][PS][OM]

IA=Input Assembler, VS=Vertex Shader,
RS=Rasterizer, PS=Pixel Shader, OM=Output Merger
```

삼각형1이 Pixel Shader 처리 중일 때, 삼각형3은 이미 Input Assembly 중.
→ GPU 활용률 극대화

---

## 2.2 입력 조립 (Input Assembler)

### 역할

버텍스 버퍼와 인덱스 버퍼에서 데이터를 읽어 **프리미티브(삼각형)를 조립**한다.

```
입력:
- Vertex Buffer: [V0, V1, V2, V3, V4, V5, ...]
- Index Buffer: [0, 1, 2, 2, 1, 3, ...]
- Primitive Topology: Triangle List

출력:
- 삼각형 0: (V0, V1, V2)
- 삼각형 1: (V2, V1, V3)
- ...
```

### Primitive Topology

버텍스를 어떻게 연결할지 정의.

```
Point List:       Line List:        Line Strip:
  ●  ●  ●           ●───●              ●───●
  ●  ●  ●           ●───●                  │
                                       ●───●

Triangle List:    Triangle Strip:   Triangle Fan:
    ●                 ●───●             ●
   ╱ ╲               ╱ ╲ ╱ ╲           ╱│╲
  ●───●             ●───●───●         ● │ ●
    ●                                  ╲│╱
   ╱ ╲                                  ●
  ●───●
```

```cpp
// DX12에서 토폴로지 설정
commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
```

**Triangle List**: 가장 일반적. 삼각형마다 3개 인덱스.
**Triangle Strip**: 연속된 삼각형. 메모리 효율적이지만 제약 있음.

### Input Layout

버텍스 데이터의 구조를 GPU에 알려준다.

```cpp
struct Vertex {
    float position[3];  // 12 bytes, offset 0
    float normal[3];    // 12 bytes, offset 12
    float texCoord[2];  // 8 bytes, offset 24
};
// 총 32 bytes per vertex
```

```cpp
// DX12 Input Layout 정의
D3D12_INPUT_ELEMENT_DESC inputLayout[] = {
    {
        "POSITION",                          // SemanticName
        0,                                   // SemanticIndex
        DXGI_FORMAT_R32G32B32_FLOAT,        // Format
        0,                                   // InputSlot
        0,                                   // AlignedByteOffset
        D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,
        0                                    // InstanceDataStepRate
    },
    {
        "NORMAL",
        0,
        DXGI_FORMAT_R32G32B32_FLOAT,
        0,
        12,  // position 다음 (12 bytes)
        D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,
        0
    },
    {
        "TEXCOORD",
        0,
        DXGI_FORMAT_R32G32_FLOAT,
        0,
        24,  // position + normal 다음 (24 bytes)
        D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,
        0
    },
};
```

### 인스턴싱 (Instancing)

같은 메시를 여러 번 그릴 때, 버텍스 데이터를 복제하지 않고 **인스턴스 데이터**만 다르게.

```cpp
// 인스턴스별 데이터
struct InstanceData {
    float4x4 worldMatrix;
    float4 color;
};

// Input Layout에 인스턴스 데이터 추가
{
    "INSTANCE_WORLD",
    0,
    DXGI_FORMAT_R32G32B32A32_FLOAT,  // 행렬 첫 행
    1,                                // InputSlot 1 (다른 버퍼)
    0,
    D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA,  // ← 핵심!
    1                                              // 인스턴스마다
},
// ... (행렬 나머지 3행도 추가)
```

```cpp
// 드로우 콜
commandList->DrawIndexedInstanced(
    indexCount,      // 인덱스 개수
    instanceCount,   // 인스턴스 개수 (1000개 나무 등)
    0, 0, 0
);
```

**효과**: 나무 1000그루를 1번의 드로우 콜로 그릴 수 있음.

---

## 2.3 버텍스 셰이더 (Vertex Shader)

### 역할

각 버텍스의 **위치를 변환**하고 **데이터를 가공**한다.

```
입력: 로컬 공간의 버텍스 데이터
출력: 클립 공간의 버텍스 + 추가 데이터

[필수] 클립 공간 위치 (SV_POSITION) 출력해야 함
```

### 기본 버텍스 셰이더

```hlsl
cbuffer PerFrame : register(b0) {
    float4x4 viewProjection;
    float3 cameraPosition;
    float time;
};

cbuffer PerObject : register(b1) {
    float4x4 world;
    float4x4 worldInverseTranspose;  // 법선 변환용
};

struct VSInput {
    float3 position : POSITION;
    float3 normal : NORMAL;
    float2 texCoord : TEXCOORD0;
};

struct VSOutput {
    float4 clipPosition : SV_POSITION;  // 필수!
    float3 worldPosition : TEXCOORD0;
    float3 worldNormal : TEXCOORD1;
    float2 texCoord : TEXCOORD2;
};

VSOutput main(VSInput input) {
    VSOutput output;

    // 월드 변환
    float4 worldPos = mul(float4(input.position, 1.0), world);
    output.worldPosition = worldPos.xyz;

    // 클립 변환 (필수!)
    output.clipPosition = mul(worldPos, viewProjection);

    // 법선 변환 (비균등 스케일 처리)
    output.worldNormal = mul(input.normal, (float3x3)worldInverseTranspose);

    // UV 전달
    output.texCoord = input.texCoord;

    return output;
}
```

### 법선 변환이 특별한 이유

물체에 비균등 스케일(예: X만 2배)이 적용되면 법선도 같이 변환하면 **틀린 방향**이 됨.

```
원본:                 X 2배 스케일:
    ↑N                      ↗ N (잘못됨)
────●────              ────────●────────
  표면                      표면

올바른 법선:
                            ↑ N (맞음)
                       ────────●────────
```

해결: **World Inverse Transpose** 행렬 사용

```cpp
Matrix4 worldInverseTranspose = transpose(inverse(worldMatrix));
```

### 스키닝 (Skeletal Animation)

캐릭터 애니메이션에서 뼈대에 따라 버텍스 위치 변형.

```hlsl
// 본(뼈) 행렬들
cbuffer BoneMatrices : register(b2) {
    float4x4 bones[256];  // 최대 256개 본
};

struct SkinnedVSInput {
    float3 position : POSITION;
    float3 normal : NORMAL;
    float2 texCoord : TEXCOORD0;
    uint4 boneIndices : BLENDINDICES;   // 영향 받는 본 인덱스 (최대 4개)
    float4 boneWeights : BLENDWEIGHT;   // 각 본의 가중치
};

VSOutput main(SkinnedVSInput input) {
    // 4개 본의 영향을 가중 평균
    float4x4 skinMatrix =
        bones[input.boneIndices.x] * input.boneWeights.x +
        bones[input.boneIndices.y] * input.boneWeights.y +
        bones[input.boneIndices.z] * input.boneWeights.z +
        bones[input.boneIndices.w] * input.boneWeights.w;

    // 스키닝된 위치
    float4 skinnedPos = mul(float4(input.position, 1.0), skinMatrix);

    // 이후 일반 변환...
    output.clipPosition = mul(skinnedPos, viewProjection);
    // ...
}
```

### 버텍스 셰이더에서 할 수 있는 것들

```hlsl
// 1. 버텍스 애니메이션 (풀, 깃발 흔들림)
float wave = sin(time + input.position.x) * 0.1;
position.y += wave;

// 2. 빌보드 (항상 카메라를 향함)
float3 right = float3(view[0][0], view[1][0], view[2][0]);
float3 up = float3(view[0][1], view[1][1], view[2][1]);
position = center + right * input.position.x + up * input.position.y;

// 3. 터레인 LOD (거리에 따른 디테일)
float dist = length(cameraPosition - worldPos);
float lodFactor = saturate(dist / maxDistance);
position.y = lerp(highDetailHeight, lowDetailHeight, lodFactor);

// 4. 모프 타겟 (표정 애니메이션)
position = lerp(basePosition, targetPosition, morphWeight);
```

---

## 2.4 래스터화 (Rasterization)

### 역할

3D 삼각형을 2D 픽셀(프래그먼트)로 변환한다.

```
입력: 클립 공간의 삼각형 (3개 버텍스)
출력: 해당 삼각형이 덮는 모든 픽셀(프래그먼트)

삼각형:              래스터화 후:
    ●               ■ ■
   ╱ ╲              ■ ■ ■ ■
  ╱   ╲      →      ■ ■ ■ ■ ■ ■
 ╱     ╲            ■ ■ ■ ■ ■ ■ ■ ■
●───────●           ■ ■ ■ ■ ■ ■ ■ ■ ■ ■
```

### 래스터화 과정

```
1. 클리핑 (Clipping)
   - 화면 밖으로 나간 부분 잘라냄
   - Near/Far 평면 밖도 잘라냄

2. 원근 나눗셈 (Perspective Division)
   - (x, y, z, w) → (x/w, y/w, z/w)
   - NDC 좌표로 변환

3. 뷰포트 변환 (Viewport Transform)
   - NDC (-1~1) → 화면 좌표 (0~1920, 0~1080)

4. 삼각형 셋업 (Triangle Setup)
   - 삼각형 경계 계산
   - 보간 준비

5. 삼각형 순회 (Triangle Traversal)
   - 삼각형 내부의 모든 픽셀 찾기
   - 각 픽셀에 대해 프래그먼트 생성
```

### 보간 (Interpolation)

버텍스 셰이더 출력을 픽셀 셰이더 입력으로 보간.

```
버텍스 A (빨강, UV 0,0)
      ●
     ╱●╲        ← P 지점의 픽셀
    ╱   ╲         P = A*0.3 + B*0.5 + C*0.2 (무게중심 좌표)
   ●─────●
   B       C
(녹색,1,0) (파랑,0,1)
```

```hlsl
// 보간 모드 지정 (HLSL)
struct PSInput {
    float4 position : SV_POSITION;

    // 기본: 원근 보정 보간
    float3 normal : NORMAL;

    // 원근 보정 없이 (UI 등)
    noperspective float2 screenUV : TEXCOORD0;

    // 보간 없이 첫 버텍스 값 사용
    nointerpolation uint materialID : TEXCOORD1;

    // 중심 샘플링 (MSAA)
    centroid float2 texCoord : TEXCOORD2;
};
```

### Face Culling (면 제거)

뒷면(Back Face)은 안 보이니까 처리하지 않음.

```
카메라에서 봤을 때:

시계 방향 (CW):      반시계 방향 (CCW):
    0                    0
   ╱ ╲  ← 앞면          ╱ ╲  ← 뒷면
  2───1                 1───2

CullMode = Back이고 FrontFace = CCW면:
→ 시계 방향(뒷면) 삼각형은 그리지 않음
```

```cpp
// DX12 래스터라이저 설정
D3D12_RASTERIZER_DESC rasterDesc = {};
rasterDesc.FillMode = D3D12_FILL_MODE_SOLID;
rasterDesc.CullMode = D3D12_CULL_MODE_BACK;
rasterDesc.FrontCounterClockwise = TRUE;  // 반시계가 앞면
rasterDesc.DepthClipEnable = TRUE;
```

### 와이어프레임 모드

```cpp
rasterDesc.FillMode = D3D12_FILL_MODE_WIREFRAME;
// 삼각형의 외곽선만 그림 (디버깅용)
```

### 깊이 바이어스 (Depth Bias)

그림자 매핑에서 Shadow Acne 방지.

```
문제 상황:
표면 ────────────── 깊이 0.500
그림자맵 샘플 ───── 깊이 0.501 (부동소수점 오차)
→ 자기 자신의 그림자가 생김 (Shadow Acne)

해결:
깊이 바이어스로 약간 밀어냄
표면 ────────────── 깊이 0.500 + bias = 0.502
그림자맵 샘플 ───── 깊이 0.501
→ 자기 그림자 안 생김
```

```cpp
rasterDesc.DepthBias = 10000;
rasterDesc.DepthBiasClamp = 0.0f;
rasterDesc.SlopeScaledDepthBias = 1.0f;
```

---

## 2.5 픽셀 셰이더 (Pixel Shader)

### 역할

각 픽셀(프래그먼트)의 **최종 색상을 결정**한다.

```
입력: 보간된 버텍스 데이터, 텍스처, 상수 등
출력: RGBA 색상 (하나 이상의 렌더 타겟)
```

### 기본 픽셀 셰이더

```hlsl
// 텍스처와 샘플러
Texture2D albedoTexture : register(t0);
Texture2D normalTexture : register(t1);
SamplerState linearSampler : register(s0);

// 조명 정보
cbuffer Lighting : register(b0) {
    float3 lightDirection;
    float3 lightColor;
    float3 ambientColor;
    float3 cameraPosition;
};

struct PSInput {
    float4 position : SV_POSITION;
    float3 worldPosition : TEXCOORD0;
    float3 worldNormal : TEXCOORD1;
    float2 texCoord : TEXCOORD2;
};

float4 main(PSInput input) : SV_TARGET {
    // 텍스처 샘플링
    float4 albedo = albedoTexture.Sample(linearSampler, input.texCoord);

    // 법선 정규화
    float3 N = normalize(input.worldNormal);

    // 조명 방향 (표면에서 빛으로)
    float3 L = normalize(-lightDirection);

    // 시선 방향 (표면에서 카메라로)
    float3 V = normalize(cameraPosition - input.worldPosition);

    // 반사 방향
    float3 R = reflect(-L, N);

    // Diffuse (Lambert)
    float NdotL = max(0, dot(N, L));
    float3 diffuse = albedo.rgb * lightColor * NdotL;

    // Specular (Blinn-Phong)
    float3 H = normalize(L + V);  // 하프 벡터
    float NdotH = max(0, dot(N, H));
    float3 specular = lightColor * pow(NdotH, 64.0);  // 64 = shininess

    // Ambient
    float3 ambient = albedo.rgb * ambientColor;

    // 최종 색상
    float3 finalColor = ambient + diffuse + specular;

    return float4(finalColor, albedo.a);
}
```

### 노말 매핑 (Normal Mapping)

텍스처로 저장된 법선으로 디테일한 표면 표현.

```hlsl
// 탄젠트 공간 → 월드 공간 변환
float3 T = normalize(input.worldTangent);
float3 N = normalize(input.worldNormal);
float3 B = cross(N, T) * input.tangentSign;  // Bitangent
float3x3 TBN = float3x3(T, B, N);

// 노말맵 샘플링 (0~1 범위)
float3 normalMap = normalTexture.Sample(linearSampler, input.texCoord).rgb;

// -1~1 범위로 변환
normalMap = normalMap * 2.0 - 1.0;

// 월드 공간 법선
float3 worldNormal = normalize(mul(normalMap, TBN));
```

### PBR (Physically Based Rendering)

현실적인 재질 표현.

```hlsl
// PBR 텍스처들
Texture2D albedoMap : register(t0);
Texture2D normalMap : register(t1);
Texture2D metallicRoughnessMap : register(t2);  // R=?, G=Roughness, B=Metallic
Texture2D aoMap : register(t3);                 // Ambient Occlusion

float4 main(PSInput input) : SV_TARGET {
    // 재질 파라미터
    float4 albedo = albedoMap.Sample(sampler, uv);
    float metallic = metallicRoughnessMap.Sample(sampler, uv).b;
    float roughness = metallicRoughnessMap.Sample(sampler, uv).g;
    float ao = aoMap.Sample(sampler, uv).r;

    // F0 (수직 입사 반사율)
    float3 F0 = lerp(float3(0.04, 0.04, 0.04), albedo.rgb, metallic);

    // Cook-Torrance BRDF
    // ... (복잡한 PBR 수식)

    return float4(finalColor, 1.0);
}
```

### 픽셀 셰이더에서 할 수 있는 것들

```hlsl
// 1. 알파 테스트 (잎사귀, 철조망)
if (albedo.a < 0.5)
    discard;  // 이 픽셀 버림

// 2. 디더링 (부드러운 알파)
float dither = frac(sin(dot(input.position.xy, float2(12.9898, 78.233))) * 43758.5453);
if (albedo.a < dither)
    discard;

// 3. 깊이 출력 수정
output.depth = customDepth;  // SV_Depth semantic

// 4. 다중 출력 (MRT)
output.color0 = albedo;
output.color1 = float4(normal * 0.5 + 0.5, 1);
output.color2 = float4(metallic, roughness, ao, 1);
```

### Early-Z vs Late-Z

```
Early-Z (조기 깊이 테스트):
1. 래스터화
2. 깊이 테스트 ← 여기서 먼저!
3. 통과한 픽셀만 Pixel Shader 실행
장점: 가려진 픽셀의 셰이더 실행 안 함

Late-Z (지연 깊이 테스트):
1. 래스터화
2. Pixel Shader 실행 (모든 픽셀)
3. 깊이 테스트
단점: 버려질 픽셀도 셰이더 실행

discard 사용하면 Early-Z 못 씀 → 성능 저하
```

---

## 2.6 출력 병합 (Output Merger)

### 역할

픽셀 셰이더 출력을 렌더 타겟에 **병합**한다.

```
픽셀 셰이더 출력
        │
        ▼
┌───────────────┐
│ Depth Test    │  깊이 테스트
└───────┬───────┘
        ▼
┌───────────────┐
│ Stencil Test  │  스텐실 테스트
└───────┬───────┘
        ▼
┌───────────────┐
│ Blending      │  블렌딩
└───────┬───────┘
        ▼
   Render Target
```

### 깊이 테스트 (Depth Test)

```cpp
D3D12_DEPTH_STENCIL_DESC dsDesc = {};

// 깊이 테스트 활성화
dsDesc.DepthEnable = TRUE;

// 깊이 쓰기 (불투명 물체는 TRUE, 투명 물체는 보통 FALSE)
dsDesc.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ALL;

// 비교 함수
dsDesc.DepthFunc = D3D12_COMPARISON_FUNC_LESS;
// LESS: 새 깊이 < 기존 깊이면 통과 (일반적)
// LESS_EQUAL: 새 깊이 <= 기존 깊이면 통과
// GREATER: Reversed-Z 사용 시
// EQUAL: 같은 깊이만 (데칼, 아웃라인)
```

### 스텐실 테스트 (Stencil Test)

```cpp
dsDesc.StencilEnable = TRUE;
dsDesc.StencilReadMask = 0xFF;
dsDesc.StencilWriteMask = 0xFF;

// 앞면 설정
dsDesc.FrontFace.StencilFunc = D3D12_COMPARISON_FUNC_EQUAL;
dsDesc.FrontFace.StencilPassOp = D3D12_STENCIL_OP_KEEP;
dsDesc.FrontFace.StencilFailOp = D3D12_STENCIL_OP_KEEP;
dsDesc.FrontFace.StencilDepthFailOp = D3D12_STENCIL_OP_KEEP;

// 뒷면도 동일하게 또는 다르게 설정 가능
```

스텐실 활용 예시:
```
1. 포탈/거울
   - 포탈 영역에 스텐실 1 쓰기
   - 스텐실 1인 곳에만 다른 월드 렌더링

2. 아웃라인
   - 물체 그리면서 스텐실 1 쓰기
   - 물체를 약간 크게 그리되 스텐실 1이 아닌 곳만

3. 그림자 볼륨
   - 그림자 앞면에서 스텐실++
   - 그림자 뒷면에서 스텐실--
   - 스텐실 > 0인 곳이 그림자
```

### 블렌딩 (Blending)

새 색상과 기존 색상을 섞는다.

```cpp
D3D12_RENDER_TARGET_BLEND_DESC blendDesc = {};

// 블렌딩 활성화
blendDesc.BlendEnable = TRUE;

// 색상 블렌딩
blendDesc.SrcBlend = D3D12_BLEND_SRC_ALPHA;
blendDesc.DestBlend = D3D12_BLEND_INV_SRC_ALPHA;
blendDesc.BlendOp = D3D12_BLEND_OP_ADD;
// 결과 = Src * SrcAlpha + Dest * (1 - SrcAlpha)

// 알파 블렌딩
blendDesc.SrcBlendAlpha = D3D12_BLEND_ONE;
blendDesc.DestBlendAlpha = D3D12_BLEND_ZERO;
blendDesc.BlendOpAlpha = D3D12_BLEND_OP_ADD;

// 쓰기 마스크
blendDesc.RenderTargetWriteMask = D3D12_COLOR_WRITE_ENABLE_ALL;
```

블렌딩 공식:
```
최종색상 = (SrcColor × SrcBlend) BlendOp (DestColor × DestBlend)

일반 알파 블렌딩 (반투명):
최종 = Src × α + Dest × (1-α)

Additive (불꽃, 빛):
최종 = Src + Dest

Multiply (그림자):
최종 = Src × Dest
```

### 블렌딩과 그리기 순서

**문제**: 반투명 물체는 그리는 순서가 중요!

```
잘못된 순서:
1. 앞 유리 그림 (α=0.5)
2. 뒤 유리 그림 (α=0.5)
→ 뒤 유리가 깊이 테스트 실패로 안 그려짐

올바른 순서:
1. 뒤 유리 그림
2. 앞 유리 그림
→ 제대로 섞임
```

해결책:
```cpp
// 1. 불투명 물체 먼저 (깊이 쓰기 ON)
renderOpaqueObjects();

// 2. 반투명 물체를 뒤→앞 순서로 (깊이 쓰기 OFF)
sortTransparentByDistance();  // 카메라에서 먼 순
for (auto& obj : transparentObjects) {
    renderTransparent(obj);
}
```

### 독립 블렌딩 (Independent Blending)

MRT(Multiple Render Targets)에서 각 타겟마다 다른 블렌딩.

```cpp
D3D12_BLEND_DESC blendDesc = {};
blendDesc.IndependentBlendEnable = TRUE;

// RT0: 일반 알파 블렌딩
blendDesc.RenderTarget[0].BlendEnable = TRUE;
// ...

// RT1: 블렌딩 없음 (법선맵)
blendDesc.RenderTarget[1].BlendEnable = FALSE;

// RT2: Additive (발광)
blendDesc.RenderTarget[2].BlendEnable = TRUE;
blendDesc.RenderTarget[2].SrcBlend = D3D12_BLEND_ONE;
blendDesc.RenderTarget[2].DestBlend = D3D12_BLEND_ONE;
```

---

## 2.7 컴퓨트 셰이더 (Compute Shader)

### 그래픽스 파이프라인과의 차이

```
Graphics Pipeline:          Compute Pipeline:
버텍스 → 래스터 → 픽셀      그냥 GPU에서 계산
삼각형 처리에 특화           범용 계산
입출력 형식 정해짐           자유로운 입출력
```

### 기본 구조

```hlsl
// 스레드 그룹 크기 정의
// 256개 스레드가 한 그룹
[numthreads(256, 1, 1)]
void CSMain(
    uint3 groupID : SV_GroupID,           // 그룹 ID
    uint3 threadID : SV_DispatchThreadID, // 전체 스레드 ID
    uint3 localID : SV_GroupThreadID,     // 그룹 내 스레드 ID
    uint groupIndex : SV_GroupIndex       // 그룹 내 1D 인덱스
) {
    // 처리할 요소의 인덱스
    uint index = threadID.x;

    // 계산 수행
    outputBuffer[index] = inputBuffer[index] * 2;
}
```

### Dispatch 호출

```cpp
// CPU에서 컴퓨트 셰이더 실행
commandList->Dispatch(
    numGroupsX,  // X 방향 그룹 수
    numGroupsY,  // Y 방향 그룹 수
    numGroupsZ   // Z 방향 그룹 수
);

// 예: 1024개 요소 처리, 그룹당 256 스레드
// Dispatch(1024/256, 1, 1) = Dispatch(4, 1, 1)
```

### 2D 예제: 이미지 처리

```hlsl
Texture2D<float4> inputImage : register(t0);
RWTexture2D<float4> outputImage : register(u0);  // RW = Read-Write

[numthreads(8, 8, 1)]  // 8×8 = 64 스레드 per 그룹
void BlurCS(uint3 threadID : SV_DispatchThreadID) {
    // 현재 픽셀 좌표
    int2 pixel = threadID.xy;

    // 3x3 박스 블러
    float4 sum = float4(0, 0, 0, 0);
    for (int y = -1; y <= 1; y++) {
        for (int x = -1; x <= 1; x++) {
            sum += inputImage[pixel + int2(x, y)];
        }
    }

    outputImage[pixel] = sum / 9.0;
}
```

```cpp
// 1920×1080 이미지 처리
// 그룹 크기 8×8이므로:
commandList->Dispatch(1920/8, 1080/8, 1);
// = Dispatch(240, 135, 1)
```

### 그룹 공유 메모리 (Group Shared Memory)

같은 그룹 내 스레드들이 공유하는 빠른 메모리.

```hlsl
// 그룹 공유 메모리 선언 (최대 32KB)
groupshared float sharedData[256];

[numthreads(256, 1, 1)]
void ReductionCS(uint3 threadID : SV_DispatchThreadID,
                 uint localIndex : SV_GroupIndex) {

    // 1. 각 스레드가 데이터를 공유 메모리로 로드
    sharedData[localIndex] = inputBuffer[threadID.x];

    // 2. 모든 스레드가 로드할 때까지 대기
    GroupMemoryBarrierWithGroupSync();

    // 3. Reduction (합계 계산)
    for (uint stride = 128; stride > 0; stride >>= 1) {
        if (localIndex < stride) {
            sharedData[localIndex] += sharedData[localIndex + stride];
        }
        GroupMemoryBarrierWithGroupSync();
    }

    // 4. 첫 스레드가 결과 저장
    if (localIndex == 0) {
        outputBuffer[groupID.x] = sharedData[0];
    }
}
```

### 컴퓨트 셰이더 활용 예시

```hlsl
// 1. GPU 파티클 시뮬레이션
[numthreads(256, 1, 1)]
void ParticleUpdate(uint id : SV_DispatchThreadID) {
    Particle p = particles[id];

    // 물리 업데이트
    p.velocity += gravity * deltaTime;
    p.position += p.velocity * deltaTime;
    p.lifetime -= deltaTime;

    // 죽은 파티클 재활용
    if (p.lifetime <= 0) {
        p = SpawnParticle();
    }

    particles[id] = p;
}

// 2. GPU 컬링
[numthreads(64, 1, 1)]
void FrustumCull(uint id : SV_DispatchThreadID) {
    BoundingBox bbox = boundingBoxes[id];

    // 프러스텀과 교차 테스트
    bool visible = TestFrustum(frustumPlanes, bbox);

    if (visible) {
        // 가시 목록에 추가 (원자적 연산)
        uint index;
        InterlockedAdd(visibleCount[0], 1, index);
        visibleIndices[index] = id;
    }
}

// 3. 포스트 프로세싱 (블룸, 톤매핑 등)
[numthreads(8, 8, 1)]
void ToneMapping(uint3 id : SV_DispatchThreadID) {
    float3 hdrColor = hdrImage[id.xy].rgb;

    // Reinhard 톤매핑
    float3 ldrColor = hdrColor / (hdrColor + 1.0);

    // 감마 보정
    ldrColor = pow(ldrColor, 1.0 / 2.2);

    outputImage[id.xy] = float4(ldrColor, 1.0);
}
```

### 그래픽스 vs 컴퓨트 선택 기준

| 상황 | 선택 |
|------|------|
| 삼각형 그리기 | Graphics |
| 이미지 전체 처리 | Compute (더 효율적) |
| 버퍼 데이터 처리 | Compute |
| G-Buffer 생성 | Graphics (MRT) |
| 포스트 프로세싱 | Compute |
| 물리/파티클 | Compute |

---

## 요약

| 단계 | 역할 | 유형 |
|------|------|------|
| Input Assembler | 버텍스를 삼각형으로 조립 | 고정 |
| Vertex Shader | 버텍스 위치 변환 | 프로그래밍 |
| Rasterizer | 삼각형 → 픽셀 | 고정 |
| Pixel Shader | 픽셀 색상 결정 | 프로그래밍 |
| Output Merger | 깊이/블렌딩 | 고정(설정가능) |
| Compute Shader | 범용 GPU 계산 | 프로그래밍 |

---

## 다음 단계

Part 2를 이해했다면 Part 3(GPU와 통신하기)로 넘어가자.
질문 있으면 언제든 물어봐.
