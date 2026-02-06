# 커밋 #16: f0203679 - DX12: root descriptor can use buffer offset

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `f0203679` |
| 날짜 | 2025-11-29 |
| 작성자 | Turánszki János |
| 카테고리 | 기능 / 버그수정 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | Root descriptor 바인딩 시 buffer offset 지원 |

---

## 배경 지식: Root Descriptor vs Descriptor Table

### DX12 리소스 바인딩 방식

```
┌─────────────────────────────────────────────────────────────────┐
│              DX12 Root Signature 바인딩 방식                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Descriptor Table (일반적)                                   │
│  ─────────────────────────────                                  │
│  Root Signature:                                                │
│  ┌─────────────────────────────────────┐                        │
│  │ [0] Descriptor Table → Heap Index  │                        │
│  └─────────────────────────────────────┘                        │
│                    │                                            │
│                    ▼                                            │
│  Descriptor Heap:                                               │
│  ┌─────┬─────┬─────┬─────┬─────┐                                │
│  │ SRV │ SRV │ UAV │ CBV │ ... │                                │
│  └─────┴─────┴─────┴─────┴─────┘                                │
│                                                                 │
│  장점: 많은 리소스를 한 번에 바인딩                             │
│  단점: Heap 관리 필요, 간접 참조 오버헤드                       │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  2. Root Descriptor (직접 바인딩)                               │
│  ─────────────────────────────────                              │
│  Root Signature:                                                │
│  ┌─────────────────────────────────────┐                        │
│  │ [0] Root SRV → GPU Virtual Address │ ← 직접 주소!           │
│  └─────────────────────────────────────┘                        │
│                    │                                            │
│                    ▼                                            │
│  GPU Memory:                                                    │
│  ┌──────────────────────────────────────┐                       │
│  │ Buffer Data                          │                       │
│  └──────────────────────────────────────┘                       │
│                                                                 │
│  장점: Heap 없이 직접 바인딩, 빠름                              │
│  단점: 버퍼만 가능 (텍스처 불가), Root Signature 공간 사용      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Root Descriptor 제약

```cpp
// Root Descriptor는 버퍼의 GPU 가상 주소를 직접 사용
commandList->SetGraphicsRootShaderResourceView(
    rootParameterIndex,
    buffer->GetGPUVirtualAddress()  // 버퍼 시작 주소
);

// 텍스처는 Root Descriptor로 바인딩 불가!
// → 반드시 Descriptor Table 사용
```

---

## 문제: Subresource Offset 미지원

### Subresource란?

버퍼의 일부 영역을 별도 View로 사용:

```
┌─────────────────────────────────────────────────────────────────┐
│              Buffer Subresource                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  하나의 큰 버퍼:                                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ offset=0      │ offset=1024   │ offset=2048   │ ...      │   │
│  │ [Subres 0]    │ [Subres 1]    │ [Subres 2]    │          │   │
│  │ Vertices A    │ Vertices B    │ Vertices C    │          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  CreateSubresource(buffer, subresource=1, offset=1024, ...)     │
│  → Subresource 1은 offset=1024부터 시작                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기존 코드의 문제

```cpp
// 기존 코드 - offset 무시!
void BindResource(SRV srv, int slot, int subresource)
{
    auto internal = to_internal(srv.resource);
    D3D12_GPU_VIRTUAL_ADDRESS address = internal->gpu_address;
    //                                  ^^^^^^^^^^^^^^^^
    //                                  항상 버퍼 시작 주소!
    //                                  subresource의 offset이 반영 안 됨

    commandList->SetGraphicsRootShaderResourceView(slot, address);
}
```

### 문제 시나리오

```
버퍼: [Data A (0~1023)] [Data B (1024~2047)] [Data C (2048~3071)]
                        ↑
                        Subresource 1, offset=1024

쉐이더에서 Subresource 1 바인딩:
- 기대: Data B에 접근
- 실제: Data A에 접근 (offset=0)

→ 잘못된 데이터 읽음!
```

---

## 해결: buffer_offset 멤버 추가

### SingleDescriptor에 offset 저장

```cpp
struct SingleDescriptor
{
    D3D12_CPU_DESCRIPTOR_HANDLE handle = {};
    // ... 기존 멤버들 ...

    uint64_t buffer_offset = 0;  // ✅ 새로 추가!
};
```

### CreateSubresource에서 offset 저장

```cpp
int GraphicsDevice_DX12::CreateSubresource(
    GPUBuffer* buffer,
    SubresourceType type,
    uint64_t offset,  // subresource 시작 offset
    uint64_t size,
    const Format* format,
    const uint32_t* structureStride)
{
    // SRV 생성
    if (type == SubresourceType::SRV)
    {
        SingleDescriptor descriptor;

        // Descriptor 생성 (기존 코드)
        D3D12_SHADER_RESOURCE_VIEW_DESC srv_desc = {};
        srv_desc.Buffer.FirstElement = offset / stride;
        srv_desc.Buffer.NumElements = size / stride;
        // ...

        // ✅ offset 저장 (새로 추가)
        descriptor.buffer_offset = offset;

        internal_state->subresources_srv.push_back(descriptor);
    }
    // UAV도 동일하게 처리
}
```

