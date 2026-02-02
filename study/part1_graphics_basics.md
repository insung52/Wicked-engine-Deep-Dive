# Part 1: 그래픽스 기초 개념

Part 0에서 컴퓨터가 어떻게 동작하는지 배웠다.
이제 "화면에 그림을 그린다"는 게 실제로 무슨 의미인지 알아보자.

---

## 1.1 화면에 픽셀이 찍히는 과정

### 모니터는 어떻게 동작하는가?

모니터는 결국 **픽셀의 2D 배열**이다.

```
1920 x 1080 모니터:
┌────────────────────────────────────┐
│ ■ ■ ■ ■ ■ ■ ■ ... (1920개) ■ ■ ■  │ ← 1번째 줄
│ ■ ■ ■ ■ ■ ■ ■ ... (1920개) ■ ■ ■  │ ← 2번째 줄
│ ■ ■ ■ ■ ■ ■ ■ ...           ■ ■ ■  │
│ ...                                │
│ ■ ■ ■ ■ ■ ■ ■ ... (1920개) ■ ■ ■  │ ← 1080번째 줄
└────────────────────────────────────┘

총 픽셀 수: 1920 × 1080 = 2,073,600개
```

각 픽셀은 **RGB 값**을 가진다:
```cpp
struct Pixel {
    uint8_t r;  // 0~255 빨강
    uint8_t g;  // 0~255 초록
    uint8_t b;  // 0~255 파랑
};
// 3바이트 × 2백만 = 약 6MB (한 프레임)
```

### 프레임과 프레임레이트

모니터는 초당 여러 번 화면을 갱신한다:
- 60Hz 모니터: 초당 60번 (16.67ms마다)
- 144Hz 모니터: 초당 144번 (6.94ms마다)

```
시간 →
│ 프레임1 │ 프레임2 │ 프레임3 │ ...
   16ms      16ms      16ms      (60Hz 기준)
```

**게임이 60fps로 돌아간다** = 매 16ms마다 새로운 그림(프레임)을 만들어낸다.

### 더블 버퍼링

문제: 그림을 그리는 중에 모니터가 읽어가면?
```
GPU: [■■■■□□□□□□] 절반만 그린 상태
모니터: 이걸 화면에 표시 → 찢어진 화면 (Tearing)
```

해결: **더블 버퍼링**
```
┌─────────────┐     ┌─────────────┐
│ Front Buffer│     │ Back Buffer │
│ (화면 표시) │     │ (그리는 중) │
└─────────────┘     └─────────────┘
       ↑                   ↓
    모니터가            GPU가
    읽어감             그리는 중

그리기 완료되면 두 버퍼를 교체 (Swap)
```

```cpp
// 의사 코드
while (running) {
    // Back Buffer에 그림
    renderScene(backBuffer);

    // Front ↔ Back 교체
    swapBuffers();

    // 이제 방금 그린 게 화면에 보임
}
```

### VSync (수직 동기화)

**문제**: 버퍼 교체가 모니터 갱신 중간에 일어나면?
→ 여전히 Tearing 발생

**해결**: VSync - 모니터가 갱신을 시작할 때만 교체

```
모니터 갱신:  |---갱신---|---갱신---|---갱신---|
              ↑          ↑          ↑
              여기서만 Swap 허용

VSync ON:  부드럽지만 입력 지연 있을 수 있음
VSync OFF: 빠르지만 Tearing 발생 가능
```

---

## 1.2 좌표계와 변환 (MVP 행렬)

### 3D 물체를 2D 화면에 그리려면?

3D 세계의 물체 → 2D 화면의 픽셀

이 과정에 **여러 좌표계**를 거친다:

```
Local Space (모델 좌표)
    ↓  [Model Matrix]
World Space (월드 좌표)
    ↓  [View Matrix]
View Space (카메라 좌표)
    ↓  [Projection Matrix]
Clip Space (클립 좌표)
    ↓  [원근 나눗셈 + 뷰포트 변환]
Screen Space (화면 좌표)
```

### Local Space (로컬/모델 좌표)

모델 자체의 좌표. 모델 중심이 (0,0,0).

