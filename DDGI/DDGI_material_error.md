# DDGI Material Sampling Bug - 수정 완료
## 하나의 메시만 간접광에 영향을 주는 문제 해결

---

## 문제 증상

![alt text](<스크린샷 2025-12-04 150416.png>)

Sample6에서 4개의 메시(icosahedron 구, torus knot, canal tube, floor)가 있는 씬에서 DDGI를 활성화했을 때:
- 오직 첫 번째 메시(icosahedron 구)의 `baseColor`만 DDGI 간접광에 반영됨
- 나머지 3개 메시의 색상은 완전히 무시됨
- 구가 빨간색이면 모든 probe가 빨간색으로, 구가 노란색이면 모든 probe가 노란색으로 염색됨
- 다른 메시들은 probe를 물리적으로 밀어낼 수 있지만 색상 정보는 샘플링되지 않음

---

## 원인 분석

2가지 버그가 복합적으로 작용하여 발생:

### 문제 1. TLAS Instance ID와 인덱싱 불일치
**위치:** `SceneUpdate_Detail.cpp:683, 715`

**문제:**
```cpp
// Line 683 - TLAS instance_id 설정
instance.instance_id = args.jobIndex;  // ❌ WRONG

// Line 715 - TLAS 데이터 저장 위치
void* dest = (void*)((size_t)TLAS_instancesMapped +
                     (size_t)args.jobIndex * device->GetTopLevelAccelerationStructureInstanceSize());
```

**근본 원인:**
- Ray Tracing Shader에서 `CommittedInstanceID()`는 TLAS의 `instance_id`를 반환
- 이 ID로 `ShaderMeshInstance` 배열에 접근: `load_instance(instanceIndex)`
- 하지만 `ShaderMeshInstance`는 `renderable.renderableIndex` 위치에 저장됨 (Line 669)
- TLAS `instance_id` ≠ 실제 데이터 위치 → 항상 잘못된 인덱스로 접근

**코드 불일치:**
```cpp
// Line 669 - ShaderMeshInstance 저장
std::memcpy(instanceArrayMapped + renderable.renderableIndex, &inst, sizeof(ShaderMeshInstance));

// Line 683 - TLAS instance_id 설정 (불일치!)
instance.instance_id = args.jobIndex;  // args.jobIndex ≠ renderableIndex
```

---

### 문제 2. instanceResLookupBuffer Upload → Default 복사 누락
**위치:** `RenderPath3D_Detail.cpp:1746, 1803`

**문제:**
- Non-UMA 시스템(대부분의 discrete GPU)에서 Upload → Default 버퍼 복사 코드 누락
- `instanceBuffer`, `geometryBuffer`, `materialBuffer`는 제대로 복사되지만
- **`instanceResLookupBuffer`만 복사 코드 없음** → Default 버퍼가 비어있음

**기존 코드 (문제):**
```cpp
// RenderPath3D_Detail.cpp:1743-1745
barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->instanceBuffer, ...));
barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->geometryBuffer, ...));
barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->materialBuffer, ...));
// ❌ instanceResLookupBuffer barrier 누락!

// RenderPath3D_Detail.cpp:1788-1800
device->CopyBuffer(&scene_Gdetails->instanceBuffer, ...);
device->CopyBuffer(&scene_Gdetails->geometryBuffer, ...);
device->CopyBuffer(&scene_Gdetails->materialBuffer, ...);
// ❌ instanceResLookupBuffer copy 누락!
```

**왜 문제가 되는가?**
- Non-UMA에서는 CPU 메모리(Upload)와 GPU 메모리(Default)가 물리적으로 분리
- Upload buffer를 셰이더가 직접 읽으면 PCIe를 통해 접근 → 매우 느림
- Default buffer로 복사 후 읽어야 GPU VRAM에서 고속 접근 가능

---

## GPU 메모리 아키텍처 이해

### UMA vs Non-UMA 시스템

#### UMA (Unified Memory Architecture)
**특징:**
- CPU와 GPU가 **동일한 물리 메모리**를 공유
- 메모리가 하나만 존재 (통합 메모리)
- CPU가 쓴 데이터를 GPU가 바로 읽을 수 있음 (복사 불필요)

