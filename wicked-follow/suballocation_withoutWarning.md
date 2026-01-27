# GPU Buffer Suballocation Without Debug Layer Warnings

## 개요

이 문서는 DX12 Debug Layer 경고 없이 GPU 버퍼 서브할당을 구현하는 방법을 설명합니다.

**목표**: `D3D12_MESSAGE_ID_HEAP_ADDRESS_RANGE_INTERSECTS_MULTIPLE_BUFFERS` 경고를 억제(suppress)하지 않고 근본적으로 해결

**핵심 아이디어**: Aliasing Resource 생성 대신 **단일 버퍼 + 오프셋 기반 뷰** 방식 사용

---

## 1. 현재 WickedEngine 구조 분석

### 1.1 현재 아키텍처 (Aliasing 방식)

```
┌────────────────────────────────────────────────────────────────────────┐
│                     GPUSubAllocator                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              256MB Block Buffer (ID3D12Resource #1)              │  │
│  │              Flags: ALIASING_BUFFER | NO_DEFAULT_DESCRIPTORS     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│         CreateAliasingResource() 호출                                   │
│                              ▼                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Mesh A   │  │ Mesh B   │  │ Mesh C   │  │ Mesh D   │  ...          │
│  │ Resource │  │ Resource │  │ Resource │  │ Resource │               │
│  │ #2       │  │ #3       │  │ #4       │  │ #5       │               │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ⚠️ D3D12_MESSAGE_ID_HEAP_ADDRESS_RANGE_INTERSECTS_MULTIPLE_BUFFERS
        (여러 ID3D12Resource가 동일한 힙 주소 범위를 점유)
```

### 1.2 관련 코드 구조

**MeshComponent 멤버 (wiScene_Components.h:725-728)**:
```cpp
wi::graphics::GPUBuffer generalBuffer;                      // 개별 메시 버퍼 (aliased resource)
wi::graphics::GPUBuffer streamoutBuffer;                    // 동적 버퍼 (skinning 출력)
wi::allocator::PageAllocator::Allocation generalBufferOffsetAllocation;  // 블록 내 offset
wi::graphics::GPUBuffer generalBufferOffsetAllocationAlias; // 256MB 블록 버퍼 참조
```

**BufferSuballocation 구조체 (wiRenderer.h:74-79)**:
```cpp
struct BufferSuballocation
{
    wi::graphics::GPUBuffer alias;                          // 256MB 블록 버퍼
    wi::allocator::PageAllocator::Allocation allocation;    // offset + size 정보
    inline bool IsValid() const { return allocation.IsValid(); }
};
```

**서브할당 생성 (wiScene_Components.cpp:1366-1370)**:
```cpp
// CreateBuffer2()가 내부적으로 CreateAliasingResource() 호출
bool success = device->CreateBuffer2(&bd, init_callback, &generalBuffer,
                                     &suballoc.alias, suballoc.allocation.byte_offset);
generalBufferOffsetAllocation = std::move(suballoc.allocation);
generalBufferOffsetAllocationAlias = std::move(suballoc.alias);  // 블록 버퍼 참조 저장
```

**Index Buffer 바인딩 (wiRenderer.cpp:3387-3396)**:
```cpp
// 서브할당된 경우 블록 버퍼를 바인딩
const GPUBuffer* ib = mesh.generalBufferOffsetAllocation.IsValid()
    ? &mesh.generalBufferOffsetAllocationAlias   // 256MB 블록
    : &mesh.generalBuffer;                        // 독립 버퍼
device->BindIndexBuffer(ib, ibformat, 0, cmd);   // offset은 0으로 바인딩
```

**Index Offset 계산 (wiRenderer.cpp:3422-3432)**:
```cpp
if (mesh.generalBufferOffsetAllocation.IsValid())
{
    // 블록 내 base offset + 메시 내 IB offset
    indexOffset = uint32_t(((uint64_t)mesh.generalBufferOffsetAllocation.byte_offset
                           + ibv.offset) / ib_stride) + subset.indexOffset;
}
else
{
    // 독립 버퍼: 메시 내 offset만 사용
    indexOffset = uint32_t(ibv.offset / ib_stride) + subset.indexOffset;
}
```

### 1.3 경고 발생 원인