```cpp
// 큐브의 버텍스 (로컬 좌표)
// 중심이 원점, 크기가 2x2x2
Vertex cubeVertices[] = {
    {-1, -1, -1},  // 왼쪽 아래 뒤
    { 1, -1, -1},  // 오른쪽 아래 뒤
    { 1,  1, -1},  // 오른쪽 위 뒤
    {-1,  1, -1},  // 왼쪽 위 뒤
    {-1, -1,  1},  // 왼쪽 아래 앞
    { 1, -1,  1},  // 오른쪽 아래 앞
    { 1,  1,  1},  // 오른쪽 위 앞
    {-1,  1,  1},  // 왼쪽 위 앞
};
```

### World Space (월드 좌표)

게임 세계에서의 위치.

```
World Space:
                    ↑ Y (위)
                    │
                    │    ╱ Z (앞)
                    │   ╱
                    │  ╱
                    │ ╱
────────────────────┼────────────→ X (오른쪽)
                   ╱│
                  ╱ │
                 ╱  │

큐브 A: 위치 (10, 0, 5)
큐브 B: 위치 (-5, 2, 0)
플레이어: 위치 (0, 1, -10)
```

### Model Matrix (모델 행렬)

Local → World 변환. **이동, 회전, 크기 조절**.

```cpp
// 4x4 행렬 (동차 좌표계 사용)
// 왜 4x4? → 이동(Translation)을 행렬 곱으로 표현하려고

// 이동 행렬 (x로 10, y로 5 이동)
Matrix4 translation = {
    1, 0, 0, 10,
    0, 1, 0,  5,
    0, 0, 1,  0,
    0, 0, 0,  1
};

// 크기 행렬 (2배 확대)
Matrix4 scale = {
    2, 0, 0, 0,
    0, 2, 0, 0,
    0, 0, 2, 0,
    0, 0, 0, 1
};

// 회전 행렬 (Y축 기준 45도)
float angle = 45 * (PI / 180);
Matrix4 rotationY = {
    cos(angle),  0, sin(angle), 0,
    0,           1, 0,          0,
    -sin(angle), 0, cos(angle), 0,
    0,           0, 0,          1
};

// 모델 행렬 = Scale × Rotation × Translation
// (순서 중요! 행렬 곱은 교환법칙 성립 안 함)
Matrix4 modelMatrix = scale * rotationY * translation;
```

적용:
```cpp
// 로컬 좌표 → 월드 좌표
Vector4 localPos = {1, 0, 0, 1};  // w=1 (점)
Vector4 worldPos = modelMatrix * localPos;
```

### View Space (뷰/카메라 좌표)

카메라 기준 좌표. 카메라가 원점, 카메라가 보는 방향이 -Z.

```
카메라 시점:
            ↑ Y (위)
            │
            │
    ────────┼────────→ X (오른쪽)
           ╱│
          ╱ │
         ╱  │
        ↙ -Z (카메라가 보는 방향)

카메라 위치가 (0,0,0)
카메라 앞에 있는 물체 = Z가 음수
```

### View Matrix (뷰 행렬)

World → View 변환. 카메라의 역행렬.

```cpp
// 카메라 정보
Vector3 cameraPos = {0, 5, 10};      // 카메라 위치
Vector3 target = {0, 0, 0};          // 바라보는 지점
Vector3 up = {0, 1, 0};              // 위쪽 방향

// LookAt 함수로 View Matrix 생성
Matrix4 viewMatrix = lookAt(cameraPos, target, up);

// lookAt 내부 구현 (개념)
Matrix4 lookAt(Vector3 eye, Vector3 target, Vector3 up) {
    Vector3 zAxis = normalize(eye - target);     // 카메라 뒤 방향
    Vector3 xAxis = normalize(cross(up, zAxis)); // 카메라 오른쪽
    Vector3 yAxis = cross(zAxis, xAxis);         // 카메라 위

    // 회전 + 이동의 조합
    return Matrix4(
        xAxis.x, xAxis.y, xAxis.z, -dot(xAxis, eye),
        yAxis.x, yAxis.y, yAxis.z, -dot(yAxis, eye),
        zAxis.x, zAxis.y, zAxis.z, -dot(zAxis, eye),
        0,       0,       0,        1
    );
}
```

### Projection (투영)

3D → 2D 변환. 두 가지 방식:

**1. Perspective (원근 투영)** - 현실적
```
멀리 있는 물체가 작게 보임
     ╲         ╱
      ╲   ●   ╱   ← 멀리 있는 물체 (작게)
       ╲     ╱
        ╲   ╱
         ╲ ╱
    카메라 위치
         ╱╲
        ╱  ╲
       ╱ ●  ╲     ← 가까운 물체 (크게)
      ╱      ╲
```

**2. Orthographic (직교 투영)** - 2D 게임, UI
```
멀리 있어도 크기 동일
    │       │
    │   ●   │   ← 멀리 있어도 같은 크기
    │       │
    │   ●   │   ← 가까워도 같은 크기
    │       │
```

```cpp
// Perspective Projection Matrix
Matrix4 perspective(float fov, float aspect, float near, float far) {
    float tanHalfFov = tan(fov / 2);

    return Matrix4(
        1/(aspect*tanHalfFov), 0,            0,                         0,
        0,                     1/tanHalfFov, 0,                         0,
        0,                     0,            -(far+near)/(far-near),   -2*far*near/(far-near),
        0,                     0,            -1,                        0
    );
}

// 일반적인 값
Matrix4 projMatrix = perspective(
    45.0f * (PI/180),  // FOV: 45도
    16.0f / 9.0f,      // Aspect Ratio: 16:9
    0.1f,              // Near Plane: 0.1
    1000.0f            // Far Plane: 1000
);
```

### MVP 행렬

세 행렬을 합친 것:

```cpp
Matrix4 mvpMatrix = projMatrix * viewMatrix * modelMatrix;

// 셰이더에서 사용
Vector4 clipPos = mvpMatrix * localPos;
```

**왜 미리 곱해두나?**
- 버텍스마다 3번 곱셈 vs 1번 곱셈
- 1만 개 버텍스면 3만 번 → 1만 번으로 줄어듦

### Clip Space와 NDC

Projection 후의 좌표:
```cpp
// Clip Space 좌표 (아직 w로 안 나눔)
Vector4 clipPos = projMatrix * viewPos;
// clipPos = (x, y, z, w)

// NDC (Normalized Device Coordinates)
// w로 나눠서 -1 ~ 1 범위로 정규화
Vector3 ndc = {
    clipPos.x / clipPos.w,  // -1 ~ 1
    clipPos.y / clipPos.w,  // -1 ~ 1
    clipPos.z / clipPos.w   // 0 ~ 1 (DX) 또는 -1 ~ 1 (OpenGL)
};
```

```
NDC 공간:
        (-1,1)        (1,1)
           ┌────────────┐
           │            │
           │   (0,0)    │
           │     ●      │
           │            │
           └────────────┘
        (-1,-1)       (1,-1)
```

### Screen Space (화면 좌표)

NDC → 실제 픽셀 좌표:

```cpp
// Viewport 변환
int screenX = (ndc.x + 1) * 0.5 * screenWidth;
int screenY = (1 - ndc.y) * 0.5 * screenHeight;  // Y는 뒤집힘

// 예: 1920x1080 화면
// NDC (0, 0) → Screen (960, 540) 화면 중앙
// NDC (-1, 1) → Screen (0, 0) 왼쪽 위
// NDC (1, -1) → Screen (1920, 1080) 오른쪽 아래
```

---

## 1.3 버텍스, 인덱스, 메시

### 버텍스 (Vertex)

3D 공간의 한 점 + 추가 정보.

```cpp
// 가장 기본적인 버텍스
struct SimpleVertex {
    float x, y, z;  // 위치
};

// 실제로 쓰는 버텍스
struct Vertex {
    float position[3];  // 위치 (x, y, z)
    float normal[3];    // 법선 벡터 (조명 계산용)
    float texCoord[2];  // 텍스처 좌표 (u, v)
    float tangent[3];   // 탄젠트 (노말맵용)
    uint8_t color[4];   // 버텍스 컬러 (r, g, b, a)
};
```

### 법선 벡터 (Normal)

표면이 향하는 방향. 조명 계산에 필수.

```
        법선(N)
          ↑
          │
    ──────┼──────  표면
          │

빛이 표면에 수직으로 들어오면 → 밝음
빛이 표면과 평행하면 → 어두움
```

