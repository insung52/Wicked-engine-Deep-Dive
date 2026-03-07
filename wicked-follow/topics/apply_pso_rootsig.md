# VizMotive 적용: PSO & Root Signature

**[Notebooklm pdf 문서 : pso_rootsig](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/pdf/pso_rootsig.pdf)**

## 개요

WickedEngine의 PSO/Root Signature 개선(topic_pso_rootsig.md)을 참고하여,
VizMotive에 실제로 적용할 변경 사항을 분석하고 적용한 기록.

> **주의**: topic_pso_rootsig.md의 VizMotive 적용 현황은 일부 오류가 있다.
> 심층 분석을 통해 실제 상태를 재확인한 결과를 이 문서에 기록한다.

---

## 실제 적용 항목 정리

| # | 항목 | 원본 커밋 | Topic doc | 실제 상태 | 이 문서에서 |
|---|------|----------|-----------|-----------|------------|
| 1 | Dynamic PSO 캐시 히트 시 active_pso 미갱신 | dx1 #11 | ✅ | ❌ 버그 있음 | 수정 |
| 2 | Static PSO early return | dx1 #11 | ✅ | ⚠️ 미완 | 수정 |
| 3 | Root Descriptor SRV/UAV buffer offset | dx2 #16 | ✅ | ❌ 미적용 | 수정 |
| - | Dynamic Depth Bias (PSO Flags) | dx1 #14 | ⏭️ | ⏭️ | 미적용 유지 |
| - | Root Signature 수명 연장 | dx1 #16 | ✅ | ✅ (구조 다름) | 변경 불필요 |

---

## 배경 지식: PSO와 DescriptorBinder