```cpp
// wiGraphicsDevice_DX12.cpp:3362-3372
hr = dx12_check(allocationhandler->allocator->CreateAliasingResource(
    alias_internal->allocation.Get(),  // 원본 256MB 블록의 allocation
    alias_offset,                       // 블록 내 offset
    &resourceDesc,
    resourceState,
    nullptr,
    PPV_ARGS(internal_state->resource)  // 새로운 ID3D12Resource 생성!
));
```

**문제**: `CreateAliasingResource()`가 **별도의 ID3D12Resource**를 생성
- 동일한 힙 메모리를 여러 Resource가 참조
- Debug Layer가 "의도된 동작인가?" 확인 차원에서 경고 발생

---

## 2. 순수 오프셋 뷰 방식 설계

### 2.1 새로운 아키텍처

```
┌────────────────────────────────────────────────────────────────────────┐
│                     GPUSubAllocator                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │              256MB Block Buffer (ID3D12Resource #1)              │  │
│  │              Flags: NO ALIASING (일반 버퍼)                       │  │
│  │  ┌─────────┬─────────┬─────────┬─────────┬──────────────────┐   │  │
│  │  │ Mesh A  │ Mesh B  │ Mesh C  │ Mesh D  │     (미사용)      │   │  │
│  │  │ offset=0│ off=4K  │ off=12K │ off=20K │                   │   │  │
│  │  └─────────┴─────────┴─────────┴─────────┴──────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│         CreateSubresource(offset, size) - View만 생성                   │
│                              ▼                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ SRV @0   │  │ SRV @4K  │  │ SRV @12K │  │ SRV @20K │  ...          │
│  │ (View)   │  │ (View)   │  │ (View)   │  │ (View)   │               │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘               │
└────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ✅ NO WARNING
        (단일 Resource에 대한 오프셋 기반 뷰 - 정상 동작)
```

### 2.2 핵심 변경사항

| 구성요소 | 현재 (Aliasing) | 변경 후 (Offset View) |
|---------|----------------|----------------------|
| 블록 버퍼 플래그 | `ALIASING_BUFFER` | 일반 버퍼 (aliasing 없음) |
| 메시별 Resource | `CreateAliasingResource()` → 별도 Resource | 없음 (블록 참조만) |
| 메시별 저장 데이터 | `generalBuffer` (aliased resource) | `blockBufferRef` + `baseOffset` |
| SRV/UAV 생성 | 메시 버퍼에 대해 생성 | 블록 버퍼 + 절대 offset으로 생성 |
| VB/IB 바인딩 | 메시 버퍼 또는 블록 버퍼 | 항상 블록 버퍼 + offset |

### 2.3 새로운 데이터 구조

```cpp
// 변경 전 MeshComponent
struct MeshComponent {
    GPUBuffer generalBuffer;                        // aliased resource (제거)
    GPUBuffer generalBufferOffsetAllocationAlias;   // 256MB 블록 참조
    PageAllocator::Allocation generalBufferOffsetAllocation;
    BufferView ib, vb_pos_wind, vb_nor, ...;        // 메시 내 상대 offset
};

// 변경 후 MeshComponent
struct MeshComponent {
    // generalBuffer 제거! aliased resource 더 이상 생성 안함

    GPUBuffer* blockBufferRef = nullptr;            // 256MB 블록 버퍼 포인터
    uint64_t blockBaseOffset = 0;                   // 블록 내 base offset
    PageAllocator::Allocation allocation;           // 해제용 allocation 정보

    BufferView ib, vb_pos_wind, vb_nor, ...;        // 메시 내 상대 offset (동일)

    // 절대 offset 계산 헬퍼
    uint64_t GetAbsoluteOffset(const BufferView& view) const {
        return blockBaseOffset + view.offset;
    }
};
```

---

## 3. 구현 계획

### 3.1 단계별 구현

#### Phase 1: 데이터 구조 변경

**1.1 MeshComponent 수정**
```cpp
// 제거
GPUBuffer generalBuffer;  // aliased resource는 더 이상 필요 없음

// 변경
GPUBuffer generalBufferOffsetAllocationAlias;  // → GPUBuffer* blockBufferRef;
// 포인터로 변경하여 블록 버퍼를 직접 참조
```

