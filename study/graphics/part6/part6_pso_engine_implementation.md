# Part 6: 엔진에서의 PSO 관리

Part 3에서 PSO와 Root Signature의 기본 개념을 배웠다.
이 문서는 **실제 게임 엔진 코드에서 PSO가 어떻게 관리되는지**를 다룬다.

---

## 6.1 Static PSO vs Dynamic PSO

실제 엔진은 두 가지 방식으로 PSO를 관리한다.

### Static PSO

```cpp
// 엔진 시작 시 (또는 셰이더 컴파일 시)
ID3D12PipelineState* compiledPSO;
device->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&compiledPSO));

// PipelineState_DX12 내부:
//   resource = compiledPSO  ← nullptr이 아님
```

- 런타임 이전에 미리 컴파일된 PSO
- `resource != nullptr`이면 Static PSO
- 항상 동일한 RTV 포맷, 샘플 수, 렌더 패스 구성 가정

### Dynamic PSO

```cpp
// PipelineState_DX12 내부:
//   resource = nullptr  ← Static PSO가 아님

// 드로우콜 직전에:
void GraphicsDevice::BindPipelineState(const PipelineState* pso, CommandList cmd)
{
    auto internal_state = to_internal(pso);
    if (internal_state->resource == nullptr)  // Dynamic PSO 경로
    {
        // 현재 렌더 패스의 RTV 포맷, MSAA 수준 등을 조합해
        // 실시간으로 PSO를 생성하거나 캐시에서 꺼냄
        auto actual_pso = GetOrCreateDynamicPSO(pso, renderpass_info);
        commandList->SetPipelineState(actual_pso.Get());
    }
}
```

- 런타임에 현재 렌더 패스 구성에 맞게 PSO를 생성
- 같은 `PipelineState*`여도 렌더 패스에 따라 다른 GPU PSO 객체가 생성됨
- 렌더 패스마다 RTV 포맷, MSAA가 다를 수 있어서 필요

### 왜 두 방식이 공존하나?

```
Static PSO:
  장점: 사전 컴파일 → 런타임에 빠름
  단점: 모든 렌더 패스 조합을 미리 알아야 함 (어려움)

Dynamic PSO:
  장점: 어떤 렌더 패스에도 대응
  단점: 처음 사용 시 생성 비용 (이후 캐시 히트 시 빠름)
```

---

## 6.2 Pipeline Hash와 동일 PSO 캐시

Dynamic PSO는 `(PSO 포인터, 렌더패스 포맷)` 조합으로 실제 GPU PSO 객체를 캐시한다.
커맨드리스트 레벨에서도 "직전 드로우와 같은 조합인가?"를 추적해 중복 바인딩을 방지한다.

```cpp
struct PipelineHash
{
    const PipelineState* pso;       // PSO 포인터
    uint32_t renderpass_hash;       // 현재 렌더 패스 포맷 해시
};

// CommandList_DX12 멤버:
PipelineHash prev_pipeline_hash;
```

```
드로우콜 A: PSO=X, renderpass=MRT_A  →  hash(X, MRT_A) 생성 및 저장
드로우콜 B: PSO=X, renderpass=MRT_A  →  hash 동일 → return (스킵)
드로우콜 C: PSO=X, renderpass=MRT_B  →  hash 다름  → 새 PSO 바인딩
```

Static PSO는 별도의 hash 체크 없이 포인터 비교(`active_pso == pso`)로 중복을 방지한다.

---

## 6.3 active_pso — 현재 바인딩 상태 추적

```cpp
// CommandList_DX12 멤버:
const PipelineState* active_pso;   // 현재 바인딩된 PSO
const Shader* active_cs;           // 현재 바인딩된 Compute Shader
```

`active_pso`는 단순한 최적화 변수가 아니다.
**DescriptorBinder::flush()에서 Root Signature Optimizer를 찾는 데 사용된다.**

```cpp
void DescriptorBinder::flush(bool graphics, CommandList cmd)
{
    auto& commandlist = device->GetCommandList(cmd);

    // active_pso로부터 rootsig_optimizer를 얻음
    auto pso_internal = graphics
        ? to_internal(commandlist.active_pso)    // ← 여기
        : to_internal(commandlist.active_cs);

    auto& rootsig_optimizer = pso_internal->rootsig_optimizer;  // ← 이것을 사용
    // rootsig_optimizer로 어느 root parameter에 무엇을 바인딩할지 결정
}
```