```cpp
// 법선과 빛 방향으로 밝기 계산 (Lambert)
float brightness = max(0, dot(normal, lightDirection));
```

### 텍스처 좌표 (UV)

2D 이미지를 3D 표면에 입히기 위한 좌표.

```
텍스처 이미지:          3D 모델에 매핑:
(0,0)────────(1,0)
  │            │         ┌─────┐
  │   이미지   │    →    │     │ 큐브 한 면
  │            │         └─────┘
(0,1)────────(1,1)

각 버텍스에 (u,v) 좌표 지정
→ 그 사이는 보간됨
```

### 삼각형 (Triangle)

GPU는 **삼각형만** 그릴 수 있다.

왜 삼각형인가?
- 3개의 점은 **항상** 한 평면 위에 있음 (4개는 아닐 수 있음)
- 수학적으로 다루기 쉬움
- 어떤 다각형이든 삼각형으로 분해 가능

```
사각형 = 삼각형 2개:
    0───1          0───1    0
    │   │    →     │ ╱  +   ╲│
    │   │          │╱        ╲│
    3───2          3          3───2

    삼각형1: 0-1-3
    삼각형2: 1-2-3
```

### 인덱스 (Index)

버텍스를 재사용하기 위한 번호.

```cpp
// 인덱스 없이 사각형 (6개 버텍스 필요)
Vertex vertices[] = {
    // 삼각형 1
    {0, 0, 0}, {1, 0, 0}, {0, 1, 0},
    // 삼각형 2
    {1, 0, 0}, {1, 1, 0}, {0, 1, 0},  // 중복!
};

// 인덱스 사용 (4개 버텍스 + 6개 인덱스)
Vertex vertices[] = {
    {0, 0, 0},  // 0
    {1, 0, 0},  // 1
    {1, 1, 0},  // 2
    {0, 1, 0},  // 3
};

uint16_t indices[] = {
    0, 1, 3,  // 삼각형 1
    1, 2, 3,  // 삼각형 2
};
```

**왜 인덱스를 쓰나?**
```
큐브: 8개 버텍스, 12개 삼각형 (36개 인덱스)
- 인덱스 없이: 36개 버텍스 × 32바이트 = 1,152 바이트
- 인덱스 사용: 8개 버텍스 × 32바이트 + 36개 × 2바이트 = 328 바이트
→ 3.5배 절약!

복잡한 모델은 수십만 버텍스 → 메모리 절약 효과 큼
```

### 메시 (Mesh)

버텍스 배열 + 인덱스 배열 = 하나의 3D 모델

```cpp
struct Mesh {
    std::vector<Vertex> vertices;
    std::vector<uint32_t> indices;

    // GPU에 업로드된 버퍼들
    ID3D12Resource* vertexBuffer;
    ID3D12Resource* indexBuffer;
};

// 사용 예
Mesh cubeMesh;
cubeMesh.vertices = { /* 8개 버텍스 */ };
cubeMesh.indices = { /* 36개 인덱스 */ };
uploadToGPU(cubeMesh);
```

### Winding Order (감기 순서)

삼각형의 앞면/뒷면을 구분하는 방법.

```
시계 방향 (CW):          반시계 방향 (CCW):
      0                        0
     ╱╲                       ╱╲
    ╱  ╲                     ╱  ╲
   ╱ →→ ╲                   ╱ ←← ╲
  2──────1                 1──────2

  0→1→2 (시계)             0→1→2 (반시계)
```

GPU는 이걸 보고 **Backface Culling** (뒷면 제거)을 한다:
```cpp
// DX12에서 래스터라이저 설정
D3D12_RASTERIZER_DESC rasterDesc = {};
rasterDesc.FrontCounterClockwise = TRUE;  // 반시계가 앞면
rasterDesc.CullMode = D3D12_CULL_MODE_BACK;  // 뒷면 제거
```

카메라에서 봤을 때 시계/반시계인지로 앞뒷면 판단 → 뒷면은 그리지 않음 → 성능 2배 향상.

---

## 1.4 텍스처와 샘플링

### 텍스처란?

GPU에서 사용하는 이미지 데이터.

```cpp
// 텍스처 = 2D 픽셀(텍셀) 배열
struct Texture {
    uint32_t width;
    uint32_t height;
    uint8_t* data;  // RGBA 데이터
    // 실제로는 GPU 메모리에 있음
};

// 512x512 RGBA 텍스처 = 512 × 512 × 4 = 1MB
```