**1.2 BufferSuballocation 구조 수정**
```cpp
struct BufferSuballocation
{
    GPUBuffer* blockBuffer;                         // 블록 버퍼 포인터 (복사 대신)
    wi::allocator::PageAllocator::Allocation allocation;
    inline bool IsValid() const { return allocation.IsValid(); }
    inline uint64_t GetOffset() const { return allocation.byte_offset; }
};
```

#### Phase 2: 버퍼 생성 로직 변경

**2.1 GPUSubAllocator 수정 (wiRenderer.cpp)**
```cpp
struct GPUSubAllocator
{
    static constexpr uint64_t blocksize = 256ull * 1024ull * 1024ull;
    struct Block
    {
        wi::allocator::PageAllocator allocator;
        GPUBuffer buffer;  // 단일 버퍼, aliasing 플래그 제거
    };
    wi::vector<Block> blocks;
    std::mutex locker;
};

BufferSuballocation SuballocateGPUBuffer(uint64_t size)
{
    // ... 기존 할당 로직 ...

    // 새 블록 생성 시 ALIASING_BUFFER 플래그 제거
    GPUBufferDesc desc;
    desc.size = GPUSubAllocator::blocksize;
    desc.bind_flags = BindFlag::SHADER_RESOURCE | BindFlag::VERTEX_BUFFER |
                      BindFlag::INDEX_BUFFER | BindFlag::UNORDERED_ACCESS;
    // desc.misc_flags = ResourceMiscFlag::ALIASING_BUFFER;  // 제거!
    desc.misc_flags = ResourceMiscFlag::NO_DEFAULT_DESCRIPTORS;
    if (device->CheckCapability(GraphicsDeviceCapability::RAYTRACING))
    {
        desc.misc_flags |= ResourceMiscFlag::RAY_TRACING;
    }
    // ...
}
```

**2.2 MeshComponent::CreateRenderData 수정**
```cpp
void MeshComponent::CreateRenderData()
{
    // 서브할당 요청
    BufferSuballocation suballoc = SuballocateGPUBuffer(required_size);

    if (suballoc.IsValid())
    {
        // 변경 전: CreateBuffer2()로 aliased resource 생성
        // device->CreateBuffer2(&bd, init_callback, &generalBuffer,
        //                       &suballoc.alias, suballoc.allocation.byte_offset);

        // 변경 후: 블록 버퍼 참조만 저장
        blockBufferRef = suballoc.blockBuffer;
        blockBaseOffset = suballoc.allocation.byte_offset;
        allocation = std::move(suballoc.allocation);

        // 초기 데이터 업로드는 블록 버퍼의 해당 offset에 직접 수행
        UploadDataToBlockBuffer(blockBufferRef, blockBaseOffset, init_data, data_size);
    }
    else
    {
        // 서브할당 실패: 독립 버퍼 생성 (폴백)
        device->CreateBuffer(&bd, init_data, &standaloneBuffer);
        blockBufferRef = &standaloneBuffer;
        blockBaseOffset = 0;
    }
}
```

#### Phase 3: View 생성 로직 변경

**3.1 SRV/UAV 생성 시 절대 offset 사용**
```cpp
// 변경 전: 메시 버퍼(generalBuffer)에 상대 offset으로 생성
ib.subresource_srv = device->CreateSubresource(&generalBuffer, SubresourceType::SRV,
                                                ib.offset, ib.size, &ib_format);

// 변경 후: 블록 버퍼에 절대 offset으로 생성
uint64_t absoluteOffset = blockBaseOffset + ib.offset;
ib.subresource_srv = device->CreateSubresource(blockBufferRef, SubresourceType::SRV,
                                                absoluteOffset, ib.size, &ib_format);
```

**3.2 모든 BufferView에 적용**
```cpp
void MeshComponent::CreateBufferViews()
{
    const Format ib_format = GetIndexFormat() == IndexBufferFormat::UINT32
        ? Format::R32_UINT : Format::R16_UINT;

    // Index Buffer
    uint64_t ib_abs_offset = blockBaseOffset + ib.offset;
    ib.subresource_srv = device->CreateSubresource(blockBufferRef, SRV,
                                                    ib_abs_offset, ib.size, &ib_format);

    // Position Buffer
    uint64_t pos_abs_offset = blockBaseOffset + vb_pos_wind.offset;
    vb_pos_wind.subresource_srv = device->CreateSubresource(blockBufferRef, SRV,
                                                             pos_abs_offset, vb_pos_wind.size,
                                                             &position_format);

    // ... 나머지 BufferView들도 동일하게 처리 ...
}
```

