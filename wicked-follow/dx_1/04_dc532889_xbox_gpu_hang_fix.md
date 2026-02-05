# 커밋 #4: dc532889 - xbox gpu hang fix

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `dc532889` |
| 날짜 | 2025-03-16 |
| 작성자 | Turánszki János |
| 카테고리 | 안정성 (Stability) |
| 우선순위 | 높음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice.h | `CreateBufferZeroed`, `CreateBufferCleared` 헬퍼 함수 추가 (+10줄) |
| wiEmittedParticle.cpp | indirect 버퍼 초기화 방식 변경 |
| wiRenderer.cpp | indirect 버퍼 초기화 방식 변경 |
| wiHairParticle.cpp | `InitializeGPUDataIfNeeded` 함수 제거 |
| wiOcean.cpp | 수동 `zero_data` 제거 |
| wiScene.cpp | SurfelGI, DDGI, impostor 버퍼 초기화 방식 변경 |

---

## 문제 상황

### 증상
Xbox에서 초기화되지 않은 indirect 버퍼로 인한 **GPU hang** 발생.
Windows PC에서는 문제없이 동작하지만, Xbox나 특정 드라이버에서 크래시.

### 원인
`ExecuteIndirect`가 미초기화된 버퍼의 **쓰레기값**을 draw/dispatch 파라미터로 해석.

---

## 배경 지식: Indirect Buffer란?

### 일반적인 Draw Call (Direct)

**CPU가 직접 파라미터를 지정**:

```cpp
// CPU 코드
commandList->DrawIndexedInstanced(
    indexCount,      // 인덱스 개수: 36
    instanceCount,   // 인스턴스 개수: 100
    startIndex,      // 시작 인덱스: 0
    baseVertex,      // 베이스 버텍스: 0
    startInstance    // 시작 인스턴스: 0
);
```

```
CPU ──[36, 100, 0, 0, 0]──▶ GPU: "36개 인덱스로 100개 그려라"
```

**특징**: CPU가 미리 "몇 개 그릴지" 알아야 함

### Indirect Draw Call (GPU-Driven)

**GPU가 버퍼에서 파라미터를 읽음**:

```cpp
// CPU 코드 - "저 버퍼에 있는 값대로 그려라"
commandList->ExecuteIndirect(
    commandSignature,   // 명령 형식
    maxCommandCount,    // 최대 명령 수
    argumentBuffer,     // ← 파라미터가 담긴 버퍼!
    argumentOffset,     // 오프셋
    countBuffer,        // (옵션) 실제 명령 수
    countOffset
);
```

```
         ┌─────────────────────────┐
         │ argumentBuffer (GPU)    │
         │ [36, 100, 0, 0, 0]      │  ← GPU가 직접 읽음
         └────────────┬────────────┘
                      │
GPU: "버퍼 보니까 36개 인덱스로 100개 그리라네" ──▶ 실행
```

**특징**: CPU는 "몇 개 그릴지" 몰라도 됨. GPU가 스스로 결정!

### 레스토랑 비유로 이해하기

**Direct Draw (일반 주문)**:
```
손님(CPU): "파스타 3개, 피자 2개 주세요"
주방(GPU): "네, 파스타 3개 피자 2개요"
```
손님이 정확한 개수를 미리 알고 주문.

**Indirect Draw (자율 뷔페)**:
```
손님(CPU): "저기 적힌 메모대로 만들어주세요"
         ┌──────────────────┐
         │ 메모지 (버퍼)     │
         │ 파스타: 3        │  ← 주방이 직접 읽음
         │ 피자: 2          │
         └──────────────────┘
주방(GPU): "메모 보니까 파스타 3개 피자 2개네"
```
손님은 메모에 뭐가 적혔는지 몰라도 됨. **메모지 내용은 누군가가 미리 써놔야 함!**

---

## Indirect Buffer의 용도

### 1. GPU Culling (가장 일반적)

CPU는 모든 오브젝트를 제출하고, **GPU가 보이는 것만 선별**:

```
Frame 시작:
  CPU: "오브젝트 10,000개 있어. GPU야 알아서 그려"

GPU Culling Compute Shader:
  for each object:
    if (절두체 안에 있음 && 오클루전 통과):
      indirectBuffer에 draw 파라미터 기록
      drawCount++

GPU Draw:
  ExecuteIndirect(indirectBuffer, drawCount)
  → 실제로는 500개만 그림 (나머지 9,500개는 컬링됨)
```