### 텍스처 좌표 (UV)

```
UV 좌표계 (DX 기준):
(0,0)────────→ U (1,0)
  │
  │
  ↓
  V
(0,1)

OpenGL은 V가 반대 (위가 0, 아래가 1)
```

### 텍스처 샘플링

UV 좌표로 텍스처에서 색상을 가져오는 것.

```cpp
// 셰이더에서
float4 color = texture.Sample(sampler, uv);
```

**문제**: UV가 정확히 픽셀 중앙이 아니면?

```
텍스처 (4x4):
┌───┬───┬───┬───┐
│ A │ B │ C │ D │
├───┼───┼───┼───┤
│ E │ F │ G │ H │
├───┼───┼───┼───┤
│ I │ J │ K │ L │
├───┼───┼───┼───┤
│ M │ N │ O │ P │
└───┴───┴───┴───┘

UV = (0.3, 0.3) → A, B, E, F 사이 어딘가
어떤 색을 반환할 것인가?
```

### 필터링 모드

**1. Point (Nearest) - 가장 가까운 픽셀**
```
UV (0.3, 0.3) → 가장 가까운 F 반환
결과: 픽셀화된 느낌 (마인크래프트 스타일)
```

**2. Bilinear - 4개 픽셀 보간**
```
UV (0.3, 0.3) → A, B, E, F를 거리에 따라 섞음
결과: 부드럽지만 가까이 가면 흐림
```

**3. Trilinear - Bilinear + Mipmap 보간**
```
두 Mipmap 레벨에서 Bilinear 후 그 결과도 보간
결과: 더 부드러움
```

**4. Anisotropic - 비등방성 필터링**
```
비스듬히 볼 때 특별 처리
결과: 바닥 텍스처 등이 멀리서도 선명
```

```cpp
// DX12 샘플러 설정
D3D12_SAMPLER_DESC samplerDesc = {};
samplerDesc.Filter = D3D12_FILTER_ANISOTROPIC;
samplerDesc.MaxAnisotropy = 16;  // 품질 (1~16)
samplerDesc.AddressU = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.AddressV = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
```

### Mipmap

텍스처의 축소 버전들.

```
Mip 0: 1024x1024 (원본)
Mip 1: 512x512
Mip 2: 256x256
Mip 3: 128x128
...
Mip 10: 1x1

멀리 있는 물체 → 작은 Mip 사용
→ 메모리 대역폭 절약 + 깜빡임(aliasing) 방지
```

```
┌─────────────────┐
│                 │
│    Mip 0        │  ← 가까울 때
│   (원본)        │
│                 │
└─────────────────┘
    ┌────────┐
    │ Mip 1  │  ← 조금 멀 때
    └────────┘
      ┌───┐
      │M2 │  ← 더 멀 때
      └───┘
       ┌┐
       └┘  ← Mip 3...
```

```cpp
// Mipmap 생성 (보통 텍스처 로딩 시)
// 각 레벨은 이전 레벨의 2x2 픽셀 평균

// GPU에서 자동으로 적절한 Mip 선택
// (화면에 차지하는 크기에 따라)
```

### Texture Address Mode

UV가 0~1 범위를 벗어나면?

```cpp
// Wrap: 반복
UV (1.5, 0.5) → (0.5, 0.5)처럼 동작
┌───┬───┬───┐
│ A │ A │ A │
├───┼───┼───┤
│ A │ A │ A │  ← 패턴 반복
└───┴───┴───┘

// Clamp: 가장자리 색 유지
UV (1.5, 0.5) → (1.0, 0.5) 가장자리 색
┌───┬───┬───┐
│ A │ B │ B │
├───┼───┼───┤  ← 가장자리 늘어남
│ C │ D │ D │
└───┴───┴───┘

// Mirror: 거울 반복
┌───┬───┬───┐
│ A │ B │ B │
├───┼───┼───┤  ← 뒤집어서 반복
│ A │ B │ B │
└───┴───┴───┘
```

### 텍스처 포맷