#### Phase 4: 렌더링 로직 변경

**4.1 Index Buffer 바인딩 단순화**
```cpp
// 변경 전
const GPUBuffer* ib = mesh.generalBufferOffsetAllocation.IsValid()
    ? &mesh.generalBufferOffsetAllocationAlias
    : &mesh.generalBuffer;

// 변경 후: 항상 블록 버퍼 사용
const GPUBuffer* ib = mesh.blockBufferRef;
device->BindIndexBuffer(ib, ibformat, 0, cmd);
```

**4.2 Index Offset 계산 (변경 없음 - 로직 동일)**
```cpp
// 블록 버퍼 사용 시 (서브할당된 경우)
uint32_t indexOffset = uint32_t((mesh.blockBaseOffset + ibv.offset) / ib_stride)
                       + subset.indexOffset;

// 독립 버퍼 사용 시 (서브할당 실패)
if (mesh.blockBaseOffset == 0 && mesh.blockBufferRef == &mesh.standaloneBuffer)
{
    indexOffset = uint32_t(ibv.offset / ib_stride) + subset.indexOffset;
}
```

**4.3 Vertex Buffer 바인딩 (GPU Skinning 등)**
```cpp
// Skinning을 위한 VB 접근 시
uint64_t vb_gpu_address = mesh.blockBufferRef->GetGPUVirtualAddress()
                         + mesh.blockBaseOffset
                         + mesh.vb_pos_wind.offset;
```

#### Phase 5: 초기화/해제 로직

**5.1 데이터 업로드**
```cpp
void UploadDataToBlockBuffer(GPUBuffer* blockBuffer, uint64_t offset,
                              const void* data, uint64_t size)
{
    // CopyAllocator를 사용한 업로드
    CopyAllocator::CopyCMD cmd = copyAllocator.allocate(size);
    memcpy(cmd.uploadbuffer.mapped_data, data, size);

    // 블록 버퍼의 특정 offset에 복사
    cmd.commandList->CopyBufferRegion(
        to_internal(blockBuffer)->resource.Get(),
        offset,                                      // 블록 내 offset
        to_internal(&cmd.uploadbuffer)->resource.Get(),
        0,
        size
    );
    copyAllocator.submit(cmd);
}
```

**5.2 해제 처리**
```cpp
MeshComponent::~MeshComponent()
{
    // PageAllocator에서 할당 해제만 수행
    // 실제 메모리는 블록 버퍼가 관리
    if (allocation.IsValid())
    {
        allocation.free();  // offset 범위만 반환
    }
    // blockBufferRef는 포인터이므로 삭제하지 않음
}
```

---

## 4. VizMotive 적용 고려사항

### 4.1 현재 VizMotive 구조 확인 필요

```cpp
// VizMotive의 BufferSuballocation 구조 확인
// GBackend.h 또는 관련 헤더에서 구조 확인 필요

// Renderer.cpp에서 서브할당 사용 방식 확인 필요
```

### 4.2 예상되는 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `GBackend.h` | BufferSuballocation 구조 수정 |
| `GraphicsDevice_DX12.cpp` | CreateBuffer2() aliasing 로직 제거 |
| `Renderer.cpp` | GPUSubAllocator 블록 생성 플래그 수정 |
| `GComponents.cpp` | MeshComponent 서브할당 사용 부분 수정 |
| `Renderer.cpp` (렌더링) | VB/IB 바인딩, offset 계산 로직 수정 |

### 4.3 호환성 유지

**독립 버퍼 폴백 지원**:
- 서브할당 실패 시 기존처럼 독립 버퍼 생성
- `blockBufferRef`가 자체 버퍼를 가리키도록 설정
- `blockBaseOffset = 0`으로 설정

```cpp
if (!suballoc.IsValid())
{
    // 폴백: 독립 버퍼 생성
    device->CreateBuffer(&bd, init_data, &standaloneBuffer);
    blockBufferRef = &standaloneBuffer;
    blockBaseOffset = 0;
}
```

---

## 5. 장단점 분석

### 5.1 장점