**장점**: CPU-GPU 동기화 없이 효율적인 culling

### 2. GPU Particle System

파티클 개수가 **GPU에서 동적으로 결정**:

```
Particle Update Compute Shader:
  살아있는 파티클 개수 카운트 → indirectBuffer에 기록

Particle Render:
  ExecuteIndirect(indirectBuffer)
  → 실제 살아있는 파티클만 그림
```

### 3. 계층적 렌더링 (LOD, Cluster 등)

GPU가 LOD 레벨이나 클러스터를 결정하고 직접 draw call 생성.

---

## 문제의 핵심: 미초기화 버퍼

### 정상 동작 (초기화된 버퍼)

```cpp
// 버퍼 생성 시 0으로 초기화
device->CreateBufferZeroed(&desc, &indirectBuffer);
```

```
indirectBuffer = [0, 0, 0, 0, 0]

첫 프레임 Culling 전:
  ExecuteIndirect(indirectBuffer)
  → drawCount = 0 → 아무것도 안 그림 ✅

Culling 후:
  indirectBuffer = [36, 100, 0, 0, 0]
  ExecuteIndirect(indirectBuffer)
  → 36개 인덱스로 100개 인스턴스 그림 ✅
```

### 문제 상황 (미초기화 버퍼)

```cpp
// 버퍼 생성 시 초기화 안함
device->CreateBuffer(&desc, nullptr, &indirectBuffer);
```

```
indirectBuffer = [0xCDCDCDCD, 0xCDCDCDCD, ...]  ← 쓰레기값!

첫 프레임 Culling 전:
  ExecuteIndirect(indirectBuffer)

GPU가 해석:
  indexCount = 0xCDCDCDCD = 3,452,816,845
  instanceCount = 0xCDCDCDCD = 3,452,816,845

GPU: "34억개 인덱스로 34억개 인스턴스 그려야 하네..."

결과:
  - GPU hang (무한 대기)
  - 시스템 크래시
  - TDR (Timeout Detection and Recovery)
```

### 메모지 비유로 문제 이해

```
손님(CPU): "저기 메모대로 만들어주세요"

         ┌──────────────────────────┐
         │ 메모지 (미초기화)         │
         │ 파스타: ????            │  ← 이전 손님 낙서
         │ 피자: 34억개            │  ← 쓰레기값
         └──────────────────────────┘

주방(GPU): "피자 34억개...? 평생 만들어도 못 끝내겠네" 💀
```

**교훈**: 메모지는 **반드시 깨끗하게 지운 상태**로 주방에 넘겨야 함!

---

## 왜 PC에서는 괜찮고 Xbox에서만 문제인가?

### 메모리 초기화 차이

| 플랫폼 | 할당된 메모리 초기 상태 | 결과 |
|--------|------------------------|------|
| Windows (대부분) | 0으로 초기화되는 경우 많음 | 우연히 동작 |
| Xbox | 초기화 안 됨 (성능 최적화) | 크래시 |
| 일부 GPU 드라이버 | 이전 데이터 잔존 | 간헐적 크래시 |

### Debug vs Release

| 빌드 | 메모리 상태 | 결과 |
|------|-------------|------|
| Debug | 0xCDCDCDCD 등 패턴 채움 | 큰 값 → 크래시 발생 |
| Release | 이전 데이터 잔존 | 운에 따라 동작 또는 크래시 |

**결론**: "PC에서 잘 되니까 괜찮다"는 **틀린 생각**!

---

## 해결: CreateBufferZeroed 헬퍼 함수

### 추가된 함수 (wiGraphicsDevice.h)

```cpp
// 지정된 값으로 채우면서 버퍼 생성
bool CreateBufferCleared(const GPUBufferDesc* desc, uint8_t value, GPUBuffer* buffer) const
{
    return CreateBuffer2(desc, [&](void* dest) {
        std::memset(dest, value, desc->size);  // CPU에서 메모리 초기화
    }, buffer);
}

// 0으로 채우면서 버퍼 생성 (가장 흔한 케이스)
bool CreateBufferZeroed(const GPUBufferDesc* desc, GPUBuffer* buffer) const
{
    return CreateBufferCleared(desc, 0, buffer);
}
```