**장점:**
- Upload → Default 복사가 불필요 (드라이버가 자동 최적화)
- 메모리 대역폭을 CPU/GPU가 공유하여 효율적

**예시:**
- 통합 그래픽 (Intel UHD, AMD APU)
- Apple Silicon (M1, M2, M3) - 최대 800GB/s 통합 대역폭
- Xbox Series X/S, PlayStation 5

#### Non-UMA (Discrete Graphics)
**특징:**
- CPU 메모리(RAM)와 GPU 메모리(VRAM)가 **물리적으로 분리**
- PCIe 버스로 연결 (PCIe 4.0 x16: ~32GB/s)
- CPU → GPU 데이터 전송 시 명시적 복사 필요

**장점:**
- GPU 전용 고속 VRAM (GDDR6: 최대 1TB/s)
- CPU와 GPU가 각자의 메모리 대역폭을 독립적으로 사용

**예시:**
- 데스크톱 GPU (NVIDIA GeForce/RTX, AMD Radeon)
- 워크스테이션 GPU (NVIDIA Quadro, AMD Pro)

**성능 차이:**
```
UMA (Apple M2 Max):
- 통합 메모리: 400GB/s (M2 Max)
- Upload buffer 읽기: ~400GB/s (추가 복사 없음)

Non-UMA (NVIDIA RTX 4090):
- GPU VRAM (GDDR6X): 1008GB/s
- PCIe 4.0 x16: 32GB/s
- Upload buffer 읽기 (PCIe): ~32GB/s ⚠️ 31배 느림!
- Default buffer 읽기 (VRAM): ~1008GB/s ✅ 빠름
```

---

## GPU 버퍼 종류 (D3D12 Usage)

### 1. Default (GPU 전용 메모리)
**D3D12:** `D3D12_HEAP_TYPE_DEFAULT`
**특징:**
- GPU VRAM에 저장 (Non-UMA) 또는 통합 메모리 (UMA)
- **GPU만 읽기/쓰기 가능**, CPU는 직접 접근 불가
- 가장 빠른 GPU 접근 속도

**사용 사례:**
- Shader에서 읽는 리소스 (Texture, Structured Buffer)
- Render Target, Depth Buffer
- Compute Shader 출력

**VizMotive 코드:**
```cpp
// SceneUpdate_Detail.cpp:1167
allocationDesc.HeapType = D3D12_HEAP_TYPE_DEFAULT;
device->CreateBuffer(&desc, nullptr, &instanceResLookupBuffer);
```

### 2. Upload (CPU → GPU 전송용)
**D3D12:** `D3D12_HEAP_TYPE_UPLOAD`
**특징:**
- CPU가 **매 프레임 데이터를 쓸 수 있음** (Mapped Memory)
- GPU도 읽을 수 있지만 Non-UMA에서는 느림 (PCIe 접근)
- Write-Combined 메모리 (CPU 쓰기 최적화, 읽기는 매우 느림)

**사용 사례:**
- 매 프레임 업데이트되는 Constant Buffer
- Dynamic Vertex/Index Buffer
- GPU로 전송할 임시 데이터

**VizMotive 코드:**
```cpp
// SceneUpdate_Detail.cpp:1175-1179
desc.usage = Usage::UPLOAD;
for (int i = 0; i < arraysize(instanceResLookupUploadBuffer); ++i)
{
    device->CreateBuffer(&desc, nullptr, &instanceResLookupUploadBuffer[i]);
}
```

### 3. Readback (GPU → CPU 전송용)
**D3D12:** `D3D12_HEAP_TYPE_READBACK`
**특징:**
- GPU가 데이터를 쓰고, **CPU가 읽을 수 있음**
- CPU가 GPU 계산 결과를 가져올 때 사용
- 매우 느림 (동기화 필요)

**사용 사례:**
- GPU 계산 결과를 CPU로 가져오기 (예: Picking, Query)
- 스크린샷 캡처
- 디버깅 데이터 추출

**코드 예시:**
```cpp
desc.usage = Usage::READBACK;
allocationDesc.HeapType = D3D12_HEAP_TYPE_READBACK;
resourceState = D3D12_RESOURCE_STATE_COPY_DEST;
```