| 항목 | 설명 |
|------|------|
| **경고 제거** | Debug Layer 경고 근본적 해결 |
| **Resource 감소** | ID3D12Resource 객체 수 대폭 감소 (N → 1 per block) |
| **메모리 동일** | GPU 메모리 사용량 변화 없음 |
| **바인딩 최적화** | 동일 블록 내 메시는 IB 리바인딩 불필요 |
| **코드 단순화** | Aliasing 관련 복잡한 로직 제거 |

### 5.2 단점/주의사항

| 항목 | 설명 | 대응 |
|------|------|------|
| **State 공유** | 블록 전체가 동일 resource state | 정적 데이터(VB/IB)는 문제없음 |
| **View 관리** | 절대 offset 계산 필요 | 헬퍼 함수로 추상화 |
| **포인터 관리** | 블록 버퍼 수명 주의 | 블록은 프로그램 종료까지 유지 |
| **Migration 작업** | 기존 코드 전면 수정 | 단계적 적용 가능 |

### 5.3 Resource State 고려

```cpp
// 블록 버퍼는 정적 데이터용이므로 state 전환 불필요
// 생성 시 적절한 초기 state 설정:
D3D12_RESOURCE_STATES initialState =
    D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER |  // VB로 사용
    D3D12_RESOURCE_STATE_INDEX_BUFFER |                 // IB로 사용
    D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE;     // SRV로 사용 (compute)
```

---

## 6. 검증 계획

### 6.1 단위 테스트

1. **블록 버퍼 생성**: ALIASING 플래그 없이 256MB 버퍼 생성 확인
2. **서브할당**: PageAllocator offset 할당 정상 동작 확인
3. **View 생성**: 절대 offset으로 SRV/UAV 생성 확인
4. **바인딩**: VB/IB 바인딩 + Draw 호출 정상 동작 확인

### 6.2 통합 테스트

1. **Debug Layer 활성화**: 경고 필터 제거 후 실행
2. **다중 메시 렌더링**: 여러 메시가 동일 블록 사용 시 정상 렌더링 확인
3. **메시 생성/삭제**: 동적 할당/해제 반복 시 메모리 누수 없음 확인
4. **성능 비교**: 기존 방식 대비 성능 저하 없음 확인

### 6.3 경고 검증

```cpp
// wiGraphicsDevice_DX12.cpp에서 경고 필터 제거
// disabledMessages.push_back(D3D12_MESSAGE_ID_HEAP_ADDRESS_RANGE_INTERSECTS_MULTIPLE_BUFFERS);
// ↑ 이 줄 제거/주석 처리 후 테스트
```

---

## 7. 참고 자료

### 7.1 D3D12 Buffer View 생성

```cpp
// CreateSubresource에서 FirstElement 계산
srv_desc.Buffer.FirstElement = UINT(offset / stride);
srv_desc.Buffer.NumElements = UINT(size / stride);
```

- `FirstElement`: 버퍼 시작 기준 element 단위 offset
- `NumElements`: 해당 view에서 접근 가능한 element 수

### 7.2 GPU Virtual Address

```cpp
// GPU에서 직접 주소 접근 시
D3D12_GPU_VIRTUAL_ADDRESS addr = buffer->GetGPUVirtualAddress() + byte_offset;
```

- VB/IB 바인딩, Raytracing BLAS 등에서 사용
- byte 단위 offset 직접 가산 가능

### 7.3 관련 WickedEngine 코드 위치

| 파일 | 라인 | 내용 |
|------|------|------|
| wiRenderer.cpp | 2764-2829 | GPUSubAllocator 구현 |
| wiScene_Components.cpp | 1360-1378 | CreateRenderData 서브할당 |
| wiScene_Components.cpp | 1380-1398 | BufferView 생성 |
| wiRenderer.cpp | 3386-3432 | 렌더링 시 IB 바인딩/offset 계산 |
| wiGraphicsDevice_DX12.cpp | 3362-3372 | CreateAliasingResource 호출 |
| wiGraphicsDevice_DX12.cpp | 4956-5074 | CreateSubresource 구현 |

---

## 8. VizMotive 구현 결과

### 8.1 구현 완료 (2026-01-27)

VizMotive에서 순수 오프셋 기반 서브할당을 성공적으로 구현했습니다.

#### 변경된 파일 목록