### CreateBuffer2의 동작

```cpp
// CreateBuffer2는 콜백을 통해 초기 데이터 설정
bool CreateBuffer2(const GPUBufferDesc* desc,
                   std::function<void(void*)> init_callback,
                   GPUBuffer* buffer) const
{
    // 1. 스테이징 메모리 할당 (CPU 접근 가능)
    void* mapped_data = AllocateStagingMemory(desc->size);

    // 2. 콜백으로 데이터 초기화
    init_callback(mapped_data);  // memset(dest, 0, size)

    // 3. GPU 버퍼 생성 및 데이터 복사
    CreateBuffer(desc, mapped_data, buffer);

    // 4. 스테이징 메모리 해제
    FreeStagingMemory(mapped_data);
}
```

---

## 코드 변경 상세

### 변경 전 (다양한 수동 초기화 방식)

**방법 1: 별도 벡터 생성**
```cpp
wi::vector<uint8_t> zerodata(buf.size);  // 메모리 할당 + 0 초기화
device->CreateBuffer(&buf, zerodata.data(), &ddgi.ray_buffer);
```

**방법 2: 명시적 초기 데이터 배열**
```cpp
uint indirect_data[] = { 0,0,0, 0,0,0, 0,0,0 };  // 스택에 배열
device->CreateBuffer(&buf, &indirect_data, &surfelgi.indirectBuffer);
```

**방법 3: nullptr 후 GPU ClearUAV (비효율)**
```cpp
device->CreateBuffer(&bd, nullptr, &generalBuffer);
// ... 나중에 렌더링 시점에 ...
device->ClearUAV(&generalBuffer, 0, cmd);  // GPU 커맨드 추가 필요
```

**문제점**:
- 일관성 없음
- 실수로 초기화 누락 가능
- 방법 3은 GPU 커맨드 오버헤드

### 변경 후 (헬퍼 함수 통일)

```cpp
device->CreateBufferZeroed(&buf, &ddgi.ray_buffer);
device->CreateBufferZeroed(&buf, &surfelgi.indirectBuffer);
device->CreateBufferZeroed(&bd, &generalBuffer);
```

**장점**:
- 일관된 API
- 실수 방지 (함수 이름이 의도를 명확히 표현)
- CPU 단계에서 초기화 완료 → GPU 커맨드 불필요

---

## 변경된 사용처

| 파일 | 버퍼 | 변경 전 | 변경 후 |
|------|------|---------|---------|
| `wiEmittedParticle.cpp` | `indirectBuffers` | `CreateBuffer(nullptr)` | `CreateBufferZeroed()` |
| `wiRenderer.cpp` | `INDIRECT_DEBUG_0/1` | `CreateBuffer(nullptr)` | `CreateBufferZeroed()` |
| `wiHairParticle.cpp` | `generalBuffer` | `CreateBuffer` + `ClearUAV` | `CreateBufferZeroed()` |
| `wiOcean.cpp` | `buffer_Float2_Ht` 등 | 수동 `zero_data` 벡터 | `CreateBufferZeroed()` |
| `wiScene.cpp` | SurfelGI 버퍼들 | 명시적 배열 | `CreateBufferZeroed()` |
| `wiScene.cpp` | DDGI 버퍼들 | 수동 `zerodata` 벡터 | `CreateBufferZeroed()` |
| `wiScene.cpp` | `impostorBuffer` | `CreateBuffer(nullptr)` | `CreateBufferZeroed()` |

---

## 부가 효과: InitializeGPUDataIfNeeded 제거

### 제거된 코드 (wiHairParticle.cpp)

```cpp
// 기존: 매 프레임 초기화 체크 필요
void HairParticleSystem::InitializeGPUDataIfNeeded(CommandList cmd)
{
    if (gpu_initialized)
        return;

    // GPU 커맨드로 UAV 클리어
    device->ClearUAV(&generalBuffer, 0, cmd);
    device->Barrier(...);

    gpu_initialized = true;
}

// wiRenderer.cpp에서 호출
for (auto& hair : scene->hairs)
{
    hair.InitializeGPUDataIfNeeded(cmd);  // 매 프레임 체크
}
```