> **개념 참고 문서**
> - PSO 기본 개념: [part3_gpu_communication.md §3.3](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part3_gpu_communication.md#33-pso-pipeline-state-object)
> - Root Signature 기본: [part3_gpu_communication.md §3.4](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part3_gpu_communication.md#34-루트-시그니처-root-signature)
> - 엔진 레벨 PSO 관리 (Static/Dynamic PSO, active_pso, rootsig_optimizer, DescriptorBinder): [part6_pso_engine_implementation.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part6_pso_engine_implementation.md)

### PSO(Pipeline State Object)란

GPU 파이프라인 전체 설정(셰이더, 래스터라이저, 블렌드, 깊이 스텐실 등)을
미리 컴파일한 불변 객체. 드로우콜마다 PSO를 교체하는 비용이 있어서
**같은 PSO를 반복 바인딩하는 것을 피하는 최적화**가 중요하다.

```
Static PSO:  이미 컴파일된 resource (ID3D12PipelineState) 보유
Dynamic PSO: resource == nullptr → 드로우 직전에 실시간으로 생성/캐시
```

### active_pso의 역할

`CommandList_DX12`의 `active_pso` 포인터는 현재 커맨드리스트에 바인딩된 PSO를 추적한다.
이 포인터가 `DescriptorBinder::flush()`에서 사용된다:

```cpp
// GraphicsDevice_DX12.cpp:1802
auto pso_internal = graphics ? to_internal(commandlist.active_pso) : to_internal(commandlist.active_cs);
// → pso_internal->rootsig_optimizer를 사용해 descriptor 바인딩 최적화
```

`active_pso`가 실제 바인딩된 PSO와 다르면 **잘못된 Root Signature Optimizer**를 참조하게 된다.
이는 descriptor 바인딩 오류나 크래시로 이어질 수 있다.

### Root Descriptor vs Descriptor Table

셰이더에 버퍼/텍스처를 바인딩하는 두 가지 방법:

```
Descriptor Table 방식:
  → SRV descriptor를 heap에 등록 → GPU 핸들(heap 내 주소) 전달
  → descriptor 안에 FirstElement = offset/stride 로 이미 offset 반영됨
  → SetGraphicsRootDescriptorTable(slot, gpu_handle)

Root Descriptor 방식:
  → GPU 메모리 주소를 직접 전달
  → SetGraphicsRootShaderResourceView(slot, gpu_address)
  → ← 이 주소에 byte offset을 직접 더해야 함!
```

Root Descriptor는 descriptor heap을 거치지 않아 바인딩이 빠르지만,
서브리소스(buffer suballocation)의 byte offset을 **직접 계산해서 더해야** 한다.

---

## 1. Dynamic PSO 캐시 히트 시 active_pso 미갱신 버그

### 문제

`BindPipelineState()` (GraphicsDevice_DX12.cpp:6810)의 Dynamic PSO 경로:

```cpp
void GraphicsDevice_DX12::BindPipelineState(const PipelineState* pso, CommandList cmd)
{
    ...
    if (internal_state->resource != nullptr)  // Static PSO
    {
        ...
    }
    else  // Dynamic PSO
    {
        PipelineHash pipeline_hash;
        pipeline_hash.pso = pso;
        pipeline_hash.renderpass_hash = commandlist.renderpass_info.get_hash();
        if (commandlist.prev_pipeline_hash == pipeline_hash)
        {
            return;  // ❌ active_pso 갱신 없이 return!
        }
        commandlist.prev_pipeline_hash = pipeline_hash;
        commandlist.dirty_pso = true;
    }

    ...
    commandlist.active_pso = pso;  // ← 이 줄에 도달하지 못함
}
```

**문제 시나리오:**

```
1. BindComputeShader() 호출
   → commandlist.active_pso = nullptr  (line 6866)

2. BindPipelineState(dynamic_pso) 호출 (처음)
   → hash 다름 → 통과 → active_pso = dynamic_pso ← 정상

3. BindPipelineState(dynamic_pso) 호출 (두 번째, 같은 PSO)
   → hash 같음 → return  ← active_pso 갱신 안 됨!
   → active_pso는 여전히 dynamic_pso (운 좋으면 맞지만...)

4. 만약 중간에 다른 PSO가 바인딩됐다가 다시 돌아오는 경우
   → active_pso 상태가 실제와 달라짐
   → DescriptorBinder::flush()에서 wrong rootsig_optimizer 사용
   → 잘못된 slot에 descriptor 바인딩 → 크래시 또는 렌더링 오류
```

### 수정 방법

Dynamic PSO 캐시 히트 `return` 직전에 `active_pso`를 갱신한다.

**변경 위치: GraphicsDevice_DX12.cpp:6839**

```cpp
// 변경 전
if (commandlist.prev_pipeline_hash == pipeline_hash)
{
    return;
}

// 변경 후
if (commandlist.prev_pipeline_hash == pipeline_hash)
{
    commandlist.active_pso = pso;  // ← 추가
    return;
}
```

---

## 2. Static PSO Early Return

### 문제

Static PSO 경로에서 같은 PSO가 이미 바인딩된 경우에도
Root Signature 체크까지 불필요하게 수행된다.

```cpp
if (internal_state->resource != nullptr)  // Static PSO
{
    if (commandlist.active_pso != pso)
    {
        // SetPipelineState, IASetPrimitiveTopology
    }
    // active_pso == pso 여도 여기까지 항상 실행:
    commandlist.prev_pipeline_hash = {};
    commandlist.dirty_pso = false;
    // ← return 없이 아래 root sig 체크로 진행
}
// Root sig 체크 (동일 PSO면 root sig도 동일 → 불필요)
if (commandlist.active_rootsig_graphics != internal_state->rootSignature.Get()) { ... }
commandlist.active_pso = pso;
```

같은 PSO가 이미 바인딩된 상태에서도 매번 `prev_pipeline_hash` 초기화, `dirty_pso` 설정,
Root Signature 포인터 비교를 수행한다. 기능 버그는 아니지만 불필요한 연산이다.

### 수정 방법

이미 같은 PSO가 바인딩된 경우 즉시 리턴한다.

**변경 위치: GraphicsDevice_DX12.cpp:6817 블록**

```cpp
// 변경 전
if (internal_state->resource != nullptr)
{
    if (commandlist.active_pso != pso)
    {
        commandlist.GetGraphicsCommandList()->SetPipelineState(internal_state->resource.Get());
        if (commandlist.prev_pt != internal_state->primitiveTopology) { ... }
    }
    commandlist.prev_pipeline_hash = {};
    commandlist.dirty_pso = false;
}

// 변경 후
if (internal_state->resource != nullptr)
{
    if (commandlist.active_pso == pso)
    {
        return;  // 이미 바인딩됨 → 모든 후속 작업 스킵
    }
    commandlist.GetGraphicsCommandList()->SetPipelineState(internal_state->resource.Get());
    if (commandlist.prev_pt != internal_state->primitiveTopology) { ... }
    commandlist.prev_pipeline_hash = {};
    commandlist.dirty_pso = false;
}
```

`if (active_pso != pso)` 안쪽에서 하던 조건 분기를 뒤집어서,
같은 경우 early return, 다른 경우 바인딩 수행으로 구조를 바꾼다.

---

## 3. Root Descriptor SRV/UAV buffer offset

### 문제

버퍼의 특정 범위(subresource)를 Root Descriptor SRV/UAV로 바인딩할 때
byte offset이 무시된다.

**예시 상황:**

```
GPUBuffer (1MB):
├─ offset=0:    인스턴스 데이터 A (512KB)  ← subresource[0] SRV
└─ offset=512KB: 인스턴스 데이터 B (512KB) ← subresource[1] SRV

Root Descriptor로 subresource[1]을 바인딩하면?
  gpu_address = buffer.GetGPUVirtualAddress()  // 버퍼 시작 주소 (offset=0)
  → 셰이더는 인스턴스 데이터 A를 읽음!  ← 틀린 데이터
```

**현재 코드 (GraphicsDevice_DX12.cpp:2059-2087):**

```cpp
case D3D12_ROOT_PARAMETER_TYPE_SRV:
{
    int subresource = table.SRV_index[param.Descriptor.ShaderRegister]; // 선언만, 미사용!
    D3D12_GPU_VIRTUAL_ADDRESS address = {};
    if (resource.IsValid())
    {
        auto internal_state = to_internal(&resource);
        address = internal_state->gpu_address;  // ← offset 미반영
    }
    graphicscommandlist->SetGraphicsRootShaderResourceView(root_parameter_index, address);
}
```

`subresource` 변수를 선언했지만 실제로 사용하지 않는다.
`SingleDescriptor`에도 `buffer_offset` 필드가 없어서 저장할 곳도 없다.

비교: **CBV는 이미 올바르게 구현됨** (GraphicsDevice_DX12.cpp:2036):
```cpp
case D3D12_ROOT_PARAMETER_TYPE_CBV:
{
    uint64_t offset = table.CBV_offset[param.Descriptor.ShaderRegister]; // 별도 offset 저장소
    address = internal_state->gpu_address + offset;  // ← offset 반영
}
```

CBV는 `table.CBV_offset[]`라는 별도 배열에 offset을 저장하고 있다.
SRV/UAV는 이 구조가 없어서 subresource index만 있고 offset은 없다.

### 수정 방법

**수정 3-1: SingleDescriptor에 buffer_offset 필드 추가**

`GraphicsDevice_DX12.cpp:1159`의 `SingleDescriptor` 구조체:

```cpp
struct SingleDescriptor
{
    std::shared_ptr<GraphicsDevice_DX12::AllocationHandler> allocationhandler;
    D3D12_CPU_DESCRIPTOR_HANDLE handle = {};
    D3D12_DESCRIPTOR_HEAP_TYPE type = {};
    int index = -1;  // bindless
    uint64_t buffer_offset = 0;  // ← 추가: Root Descriptor 바인딩용 byte offset

    bool IsValid() const { return handle.ptr != 0; }
    ...
};
```

**수정 3-2: CreateSubresource(GPUBuffer*)에서 buffer_offset 저장**

`GraphicsDevice_DX12.cpp:5095` (SRV) / `5148` (UAV):

```cpp
// SRV (line 5095)
SingleDescriptor descriptor;
descriptor.init(this, srv_desc, internal_state->resource.Get());
descriptor.buffer_offset = offset;  // ← 추가: 파라미터로 받은 byte offset 저장

// UAV (line 5148)
SingleDescriptor descriptor;
descriptor.init(this, uav_desc, internal_state->resource.Get());
descriptor.buffer_offset = offset;  // ← 추가
```

`CreateSubresource(GPUBuffer*, type, uint64_t offset, ...)` 함수가 `offset` 파라미터를
이미 받고 있어서, 이를 `SingleDescriptor`에 저장하기만 하면 된다.

**수정 3-3: DescriptorBinder::flush에서 SRV/UAV offset 적용**

`GraphicsDevice_DX12.cpp:2059-2117`:

```cpp
// 변경 전: SRV
case D3D12_ROOT_PARAMETER_TYPE_SRV:
{
    int subresource = table.SRV_index[param.Descriptor.ShaderRegister];
    D3D12_GPU_VIRTUAL_ADDRESS address = {};
    if (resource.IsValid())
    {
        auto internal_state = to_internal(&resource);
        address = internal_state->gpu_address;
        // subresource 미사용
    }
    ...
}

// 변경 후: SRV
case D3D12_ROOT_PARAMETER_TYPE_SRV:
{
    int subresource = table.SRV_index[param.Descriptor.ShaderRegister];
    D3D12_GPU_VIRTUAL_ADDRESS address = {};
    if (resource.IsValid())
    {
        auto internal_state = to_internal(&resource);
        address = internal_state->gpu_address;
        if (subresource >= 0)
        {
            address += internal_state->subresources_srv[subresource].buffer_offset;
        }
    }
    ...
}

// 변경 전: UAV
case D3D12_ROOT_PARAMETER_TYPE_UAV:
{
    int subresource = table.UAV_index[param.Descriptor.ShaderRegister];
    D3D12_GPU_VIRTUAL_ADDRESS address = {};
    if (resource.IsValid())
    {
        auto internal_state = to_internal(&resource);
        address = internal_state->gpu_address;
        // subresource 미사용
    }
    ...
}

// 변경 후: UAV
case D3D12_ROOT_PARAMETER_TYPE_UAV:
{
    int subresource = table.UAV_index[param.Descriptor.ShaderRegister];
    D3D12_GPU_VIRTUAL_ADDRESS address = {};
    if (resource.IsValid())
    {
        auto internal_state = to_internal(&resource);
        address = internal_state->gpu_address;
        if (subresource >= 0)
        {
            address += internal_state->subresources_uav[subresource].buffer_offset;
        }
    }
    ...
}
```

---

## 참고: 적용하지 않은 항목

### Dynamic Depth Bias (dx1 #14) — ⏭️ 미적용 유지

런타임에 Depth Bias를 PSO 재생성 없이 변경하는 기능이다.
적용하려면 `PSO_STREAM1`에 `CD3DX12_PIPELINE_STATE_STREAM_FLAGS Flags` 필드를 추가하고,
드로우 시 `ID3D12GraphicsCommandList9::RSSetDepthBias()`를 호출해야 한다.

WickedEngine도 Flags 필드만 추가하고 실제로는 사용하지 않는다.
VizMotive도 현재 Static Depth Bias로 충분하므로 그대로 둔다.

### Root Signature 수명 연장 (dx1 #16) — 변경 불필요

WickedEngine은 `Shader_DX12`가 `rootsig_deserializer`를 소유하고,
`PipelineState_DX12`가 `rootsig_desc` 포인터를 빌리는 구조였다.
셰이더 삭제(핫리로드 등) 시 dangling pointer 발생.

```
WickedEngine (버그):
  Shader_DX12 소유: rootsig_deserializer → rootsig_desc 메모리
  PipelineState_DX12 빌림: rootsig_desc* ← Shader 삭제 시 dangling!

WickedEngine (fix):
  PipelineState_DX12에 shared_ptr<void> rootsig_desc_lifetime_extender 추가
  → Shader 삭제되어도 메모리 유지
```

VizMotive는 `PipelineState_DX12`가 `rootsig_deserializer`를 직접 소유한다:
```cpp
// VizMotive PipelineState_DX12
ComPtr<ID3D12VersionedRootSignatureDeserializer> rootsig_deserializer;  // 직접 소유
const D3D12_VERSIONED_ROOT_SIGNATURE_DESC* rootsig_desc = nullptr;      // 자기 자신 메모리 참조
```
자기 자신이 소유하는 deserializer에서 포인터를 얻으므로 dangling 불가. **변경 불필요.**

---

## 변경 전후 비교

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| Dynamic PSO 캐시 히트 | `return` 시 `active_pso` 갱신 안 됨 | `return` 전에 `active_pso = pso` |
| Static PSO 이미 바인딩됨 | root sig 체크까지 진행 | 즉시 `return` |
| SRV Root Descriptor | `gpu_address` 그대로 (offset 무시) | `+ subresources_srv[i].buffer_offset` |
| UAV Root Descriptor | `gpu_address` 그대로 (offset 무시) | `+ subresources_uav[i].buffer_offset` |
| `SingleDescriptor` | `buffer_offset` 없음 | `uint64_t buffer_offset = 0` 추가 |

---

## 보충 설명

### Descriptor Table vs Root Descriptor — 왜 offset이 다르게 처리되나

> 개념 참고: [part6_pso_engine_implementation.md §6.6](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part6_pso_engine_implementation.md#66-root-descriptor에서-offset이-필요한-이유) / [part3_gpu_communication.md §3.4 Root Descriptor offset 주의사항](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part3_gpu_communication.md#root-descriptor의-byte-offset-주의사항)

```
Descriptor Table SRV:
  CreateShaderResourceView(resource, &srvDesc, cpuHandle)
  srvDesc.Buffer.FirstElement = offset / stride  ← 이미 offset 반영됨
  → GPU는 descriptor에서 offset을 읽음

Root Descriptor SRV:
  SetGraphicsRootShaderResourceView(slot, gpuAddress)
  gpuAddress = resource->GetGPUVirtualAddress()  ← 버퍼 시작 주소
  → offset을 직접 더해야 함
```

Descriptor Table은 SRV 생성 시 `FirstElement` 필드에 offset을 포함해서,
GPU가 descriptor를 읽을 때 자동으로 올바른 위치를 가리킨다.
Root Descriptor는 날(raw) GPU 주소를 전달하므로, `gpu_address + byte_offset`을 직접 계산해서 넘겨야 한다.

### `subresource == -1`일 때

`subresource == -1`이면 버퍼 전체를 가리키는 메인 SRV/UAV를 사용한다는 의미.
이 경우 `internal_state->gpu_address`가 이미 버퍼 시작 주소를 가리키므로 offset 불필요.
코드에서 `if (subresource >= 0)` 조건으로 구분한다.

### CBV는 왜 다른 방식인가

CBV는 `table.CBV_offset[]`이라는 별도 배열에 offset을 저장한다.
SRV/UAV와 달리 subresource index가 없고, `BindConstantBuffer(buffer, slot, offset)` 호출 시 offset을 직접 전달하기 때문이다.
SRV/UAV는 `CreateSubresource()` 결과로 얻은 subresource index를 통해 offset을 관리한다.