| 파일 | 변경 내용 |
|------|----------|
| `GraphicsDevice_DX12.cpp` | `HEAP_ADDRESS_RANGE_INTERSECTS_MULTIPLE_BUFFERS` 경고 필터 제거, `UploadToBufferRegion()` API 추가 |
| `GBackendDevice.h` | `UploadToBufferRegion()` 가상 함수 선언 |
| `GBackend.h` | `BufferSuballocation` 구조체를 포인터 방식으로 변경 |
| `GComponents.h` | `GPrimBuffers` 구조체에 `blockBufferRef`, `blockBaseOffset`, 헬퍼 메서드 추가 |
| `Renderer.cpp` | `ALIASING_BUFFER` 플래그 제거, VB/IB 바인딩 로직 수정 |
| `GeometryComponent.cpp` | 데이터 업로드 및 뷰 생성 시 절대 offset 사용, Raytracing BLAS 버퍼 참조 수정 |

### 8.2 핵심 코드 변경

#### BufferSuballocation 구조체 (`GBackend.h`)
```cpp
// 변경 전: alias 버퍼 복사본 저장
struct BufferSuballocation
{
    GPUBuffer alias;  // 256MB 블록 버퍼 복사본
    allocator::PageAllocator::Allocation allocation;
};

// 변경 후: 포인터로 블록 버퍼 참조
struct BufferSuballocation
{
    GPUBuffer* blockBuffer = nullptr;  // 256MB 블록 버퍼 포인터
    allocator::PageAllocator::Allocation allocation;
    inline bool IsValid() const { return blockBuffer != nullptr && allocation.IsValid(); }
    inline uint64_t GetOffset() const { return allocation.byte_offset; }
};
```

#### GPrimBuffers 구조체 (`GComponents.h`)
```cpp
struct GPrimBuffers
{
    // 기존 필드 제거: generalBufferOffsetAllocationAlias

    // 새로운 필드
    graphics::GPUBuffer* blockBufferRef = nullptr;  // 256MB 블록 버퍼 포인터
    uint64_t blockBaseOffset = 0;                   // 블록 내 base offset
    allocator::PageAllocator::Allocation generalBufferOffsetAllocation;  // 해제용

    // 헬퍼 메서드
    const graphics::GPUBuffer* GetRenderBuffer() const {
        return blockBufferRef ? blockBufferRef : &generalBuffer;
    }
    bool IsSuballocated() const { return blockBufferRef != nullptr; }
};
```

#### 블록 버퍼 생성 (`Renderer.cpp`)
```cpp
// 변경 전
desc.misc_flags = ResourceMiscFlag::ALIASING_BUFFER | ResourceMiscFlag::NO_DEFAULT_DESCRIPTORS;

// 변경 후: ALIASING_BUFFER 제거
desc.misc_flags = ResourceMiscFlag::NO_DEFAULT_DESCRIPTORS;
```

#### 데이터 업로드 (`GeometryComponent.cpp`)
```cpp
if (suballoc.IsValid())
{
    part_buffers.blockBufferRef = suballoc.blockBuffer;
    part_buffers.blockBaseOffset = suballoc.GetOffset();

    // 블록 버퍼의 해당 offset에 직접 업로드
    std::vector<uint8_t> tempBuffer(bd.size);
    init_callback(tempBuffer.data());
    device->UploadToBufferRegion(part_buffers.blockBufferRef,
                                  part_buffers.blockBaseOffset,
                                  tempBuffer.data(), bd.size);
}
```

#### 뷰 생성 시 절대 offset 사용 (`GeometryComponent.cpp`)
```cpp
const GPUBuffer* renderBuffer = part_buffers.GetRenderBuffer();
const uint64_t viewBaseOffset = part_buffers.IsSuballocated() ? part_buffers.blockBaseOffset : 0;

// 절대 offset으로 SRV 생성
ib.subresource_srv = device->CreateSubresource(renderBuffer, SubresourceType::SRV,
                                                viewBaseOffset + ib.offset, ib.size, &ib_format);
```