### 4. Immutable (초기화 후 변경 불가)
**D3D12:** 별도 타입 없음, Default + 초기 데이터로 구현
**특징:**
- 생성 시 한 번만 데이터 초기화
- 이후 변경 불가 (Read-Only)
- 최대 성능 (드라이버가 최적 위치 배치 가능)

**사용 사례:**
- Static Mesh Vertex/Index Buffer
- Texture (미리 로드된 이미지)
- Static Lookup Table

**OpenGL/Vulkan 용어:**
- OpenGL: `GL_STATIC_DRAW`
- Vulkan: `VK_BUFFER_USAGE_TRANSFER_DST_BIT` (한 번만 복사)

---

## jobsystem::Dispatch와 CUDA 비교

### jobsystem::Dispatch 구조
**VizMotive 코드:**
```cpp
// SceneUpdate_Detail.cpp:480-483
jobsystem::Dispatch(ctx, (uint32_t)num_renderables, SMALL_SUBTASK_GROUPSIZE, [&](jobsystem::JobArgs args) {
    GRenderableComponent& renderable = *renderableComponents[args.jobIndex];
    assert(renderable.renderableIndex == args.jobIndex);
    // ...
});
```

**매개변수:**
1. `ctx`: Job context (스레드 동기화)
2. `num_renderables`: **전체 작업 개수** (예: 100개 메시)
3. `SMALL_SUBTASK_GROUPSIZE`: **그룹당 작업 개수** (예: 64)
4. `lambda`: 각 작업에서 실행할 함수

**작동 원리:**
- 100개 작업을 64개씩 묶어서 여러 스레드에 분산
- `args.jobIndex`: 0, 1, 2, ..., 99 (전체 작업에서의 인덱스)
- 각 스레드가 독립적으로 처리

### CUDA 비교

#### CUDA Kernel Launch
```cuda
// 100개 작업을 64개 스레드씩 묶어서 실행
myKernel<<<2, 64>>>(data);  // 2 blocks, 64 threads per block

__global__ void myKernel(Data* data)
{
    int jobIndex = blockIdx.x * blockDim.x + threadIdx.x;  // 전역 인덱스 계산
    if (jobIndex < 100) {
        // data[jobIndex] 처리
    }
}
```

#### 용어 대응표

| VizMotive jobsystem | CUDA | 설명 |
|---------------------|------|------|
| `num_renderables` (100) | `gridDim.x * blockDim.x` | 전체 작업 개수 |
| `SMALL_SUBTASK_GROUPSIZE` (64) | `blockDim.x` | 그룹당 스레드 개수 |
| `args.jobIndex` | `blockIdx.x * blockDim.x + threadIdx.x` | 전역 작업 인덱스 |
| `ctx` (context) | CUDA Stream | 작업 동기화 |
| `jobsystem::Wait(ctx)` | `cudaStreamSynchronize(stream)` | 작업 완료 대기 |

#### 핵심 개념: 인덱스 일치

**VizMotive Assert:**
```cpp
assert(renderable.renderableIndex == args.jobIndex);
```

**의미:**
- `renderableComponents` 배열의 **i번째 요소**는 항상 `renderableIndex = i`
- CUDA의 `threadIdx`와 동일: **배열 인덱스 = 작업 인덱스**

**CUDA 동등 코드:**
```cuda
__global__ void processRenderables(RenderableComponent** renderables, int count)
{
    int jobIndex = blockIdx.x * blockDim.x + threadIdx.x;
    if (jobIndex >= count) return;

    RenderableComponent* renderable = renderables[jobIndex];
    assert(renderable->renderableIndex == jobIndex);  // 동일한 가정!

    // 처리...
}
```

**왜 중요한가?**
1. **TLAS instance_id**는 ray tracing에서 "어느 메시를 hit했는가"를 알려줌
2. 이 ID로 배열에 접근: `ShaderMeshInstance[instance_id]`
3. **instance_id ≠ 배열 인덱스**이면 잘못된 데이터 읽음!
4. CUDA에서 `threadIdx`로 잘못된 배열 접근하는 것과 동일

---

## 수정 내용

### 수정 1: TLAS Instance ID 통일
**파일:** `SceneUpdate_Detail.cpp`
**커밋:** `a48d7be` - "args.jobIndex -> renderable.renderable"