### 바인딩 시 offset 적용

```cpp
void GraphicsDevice_DX12::BindResource(SRV srv, int slot, int subresource)
{
    auto internal = to_internal(srv.resource);
    D3D12_GPU_VIRTUAL_ADDRESS address = internal->gpu_address;

    // ✅ subresource의 offset 적용 (새로 추가)
    if (subresource >= 0 && subresource < internal->subresources_srv.size())
    {
        address += internal->subresources_srv[subresource].buffer_offset;
    }

    commandList->SetGraphicsRootShaderResourceView(slot, address);
}
```

---

## 다이어그램: 수정 전 vs 수정 후

```
┌─────────────────────────────────────────────────────────────────┐
│                      수정 전 (버그)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Buffer GPU Address: 0x10000                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ [0x10000]      │ [0x10400]     │ [0x10800]     │         │   │
│  │  Data A        │  Data B       │  Data C       │         │   │
│  └──────────────────────────────────────────────────────────┘   │
│        ↑                ↑                                       │
│        │                │                                       │
│        │          Subresource 1 (offset=1024)                   │
│        │                                                        │
│  BindResource(subresource=1):                                   │
│  address = 0x10000  ← 항상 버퍼 시작!                           │
│  → Data A에 잘못 접근                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      수정 후 (정상)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Buffer GPU Address: 0x10000                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ [0x10000]      │ [0x10400]     │ [0x10800]     │         │   │
│  │  Data A        │  Data B       │  Data C       │         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          ↑                                      │
│                          │                                      │
│                    Subresource 1                                │
│                    buffer_offset = 1024                         │
│                                                                 │
│  BindResource(subresource=1):                                   │
│  address = 0x10000 + 1024 = 0x10400  ✅                        │
│  → Data B에 올바르게 접근                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 비유: 책장에서 책 찾기

```
┌─────────────────────────────────────────────────────────────────┐
│                    책장 비유                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  책장 (= 버퍼):                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 칸 1      │ 칸 2      │ 칸 3      │ 칸 4      │        │    │
│  │ 수학책들  │ 과학책들  │ 영어책들  │ 역사책들  │        │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  수정 전:                                                       │
│  "과학책(칸 2) 가져와" → 항상 칸 1에서 가져옴 (수학책)          │
│                                                                 │
│  수정 후:                                                       │
│  "과학책(칸 2) 가져와" → 칸 2로 가서 과학책 가져옴 ✅           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 사용 사례

### 버퍼 서브할당 (Suballocation)

```cpp
// 하나의 큰 버퍼에 여러 데이터 저장
GPUBuffer bigBuffer;
CreateBuffer(&bigBuffer, 1024 * 1024);  // 1MB 버퍼

// 각 데이터를 subresource로 생성
int vertexSub = CreateSubresource(&bigBuffer, SRV, 0, 4096);      // 0~4095
int indexSub = CreateSubresource(&bigBuffer, SRV, 4096, 2048);    // 4096~6143
int constantSub = CreateSubresource(&bigBuffer, SRV, 6144, 256);  // 6144~6399

// Root Descriptor로 바인딩
BindResource(bigBuffer, slot, indexSub);  // offset=4096이 적용됨
```

### 장점

```
1. 메모리 효율
   - 작은 버퍼 여러 개 → 큰 버퍼 하나로 통합
   - 할당 오버헤드 감소

2. 바인딩 효율
   - 같은 버퍼의 다른 영역을 빠르게 전환
   - Root Descriptor 사용 가능 (Descriptor Table 불필요)
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-28

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.cpp | `SingleDescriptor`에 `buffer_offset` 멤버 추가 |
| GraphicsDevice_DX12.cpp | SRV root descriptor 바인딩에 offset 적용 |
| GraphicsDevice_DX12.cpp | UAV root descriptor 바인딩에 offset 적용 |
| GraphicsDevice_DX12.cpp | SRV subresource 생성 시 `buffer_offset` 설정 |

### 코드 위치

```cpp
// SingleDescriptor 구조체
struct SingleDescriptor
{
    // ...
    uint64_t buffer_offset = 0;
};

// 바인딩 시 offset 적용
if (subresource >= 0 && subresource < (int)internal_state->subresources_srv.size())
{
    address += internal_state->subresources_srv[subresource].buffer_offset;
}
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 문제 | Root Descriptor 바인딩 시 subresource offset이 적용 안 됨 |
| 증상 | 버퍼의 subresource를 바인딩해도 항상 버퍼 시작 주소 사용 |
| 해결 | `SingleDescriptor`에 `buffer_offset` 저장, 바인딩 시 적용 |
| 적용 대상 | SRV, UAV의 root descriptor 바인딩 |
| VizMotive | ✅ 적용 완료 |

### 핵심 교훈

> **Subresource는 offset 정보를 포함**
>
> Subresource를 생성할 때 지정한 offset은
> 바인딩 시에도 반영되어야 함.
>
> Descriptor Table은 View가 offset을 포함하지만,
> Root Descriptor는 직접 주소를 사용하므로
> 명시적으로 offset을 더해줘야 함.