#### 렌더링 시 버퍼 바인딩 (`Renderer.cpp`)
```cpp
// 변경 전
const GPUBuffer* ib = part_buffer.generalBufferOffsetAllocation.IsValid()
    ? &part_buffer.generalBufferOffsetAllocationAlias
    : &part_buffer.generalBuffer;

// 변경 후
const GPUBuffer* ib = part_buffer.GetRenderBuffer();

// Index offset 계산
if (part_buffer.IsSuballocated())
{
    indexOffset = uint32_t((part_buffer.blockBaseOffset + part_buffer.ib.offset) / ib_stride);
}
```

#### Raytracing BLAS 버퍼 참조 (`GeometryComponent.cpp`)
```cpp
// 변경 전
geometry.triangles.vertex_buffer = part_buffers.generalBuffer;
geometry.triangles.vertex_byte_offset = part_buffers.vbPosW.offset;

// 변경 후
const GPUBuffer* renderBuffer = part_buffers.GetRenderBuffer();
const uint64_t baseOffset = part_buffers.IsSuballocated() ? part_buffers.blockBaseOffset : 0;
geometry.triangles.vertex_buffer = *renderBuffer;
geometry.triangles.vertex_byte_offset = baseOffset + part_buffers.vbPosW.offset;
```

### 8.3 UploadToBufferRegion API (`GraphicsDevice_DX12.cpp`)

```cpp
void GraphicsDevice_DX12::UploadToBufferRegion(GPUBuffer* buffer, uint64_t offset,
                                                const void* data, uint64_t size) const
{
    if (buffer == nullptr || data == nullptr || size == 0)
        return;

    auto internal_state = to_internal(buffer);
    if (internal_state == nullptr || internal_state->resource == nullptr)
        return;

    // UPLOAD 버퍼 (UMA 모드): mapped memory에 직접 쓰기
    if (buffer->mapped_data != nullptr)
    {
        std::memcpy((uint8_t*)buffer->mapped_data + offset, data, size);
    }
    else
    {
        // DEFAULT 버퍼: CopyAllocator 사용
        CopyAllocator::CopyCMD cmd = copyAllocator.allocate(size);
        if (cmd.IsValid())
        {
            std::memcpy(cmd.uploadbuffer.mapped_data, data, size);
            cmd.commandList->CopyBufferRegion(
                internal_state->resource.Get(),
                offset,
                to_internal(&cmd.uploadbuffer)->resource.Get(),
                0,
                size
            );
            copyAllocator.submit(cmd);
        }
    }
}
```

### 8.4 테스트 결과

- ✅ Debug Layer 경고 없음 (필터 제거 후에도)
- ✅ 일반 렌더링 정상 동작
- ✅ Raytracing (BLAS) 정상 동작
- ✅ 서브할당 실패 시 독립 버퍼 폴백 정상 동작

---

## 9. 결론

순수 오프셋 뷰 방식은 **Debug Layer 경고를 근본적으로 해결**하면서 기존 aliasing 방식과 **동일한 메모리 효율**을 제공합니다.

### 핵심 변경 요약

| 항목 | WickedEngine (Aliasing) | VizMotive (Offset View) |
|------|-------------------------|-------------------------|
| 블록 버퍼 플래그 | `ALIASING_BUFFER` | 일반 버퍼 |
| 메시별 Resource | `CreateAliasingResource()` → 별도 Resource | 없음 (포인터 참조) |
| 메시별 저장 | `generalBufferOffsetAllocationAlias` (복사본) | `blockBufferRef` (포인터) + `blockBaseOffset` |
| 경고 필터 | `HEAP_ADDRESS_RANGE_INTERSECTS_MULTIPLE_BUFFERS` 필터링 | 필터 불필요 |
| Debug Layer | 경고 억제 필요 | 경고 없음 |

### WickedEngine vs VizMotive 접근법 비교

**WickedEngine의 선택**:
- `CreateAliasingResource()`를 사용하여 각 메시가 별도의 ID3D12Resource를 가짐
- 장점: 기존 코드 수정 최소화, Resource 단위 상태 관리 가능
- 단점: Debug Layer 경고 발생 → 경고 필터링 필요

**VizMotive의 선택**:
- 단일 블록 버퍼 + 오프셋 기반 뷰 방식
- 장점: Debug Layer 경고 없음, Resource 객체 수 감소
- 단점: 데이터 구조 및 렌더링 로직 수정 필요

두 방식 모두 **메모리 효율은 동일**하지만, VizMotive 방식이 **DX12의 의도에 더 부합**하는 구현입니다.