```cpp
// 일반 컬러 텍스처
DXGI_FORMAT_R8G8B8A8_UNORM      // 32bit RGBA

// HDR 텍스처
DXGI_FORMAT_R16G16B16A16_FLOAT  // 64bit RGBA (높은 정밀도)

// 법선맵
DXGI_FORMAT_R8G8B8A8_SNORM      // 부호 있는 정규화 (-1~1)

// 깊이 버퍼
DXGI_FORMAT_D32_FLOAT           // 32bit 깊이
DXGI_FORMAT_D24_UNORM_S8_UINT   // 24bit 깊이 + 8bit 스텐실

// 압축 포맷 (GPU에서 직접 사용)
DXGI_FORMAT_BC1_UNORM           // 6:1 압축 (DXT1)
DXGI_FORMAT_BC7_UNORM           // 3:1 압축 (고품질)
```

**BC (Block Compression)**:
```
원본: 1024x1024 RGBA = 4MB
BC1:  1024x1024 = 0.67MB (6배 작음)
BC7:  1024x1024 = 1.33MB (3배 작음, 고품질)

GPU가 압축 상태로 직접 읽음 → 대역폭 절약
```

---

## 1.5 셰이더란 무엇인가

### 셰이더 = GPU에서 실행되는 프로그램

```
CPU 프로그램:              GPU 프로그램(셰이더):
- C++로 작성               - HLSL/GLSL로 작성
- CPU에서 실행             - GPU에서 실행
- 순차적 처리              - 병렬 처리
- 복잡한 로직 가능         - 단순하지만 대량 처리
```

### 셰이더 종류

```
입력 데이터 (버텍스, 인덱스)
         │
         ▼
┌─────────────────────┐
│   Vertex Shader     │  ← 버텍스마다 실행
│   (정점 변환)        │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   [Hull Shader]     │  ← 테셀레이션용 (선택)
│   [Domain Shader]   │
│   [Geometry Shader] │  ← 기하 변형 (선택)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Rasterizer        │  ← 고정 기능 (프로그래밍 불가)
│   (삼각형→픽셀)      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Pixel Shader      │  ← 픽셀마다 실행
│   (색상 결정)        │
└─────────┬───────────┘
          │
          ▼
     출력 (화면)
```

### Vertex Shader (버텍스 셰이더)

각 버텍스의 위치를 변환한다.

```hlsl
// HLSL (High Level Shading Language)

// 입력 구조체
struct VSInput {
    float3 position : POSITION;
    float3 normal : NORMAL;
    float2 texCoord : TEXCOORD0;
};

// 출력 구조체
struct VSOutput {
    float4 position : SV_POSITION;  // 필수: 클립 공간 위치
    float3 worldNormal : NORMAL;
    float2 texCoord : TEXCOORD0;
    float3 worldPos : TEXCOORD1;
};

// 상수 버퍼 (CPU에서 설정)
cbuffer PerObject : register(b0) {
    float4x4 worldMatrix;
    float4x4 viewProjMatrix;
};

// 버텍스 셰이더 메인 함수
VSOutput VSMain(VSInput input) {
    VSOutput output;

    // 로컬 → 월드
    float4 worldPos = mul(float4(input.position, 1), worldMatrix);
    output.worldPos = worldPos.xyz;

    // 월드 → 클립
    output.position = mul(worldPos, viewProjMatrix);

    // 법선도 월드로 변환 (이동 무시, 회전만)
    output.worldNormal = mul(input.normal, (float3x3)worldMatrix);

    // UV는 그대로 전달
    output.texCoord = input.texCoord;

    return output;
}
```

**핵심**: 모든 버텍스에 대해 **병렬로** 실행됨.
```
버텍스 0 → GPU 코어 0 → VSMain() 실행
버텍스 1 → GPU 코어 1 → VSMain() 실행
버텍스 2 → GPU 코어 2 → VSMain() 실행
...
1만 개 버텍스를 수천 개 코어가 동시에 처리
```

### Pixel Shader (픽셀 셰이더)

각 픽셀의 최종 색상을 결정한다.