### 변경 후

```cpp
// 버퍼 생성 시점에 이미 초기화 완료
device->CreateBufferZeroed(&bd, &generalBuffer);

// InitializeGPUDataIfNeeded 함수 자체가 불필요해짐
// → 함수 삭제
// → gpu_initialized 멤버 변수 삭제
// → 매 프레임 체크 루프 삭제
```

**이점**:
- GPU 커맨드 오버헤드 제거
- 런타임 조건 체크 제거
- 코드 단순화

---

## VizMotive 적용

### 적용 일자
2025-01-26

### 적용 현황

**이미 적용됨**:
- `GBackendDevice.h`: `CreateBufferZeroed`, `CreateBufferCleared` 함수 존재
- `SceneUpdate_Detail.cpp`: DDGI 버퍼들 `CreateBufferZeroed` 사용

**새로 수정된 파일**:

| 파일 | 라인 | 버퍼 | 변경 |
|------|------|------|------|
| `ShaderEngine.cpp` | 138, 140 | `BUFFERTYPE_INDIRECT_DEBUG_0/1` | `CreateBuffer` → `CreateBufferZeroed` |
| `SortLib.cpp` | 46 | `indirectBuffer` | `CreateBuffer` → `CreateBufferZeroed` |
| `RenderPath3D_Detail.cpp` | 704, 706 | `BUFFERTYPE_INDIRECT_DEBUG_0/1` | `CreateBuffer` → `CreateBufferZeroed` |
| `RenderPath3D_Detail.cpp` | 2194 | `res.bins` | `CreateBuffer` → `CreateBufferZeroed` |
| `GaussianSplatting_Detail.cpp` | 32 | `res.indirectBuffer` | `CreateBuffer` → `CreateBufferZeroed` |

### 수정 예시

```cpp
// ShaderEngine.cpp:138-141
// 변경 전
bd.misc_flags = ResourceMiscFlag::BUFFER_RAW | ResourceMiscFlag::INDIRECT_ARGS;
device->CreateBuffer(&bd, nullptr, &buffers[BUFFERTYPE_INDIRECT_DEBUG_0]);
device->CreateBuffer(&bd, nullptr, &buffers[BUFFERTYPE_INDIRECT_DEBUG_1]);

// 변경 후
bd.misc_flags = ResourceMiscFlag::BUFFER_RAW | ResourceMiscFlag::INDIRECT_ARGS;
device->CreateBufferZeroed(&bd, &buffers[BUFFERTYPE_INDIRECT_DEBUG_0]);
device->CreateBufferZeroed(&bd, &buffers[BUFFERTYPE_INDIRECT_DEBUG_1]);
```

---

## Indirect Buffer 사용 시 체크리스트

### 버퍼 생성 시

```cpp
// ❌ 잘못된 방법
GPUBufferDesc desc;
desc.misc_flags = ResourceMiscFlag::INDIRECT_ARGS;
device->CreateBuffer(&desc, nullptr, &buffer);  // 미초기화!

// ✅ 올바른 방법
GPUBufferDesc desc;
desc.misc_flags = ResourceMiscFlag::INDIRECT_ARGS;
device->CreateBufferZeroed(&desc, &buffer);  // 0으로 초기화
```

### 식별 방법

`INDIRECT_ARGS` 플래그가 있으면 반드시 초기화:

```cpp
if (desc.misc_flags & ResourceMiscFlag::INDIRECT_ARGS) {
    // 반드시 CreateBufferZeroed 또는 초기 데이터 제공!
}
```

---

## 요약

| 변경 전 | 변경 후 |
|---------|---------|
| 다양한 수동 초기화 방식 | `CreateBufferZeroed` 통일 |
| 실수로 초기화 누락 가능 | 함수 이름이 의도 명확히 표현 |
| GPU ClearUAV 런타임 오버헤드 | CPU 생성 시점에 초기화 완료 |
| Xbox GPU hang | 안전한 0 초기화 |

### 핵심 교훈

> **Indirect Buffer = 반드시 초기화!**
>
> GPU가 버퍼 내용을 "명령"으로 해석하므로,
> 쓰레기값 = 터무니없는 명령 = GPU hang