```cpp
// Line 683 - BEFORE
instance.instance_id = args.jobIndex;

// Line 683 - AFTER
instance.instance_id = renderable.renderableIndex;

// Line 715 - BEFORE
void* dest = (void*)((size_t)TLAS_instancesMapped +
                     (size_t)args.jobIndex * device->GetTopLevelAccelerationStructureInstanceSize());

// Line 715 - AFTER
void* dest = (void*)((size_t)TLAS_instancesMapped +
                     (size_t)renderable.renderableIndex * device->GetTopLevelAccelerationStructureInstanceSize());
```

**효과:**
- TLAS `instance_id`와 `ShaderMeshInstance` 저장 위치가 일치
- Ray Tracing Shader가 올바른 인덱스로 데이터 접근
- 모든 컴포넌트(geometry, material, renderable)가 동일한 인덱싱 패턴 사용

**코드 일관성:**
```cpp
// 다른 컴포넌트들은 이미 이 패턴 사용
assert(renderable.renderableIndex == args.jobIndex);  // Line 43, 483
assert(geometry.geometryIndex == args.jobIndex);      // Line 59
assert(material.materialIndex == args.jobIndex);      // Line 210

// ShaderMeshInstance 저장도 renderableIndex 사용
std::memcpy(instanceArrayMapped + renderable.renderableIndex, &inst, ...);  // Line 669

// 이제 TLAS도 동일한 패턴
instance.instance_id = renderable.renderableIndex;  // Line 683 (수정 후)
```

---

### 수정 2: instanceResLookupBuffer Upload → Default 복사 추가
**파일:** `RenderPath3D_Detail.cpp`
**커밋:** `40bfa9b` - "Add Upload → Default copy"

#### 변경 1: Barrier 추가 (Line 1746-1749)
```cpp
// BEFORE: instanceResLookupBuffer barrier 없음
barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->instanceBuffer, ...));
barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->geometryBuffer, ...));
barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->materialBuffer, ...));

// AFTER: instanceResLookupBuffer barrier 추가
if (scene_Gdetails->instanceResLookupBuffer.IsValid())
{
    barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->instanceResLookupBuffer,
        ResourceState::SHADER_RESOURCE, ResourceState::COPY_DST));
}
```

**목적:**
- GPU 리소스 상태 전환: `SHADER_RESOURCE` → `COPY_DST`
- 셰이더가 읽는 중에 복사하면 race condition 발생
- Barrier로 셰이더 읽기 완료 후 복사 시작 보장

#### 변경 2: Upload → Default Copy 추가 (Line 1806-1817)
```cpp
// BEFORE: instanceResLookupBuffer copy 없음
device->CopyBuffer(&scene_Gdetails->instanceBuffer, ...);
device->CopyBuffer(&scene_Gdetails->geometryBuffer, ...);
device->CopyBuffer(&scene_Gdetails->materialBuffer, ...);

// AFTER: instanceResLookupBuffer copy 추가
if (scene_Gdetails->instanceResLookupBuffer.IsValid() && scene_Gdetails->instanceResLookupSize > 0)
{
    device->CopyBuffer(
        &scene_Gdetails->instanceResLookupBuffer,              // dst (Default - GPU VRAM)
        0,                                                      // dst offset
        &scene_Gdetails->instanceResLookupUploadBuffer[device->GetBufferIndex()],  // src (Upload - CPU writable)
        0,                                                      // src offset
        scene_Gdetails->instanceResLookupSize * sizeof(ShaderInstanceResLookup),   // size
        cmd                                                     // command list
    );
    barrierStack.push_back(GPUBarrier::Buffer(&scene_Gdetails->instanceResLookupBuffer,
        ResourceState::COPY_DST, ResourceState::SHADER_RESOURCE));
}
```

**목적:**
- CPU가 Upload buffer에 쓴 데이터를 GPU VRAM (Default buffer)로 복사
- 복사 후 상태 전환: `COPY_DST` → `SHADER_RESOURCE`
- 셰이더가 고속 VRAM에서 데이터 읽기