```hlsl
// 픽셀 셰이더 입력 = 버텍스 셰이더 출력이 보간된 것
struct PSInput {
    float4 position : SV_POSITION;
    float3 worldNormal : NORMAL;
    float2 texCoord : TEXCOORD0;
    float3 worldPos : TEXCOORD1;
};

// 텍스처와 샘플러
Texture2D diffuseTexture : register(t0);
SamplerState linearSampler : register(s0);

// 조명 정보
cbuffer Lighting : register(b1) {
    float3 lightDirection;
    float3 lightColor;
    float3 ambientColor;
};

// 픽셀 셰이더 메인 함수
float4 PSMain(PSInput input) : SV_TARGET {
    // 텍스처에서 색상 샘플링
    float4 texColor = diffuseTexture.Sample(linearSampler, input.texCoord);

    // 법선 정규화 (보간 후 길이가 1이 아닐 수 있음)
    float3 normal = normalize(input.worldNormal);

    // Lambert 조명 계산
    float NdotL = max(0, dot(normal, -lightDirection));
    float3 diffuse = texColor.rgb * lightColor * NdotL;

    // 환경광 추가
    float3 ambient = texColor.rgb * ambientColor;

    // 최종 색상
    float3 finalColor = ambient + diffuse;

    return float4(finalColor, texColor.a);
}
```

### 보간 (Interpolation)

버텍스 셰이더 출력 → 래스터라이저 → 픽셀 셰이더 입력

```
삼각형의 세 버텍스:
    A (빨강, UV(0,0))
     ╱╲
    ╱  ╲
   ╱ P  ╲    ← P 지점의 픽셀
  ╱      ╲
 B────────C
(녹색,UV(0,1)) (파랑,UV(1,1))

픽셀 P의 값은 A, B, C에서 보간됨:
- 색상: 빨강과 녹색과 파랑 사이 어딘가
- UV: (0,0), (0,1), (1,1) 사이 어딘가

보간 방식: Barycentric Coordinates (무게중심 좌표)
```

### Compute Shader (컴퓨트 셰이더)

그래픽스 파이프라인과 **별개로** GPU에서 범용 계산.

```hlsl
// 스레드 그룹 크기 정의
[numthreads(256, 1, 1)]
void CSMain(uint3 threadID : SV_DispatchThreadID) {
    // threadID.x = 0, 1, 2, ..., N-1

    // 예: 배열의 모든 원소를 2배
    outputBuffer[threadID.x] = inputBuffer[threadID.x] * 2;
}
```

```cpp
// CPU에서 호출
commandList->Dispatch(numElements / 256, 1, 1);
// 256개 스레드씩 그룹으로 실행
```

**Compute Shader 사용 예**:
- 파티클 시뮬레이션
- 포스트 프로세싱
- GPU 컬링
- 물리 시뮬레이션

---

## 1.6 프레임버퍼

### 프레임버퍼란?

렌더링 결과가 저장되는 버퍼들의 집합.

```
프레임버퍼 = {
    Color Buffer (색상 버퍼)     - RGB 또는 RGBA
    Depth Buffer (깊이 버퍼)     - 각 픽셀의 깊이값
    Stencil Buffer (스텐실 버퍼) - 마스킹용
}
```

### Color Buffer (렌더 타겟)

픽셀의 색상이 저장되는 곳.

```cpp
// DX12에서 렌더 타겟 생성
D3D12_RESOURCE_DESC rtDesc = {};
rtDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
rtDesc.Width = 1920;
rtDesc.Height = 1080;
rtDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
rtDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET;

device->CreateCommittedResource(..., &renderTarget);
```

**MRT (Multiple Render Targets)**:
한 번의 드로우 콜로 여러 버퍼에 동시 출력.

```hlsl
// 픽셀 셰이더에서 여러 출력
struct PSOutput {
    float4 color : SV_TARGET0;    // 색상
    float4 normal : SV_TARGET1;   // 법선
    float4 position : SV_TARGET2; // 위치
};
```

Deferred Rendering에서 G-Buffer 생성에 사용.

### Depth Buffer (깊이 버퍼)

각 픽셀의 카메라로부터의 거리.

```
왜 필요한가?

물체 A (가까움, 깊이 0.3)
물체 B (멀리, 깊이 0.7)

B를 먼저 그리고 A를 나중에 그리면?
→ 깊이 테스트로 A가 B를 가림 (올바름)

A를 먼저 그리고 B를 나중에 그리면?
→ 깊이 테스트로 B가 A를 못 가림 (올바름)

깊이 버퍼 없이는 그리는 순서가 중요해짐 (정렬 필요)
```