`active_pso`가 실제 바인딩된 PSO와 다르면 → **잘못된 rootsig_optimizer** → 잘못된 슬롯에 descriptor → 크래시 또는 잘못된 렌더링 결과.

---

## 6.4 Root Sig Optimizer

`rootsig_optimizer`는 Root Signature 구조를 파싱해서,
"어느 셰이더 레지스터(b0, t1, u2...)가 어느 root parameter index에 있는지"를 미리 계산해 저장한 구조다.

```
Root Signature:
  Slot 0: Root Constants (b0)
  Slot 1: Root CBV (b1)
  Slot 2: Root SRV (t0)
  Slot 3: Descriptor Table (t1~t10)

rootsig_optimizer:
  b0 → slot 0, type = Constants
  b1 → slot 1, type = CBV
  t0 → slot 2, type = SRV (Root Descriptor)
  t1~t10 → slot 3, type = Table
```

flush()에서 descriptor를 바인딩할 때:
```cpp
// SetGraphicsRootDescriptorTable(3, handle)  // t1은 slot 3의 Table
// SetGraphicsRootShaderResourceView(2, addr) // t0은 slot 2의 Root Descriptor
```

PSO가 바뀌면 Root Signature도 바뀔 수 있고,
같은 레지스터(t0)가 다른 slot에 있거나 다른 방식(Table vs Root Descriptor)으로 바인딩될 수 있다.
`active_pso`가 틀리면 이 매핑이 완전히 어긋난다.

---

## 6.5 DescriptorBinder 패턴

엔진은 드로우콜마다 `SetGraphicsRootXxx`를 직접 호출하지 않는다.
대신 **기록(Record) → 일괄 적용(Flush)** 패턴을 사용한다.

### 기록 단계

```cpp
// 엔진 코드 (예: renderer.cpp)
device->BindResource(&texture, 0, cmd);      // t0에 텍스처 등록
device->BindConstantBuffer(&cb, 1, cmd, 0);  // b1에 상수 버퍼 등록
device->DrawInstanced(36, 1, 0, 0, cmd);     // 드로우콜 호출
```

내부적으로 `DescriptorBinder`의 테이블에 기록만 한다:
```cpp
table.SRV[0] = &texture;     // "t0에 texture 예약"
table.CBV[1] = &cb;          // "b1에 cb 예약"
table.CBV_offset[1] = 0;     // "b1의 byte offset = 0"
```

### Flush 단계

드로우콜 직전에 `flush()`가 호출되어 실제 바인딩이 이루어진다:

```cpp
void DescriptorBinder::flush(bool graphics, CommandList cmd)
{
    // active_pso로부터 rootsig_optimizer 획득
    auto& optimizer = pso_internal->rootsig_optimizer;

    // Root Signature의 모든 parameter를 순회
    for (auto& param : optimizer.root_parameters)
    {
        switch (param.ParameterType)
        {
        case D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS:
            commandList->SetGraphicsRoot32BitConstants(...);
            break;

        case D3D12_ROOT_PARAMETER_TYPE_CBV:
        {
            uint64_t offset = table.CBV_offset[param.Descriptor.ShaderRegister];
            D3D12_GPU_VIRTUAL_ADDRESS address =
                internal_state->gpu_address + offset;  // ← offset 직접 더함
            commandList->SetGraphicsRootConstantBufferView(slot, address);
            break;
        }

        case D3D12_ROOT_PARAMETER_TYPE_SRV:
        {
            int subresource = table.SRV_index[param.Descriptor.ShaderRegister];
            D3D12_GPU_VIRTUAL_ADDRESS address = internal_state->gpu_address;
            if (subresource >= 0)
            {
                address += internal_state->subresources_srv[subresource].buffer_offset;
                // ↑ subresource의 byte offset을 더해야 정확한 위치를 가리킴
            }
            commandList->SetGraphicsRootShaderResourceView(slot, address);
            break;
        }

        case D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE:
            commandList->SetGraphicsRootDescriptorTable(slot, gpuHandle);
            break;
        }
    }
}
```