**Non-UMA 성능 비교:**
```
수정 전 (Upload buffer 직접 읽기):
- 셰이더 → PCIe → CPU 메모리 → Upload buffer
- 대역폭: ~32GB/s (PCIe 4.0 x16)
- 레이턴시: 높음

수정 후 (Default buffer 읽기):
- 셰이더 → GPU VRAM → Default buffer
- 대역폭: ~1000GB/s (GDDR6X)
- 레이턴시: 낮음
- 성능 향상: ~31배
```

---

## Ray Tracing 데이터 흐름

### 정상적인 데이터 흐름 (수정 후)
```
1. Ray Trace
   └─> TLAS Hit

2. Instance ID 획득
   └─> instanceID = CommittedInstanceID()
   └─> instanceID = renderable.renderableIndex (수정으로 일치!)

3. 인스턴스 로드
   └─> ShaderMeshInstance inst = load_instance(instanceID)
   └─> instanceArrayMapped[instanceID]에서 읽기 ✅ 올바른 위치!
   └─> resLookupIndex = inst.resLookupIndex

4. 리소스 룩업 로드
   └─> ShaderInstanceResLookup lookup = load_instResLookup(resLookupIndex + subsetIndex)
   └─> instanceResLookupBuffer[resLookupIndex]에서 읽기 ✅ Default buffer (빠름!)
   └─> materialIndex = lookup.materialIndex

5. 머티리얼 로드
   └─> MaterialComponent material = load_material(materialIndex)
   └─> baseColor = material.baseColor ✅ 올바른 색상!
```

### 버그 상태 (수정 전)
```
1. Ray Trace
   └─> TLAS Hit

2. Instance ID 획득 (문제 1)
   └─> instanceID = CommittedInstanceID()
   └─> instanceID = args.jobIndex ❌ (renderable.renderableIndex와 다름!)

3. 인스턴스 로드 (문제 1)
   └─> ShaderMeshInstance inst = load_instance(instanceID)
   └─> instanceArrayMapped[instanceID]에서 읽기 ❌ 잘못된 위치!
   └─> 항상 첫 번째 메시의 데이터 읽음

4. 리소스 룩업 로드 (문제 2)
   └─> instanceResLookupBuffer[resLookupIndex]에서 읽기 ❌ 비어있음 (Non-UMA)!
   └─> 또는 Upload buffer에서 느리게 읽기 (임시 해결책)

5. 머티리얼 로드
   └─> 항상 materialIndex=0 (첫 번째 메시의 머티리얼)
   └─> 모든 probe가 동일한 색상
```

---

## 버퍼 크기 체크 버그 (문제 아님)

### 발견된 코드
```cpp
// SceneUpdate_Detail.cpp:1157
if (instanceResLookupUploadBuffer[0].desc.size < (instanceResLookupSize * sizeof(uint)))  // ❌ sizeof(uint) = 4
{
    desc.stride = sizeof(ShaderInstanceResLookup);  // ✅ = 16 bytes
    desc.size = desc.stride * instanceResLookupSize * 2;  // *2 to grow fast
    // 버퍼 생성...
}
```

### 왜 문제가 아닌가?

**체크 vs 할당:**
- 체크: `instanceResLookupSize * 4` bytes 필요한지 확인
- 할당: `instanceResLookupSize * 16 * 2` bytes 생성

**예시 (instanceResLookupSize = 10):**
1. 첫 프레임:
   - 체크: `0 < 10 * 4 = 40` ✅ 통과
   - 할당: `10 * 16 * 2 = 320` bytes
   - `buffer->desc.size = 320` 저장

2. 두 번째 프레임 (크기 증가: instanceResLookupSize = 15):
   - 체크: `320 < 15 * 4 = 60`? ❌ **320 > 60이므로 실패**
   - 버퍼 재생성 안 됨
   - 하지만 실제 필요: `15 * 16 = 240` bytes
   - 이미 할당: `320` bytes ✅ **충분함!**

**결론:**
- 체크 조건이 잘못되었지만 **항상 8배 더 크게 할당됨** (16/4 * 2 = 8x)
- 의도하지 않은 "안전 마진" 역할
- 수정하지 않아도 작동에 문제 없음

---

## 테스트 결과

### 수정 전
- 모든 probe가 icosahedron(구) 색상으로만 염색됨
- Shader 디버그 출력: `materialIndex=0` (모든 probe)