```
깊이 테스트 과정:
1. 픽셀 셰이더가 색상과 깊이 출력
2. 현재 깊이 버퍼 값과 비교
3. 새 깊이가 더 가까우면:
   - 색상 버퍼 업데이트
   - 깊이 버퍼 업데이트
4. 새 깊이가 더 멀면:
   - 무시 (이미 더 가까운 물체가 있음)
```

```cpp
// DX12 깊이 스텐실 설정
D3D12_DEPTH_STENCIL_DESC dsDesc = {};
dsDesc.DepthEnable = TRUE;
dsDesc.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ALL;
dsDesc.DepthFunc = D3D12_COMPARISON_FUNC_LESS;  // 더 가까우면 통과
```

### Depth 정밀도 문제

깊이 버퍼는 비선형 분포:

```
Near=0.1, Far=1000 일 때:

거리      깊이 버퍼 값
0.1       0.0
1.0       0.9
10.0      0.99
100.0     0.999
1000.0    1.0

→ 가까운 곳에 정밀도 집중
→ 멀리서 Z-fighting (깜빡임) 발생
```

**해결책**:
```cpp
// 1. Near를 최대한 멀리
near = 0.1  (X)
near = 1.0  (O)

// 2. Reversed-Z
// 깊이 버퍼를 뒤집어서 사용
// Far = 0, Near = 1
dsDesc.DepthFunc = D3D12_COMPARISON_FUNC_GREATER;

// 3. Floating point depth buffer
DXGI_FORMAT_D32_FLOAT  // 24bit 정수 대신 32bit 부동소수점
```

### Stencil Buffer (스텐실 버퍼)

픽셀별 8비트 정수 마스크.

```
사용 예:
- 거울 반사 영역 마킹
- 포탈 렌더링
- 그림자 볼륨
- UI 마스킹

┌─────────────────┐
│        1        │  ← 스텐실 값 1인 영역만 그리기
│   ┌───────┐     │
│   │   0   │     │  ← 스텐실 값 0인 영역은 스킵
│   └───────┘     │
│        1        │
└─────────────────┘
```

```cpp
// 스텐실 연산 예: 거울
// Pass 1: 거울 영역에 스텐실 1 쓰기
// Pass 2: 스텐실이 1인 곳에만 반사된 물체 그리기

D3D12_DEPTH_STENCIL_DESC dsDesc = {};
dsDesc.StencilEnable = TRUE;
dsDesc.StencilReadMask = 0xFF;
dsDesc.StencilWriteMask = 0xFF;
dsDesc.FrontFace.StencilPassOp = D3D12_STENCIL_OP_REPLACE;
dsDesc.FrontFace.StencilFunc = D3D12_COMPARISON_FUNC_ALWAYS;
```

### 프레임버퍼 예시: Deferred Rendering

```
G-Buffer (여러 렌더 타겟):
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ RT0: Albedo  │ │ RT1: Normal  │ │ RT2: Rough/  │ │ Depth Buffer │
│ (색상)       │ │ (법선)       │ │ Metal       │ │ (깊이)       │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
       │                │                │                │
       └────────────────┴────────────────┴────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │    Lighting Pass      │
                    │  (G-Buffer 읽어서     │
                    │   조명 계산)          │
                    └───────────┬───────────┘
                                │
                        ┌───────▼───────┐
                        │  Final Image  │
                        └───────────────┘
```

---

## 요약

| 개념 | 핵심 |
|------|------|
| 프레임 | 화면 갱신 단위, 더블 버퍼링으로 Tearing 방지 |
| MVP 행렬 | Local→World→View→Clip 변환 |
| 버텍스 | 위치 + 법선 + UV + 기타 속성 |
| 인덱스 | 버텍스 재사용으로 메모리 절약 |
| 텍스처 | 2D 이미지, 샘플링으로 색상 가져옴 |
| 셰이더 | GPU에서 병렬 실행되는 프로그램 |
| 프레임버퍼 | Color + Depth + Stencil 버퍼 집합 |

---

## 다음 단계

Part 1을 이해했다면 Part 2(그래픽스 파이프라인)로 넘어가자.
질문 있으면 언제든 물어봐.