---

## 6.6 Root Descriptor에서 offset이 필요한 이유

Part 3 §3.4에서 설명했듯이, Root Descriptor와 Descriptor Table은 offset 처리 방식이 다르다.

```
버퍼 레이아웃 예시 (1MB GPUBuffer):
┌──────────────────────────────────────────┐
│  offset=0:     인스턴스 데이터 A (512KB)  │  ← subresource[0] SRV
│  offset=512KB: 인스턴스 데이터 B (512KB)  │  ← subresource[1] SRV
└──────────────────────────────────────────┘

Descriptor Table SRV로 subresource[1] 바인딩:
  CreateShaderResourceView(buffer, srvDesc, cpuHandle)
  srvDesc.Buffer.FirstElement = 512KB / sizeof(element)  ← descriptor 안에 offset 있음
  → GPU가 descriptor에서 자동으로 올바른 위치 참조

Root Descriptor SRV로 subresource[1] 바인딩:
  buffer->GetGPUVirtualAddress()  → 버퍼 시작 주소 (0번지)
  ← offset 정보가 없음!
  → gpu_address + 512KB 를 직접 계산해서 전달해야 함
```

**핵심 차이:**

| 방식 | offset 정보 위치 | offset 반영 주체 |
|------|-----------------|-----------------|
| Descriptor Table | SRV descriptor의 `FirstElement` | GPU (자동) |
| Root Descriptor | 없음 — 직접 더해야 함 | CPU (호출자) |

이 차이 때문에 Root Descriptor 경로에서 `buffer_offset`을 `gpu_address`에 더하는 코드가 필요하다.

---

## 6.7 드로우콜 전체 흐름

```
CPU (엔진 코드)                        GPU
─────────────────────────────────────────────────────

BindPipelineState(pso)
  → if Static PSO:
      if active_pso == pso → return  [최적화]
      SetPipelineState(pso->resource)
      active_pso = pso
  → if Dynamic PSO:
      hash = (pso, renderpass_hash)
      if prev_hash == hash:
          active_pso = pso → return  [최적화]
      actual_pso = cache[hash] 또는 새로 생성
      SetPipelineState(actual_pso)
      active_pso = pso

BindResource(&texture, t0, cmd)
  → table.SRV[0] = &texture  [기록만]

BindConstantBuffer(&cb, b1, cmd, offset)
  → table.CBV[1] = &cb       [기록만]
  → table.CBV_offset[1] = offset

DrawInstanced(...)
  → flush() 호출 (내부):
      optimizer = active_pso->rootsig_optimizer
      for each root parameter:
        CBV b1:  gpu_address + CBV_offset[1]  → SetRootCBV
        SRV t0:  gpu_address + buffer_offset  → SetRootSRV
                                                        ↓
                                                  (GPU 실행)
                                                  커맨드리스트 기록됨
```

---

## 6.8 VizMotive에서의 구조 요약

```
PipelineState
├── resource (ID3D12PipelineState*)  — nullptr이면 Dynamic
├── rootSignature (ID3D12RootSignature*)
├── rootsig_deserializer  — rootsig_desc 메모리 소유
├── rootsig_desc  — 자신의 deserializer에서 얻은 포인터
└── rootsig_optimizer  — flush()에서 사용

CommandList_DX12
├── active_pso      — 현재 바인딩된 PSO (flush()에서 optimizer 접근용)
├── active_cs       — 현재 바인딩된 Compute Shader
├── prev_pipeline_hash  — Dynamic PSO 중복 바인딩 방지
└── binder          — DescriptorBinder (resource table + flush)

SingleDescriptor (버퍼 SRV/UAV용)
├── handle          — CPU descriptor heap 핸들
├── index           — bindless 인덱스
└── buffer_offset   — Root Descriptor 바인딩 시 gpu_address에 더할 byte offset
```

---

## 관련 문서

- [Part 3 §3.3 PSO 기본](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part3_gpu_communication.md#33-pso-pipeline-state-object)
- [Part 3 §3.4 Root Signature 기본](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/study/graphics/part3_gpu_communication.md#34-루트-시그니처-root-signature)
- [apply_pso_rootsig.md — VizMotive 코드 수정 기록](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/apply_pso_rootsig.md)