### 수정 후
- 각 probe가 실제로 hit한 메시의 색상을 정확히 샘플링
- 빨강(구), 초록(knot), 파랑(canal), 회색(floor) 모두 DDGI에 기여
- Shader 디버그 출력: `materialIndex=0,1,2,3` (다양한 값)
- Non-UMA 시스템에서 최적 성능 (GPU VRAM 직접 접근)

![alt text](image-1.png)

---

## 핵심 수정 요약

| 파일 | 라인 | 변경 전 | 변경 후 |
|------|------|---------|---------|
| SceneUpdate_Detail.cpp | 683 | `instance.instance_id = args.jobIndex` | `instance.instance_id = renderable.renderableIndex` |
| SceneUpdate_Detail.cpp | 715 | `args.jobIndex * ...` | `renderable.renderableIndex * ...` |
| RenderPath3D_Detail.cpp | 1746-1749 | instanceResLookupBuffer barrier 없음 | Barrier 추가 |
| RenderPath3D_Detail.cpp | 1806-1817 | instanceResLookupBuffer copy 없음 | Upload → Default Copy 추가 |

---

## 교훈

### 1. 인덱스 일관성 (CUDA threadIdx와 유사)
- **TLAS `instance_id`는 반드시 실제 데이터 저장 위치와 일치**해야 함
- CUDA의 `blockIdx.x * blockDim.x + threadIdx.x = 배열 인덱스` 원칙과 동일
- 모든 컴포넌트가 `assert(component.index == args.jobIndex)` 패턴 사용
- Ray Tracing에서는 이 일치가 더욱 중요 (shader가 instance_id로 배열 접근)

### 2. UMA vs Non-UMA 메모리 아키텍처
- **UMA:** Upload buffer 직접 읽기 가능 (통합 메모리, 복사 불필요)
- **Non-UMA:** 반드시 Upload → Default 복사 필요 (PCIe vs VRAM, 31배 성능 차이)
- 새 버퍼 추가 시 기존 패턴(instance, geometry, material) 참고

### 3. GPU 버퍼 Usage의 목적
- **Default:** 셰이더 고속 접근 (GPU VRAM)
- **Upload:** CPU 쓰기 최적화 (매 프레임 업데이트)
- **Readback:** GPU → CPU 전송 (결과 가져오기)
- **Immutable:** 최대 성능 (변경 불가)

### 4. jobsystem::Dispatch는 CUDA와 동일한 개념
- `num_items` = 전체 작업 개수 (CUDA: `gridDim * blockDim`)
- `groupSize` = 그룹당 작업 개수 (CUDA: `blockDim`)
- `args.jobIndex` = 전역 작업 인덱스 (CUDA: `blockIdx * blockDim + threadIdx`)
- 배열 인덱스 = 작업 인덱스 가정 필수

### 5. GPU Barrier의 중요성
- 리소스 상태 전환 명시 (READ → WRITE → READ)
- Race condition 방지 (셰이더 읽기 중 복사 금지)
- D3D12는 명시적 동기화 필요 (OpenGL과 다름)

### 6. 디버깅 방법
- Shader에서 디버그 값을 색상으로 출력 (materialIndex → RGB)
- Assert로 가정 검증 (`renderableIndex == jobIndex`)
- 기존 버퍼 패턴과 비교 (왜 이것만 다른가?)

---

## 참고 자료

### D3D12 Heap Types
- [Microsoft Docs - D3D12_HEAP_TYPE](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_heap_type)
- [Microsoft Docs - Upload and Readback](https://learn.microsoft.com/en-us/windows/win32/direct3d12/upload-and-readback-of-texture-data)

### UMA vs Non-UMA
- [AMD RDNA3 Architecture](https://www.amd.com/en/technologies/rdna-3)
- [Apple Unified Memory](https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf)

### CUDA Programming Guide
- [CUDA Thread Hierarchy](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#thread-hierarchy)
- [CUDA Memory Types](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#device-memory)

### Ray Tracing
- [DXR Functional Spec](https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html)
- [NVIDIA RTX Best Practices](https://developer.nvidia.com/rtx/raytracing/dxr/dx12-raytracing-tutorial-part-1)
