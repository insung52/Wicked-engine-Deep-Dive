# 커밋 #18: 7ecbc25a - GFX allocators update

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `7ecbc25a` |
| 날짜 | 2025-12-06 |
| 작성자 | Turánszki János |
| 카테고리 | 업데이트 / 최적화 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| D3D12MemAlloc.h | 버전 2.1.0 → 3.0.1 업데이트 |
| D3D12MemAlloc.cpp | 버전 2.1.0 → 3.0.1 업데이트 |
| wiGraphicsDevice_DX12.cpp | Windows 11 최적화 플래그 추가 |

---

## 배경 지식: D3D12 Memory Allocator (D3D12MA)

### D3D12MA란?

```
┌─────────────────────────────────────────────────────────────────┐
│              D3D12 Memory Allocator                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  개발: AMD GPUOpen                                              │
│  목적: DX12 GPU 메모리 관리 단순화                              │
│                                                                 │
│  DX12 기본 메모리 관리:                                         │
│  ─────────────────────                                          │
│  - ID3D12Device::CreateCommittedResource() - 개별 할당          │
│  - ID3D12Device::CreatePlacedResource() - Heap에 배치           │
│  - ID3D12Heap 직접 관리 필요                                    │
│  → 복잡하고 단편화 발생하기 쉬움                                │
│                                                                 │
│  D3D12MA 사용:                                                  │
│  ─────────────                                                  │
│  - D3D12MA::Allocator가 메모리 풀링 및 서브할당 처리            │
│  - 단편화 최소화                                                │
│  - 간단한 API                                                   │
│                                                                 │
│  유사 라이브러리: VMA (Vulkan Memory Allocator)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### D3D12MA 사용 예시

```cpp
// D3D12MA 없이 (복잡)
D3D12_HEAP_PROPERTIES heapProps = {};
heapProps.Type = D3D12_HEAP_TYPE_DEFAULT;
D3D12_RESOURCE_DESC resourceDesc = {};
// ... 많은 설정 ...
device->CreateCommittedResource(&heapProps, ..., &resource);

// D3D12MA 사용 (간단)
D3D12MA::ALLOCATION_DESC allocDesc = {};
allocDesc.HeapType = D3D12_HEAP_TYPE_DEFAULT;
allocator->CreateResource(&allocDesc, &resourceDesc, ..., &allocation, &resource);
```

---

## 버전 변경

### 버전 정보

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 버전 | 2.1.0-development | 3.0.1 |
| 날짜 | 2023-07-05 | 2025-05-08 |
| 기간 | - | 약 2년치 업데이트 |

### 3.0.1 주요 변경사항

```
┌─────────────────────────────────────────────────────────────────┐
│              D3D12MA 3.0.1 주요 개선                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Windows 11 최적화                                           │
│     ─────────────────                                           │
│     - 작은 committed buffer 성능 문제 해결 플래그               │
│     - ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED        │
│                                                                 │
│  2. GPU Upload Heap 지원 개선                                   │
│     ─────────────────────────                                   │
│     - D3D12_HEAP_TYPE_GPU_UPLOAD 지원 (Windows 11)              │
│     - CPU에서 쓰고 GPU에서 직접 읽기                            │
│                                                                 │
│  3. Residency Priority 지원                                     │
│     ─────────────────────────                                   │
│     - 메모리 우선순위 설정 가능                                 │
│     - 메모리 부족 시 어떤 리소스를 먼저 evict할지 힌트          │
│                                                                 │
│  4. 버그 수정                                                   │
│     ─────────────                                               │
│     - 2년치 버그 수정                                           │
│     - 메모리 누수, 크래시 등 안정성 개선                        │
│                                                                 │
│  5. 새로운 API                                                  │
│     ───────────                                                 │
│     - 더 나은 통계 및 디버깅 기능                               │
│     - 커스텀 풀 옵션 확장                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Windows 11 최적화 플래그

### 문제: 작은 Committed Buffer 성능 저하

```
┌─────────────────────────────────────────────────────────────────┐
│              Windows 11 작은 버퍼 문제                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Windows 10:                                                    │
│  ───────────                                                    │
│  작은 버퍼 (< 64KB)를 CreateCommittedResource()로 생성          │
│  → 빠름 (드라이버 최적화)                                       │
│                                                                 │
│  Windows 11:                                                    │
│  ───────────                                                    │
│  작은 버퍼를 CreateCommittedResource()로 생성                   │
│  → 느림! (드라이버 동작 변경)                                   │
│                                                                 │
│  원인:                                                          │
│  - Windows 11에서 작은 committed resource 처리 방식 변경        │
│  - 내부적으로 더 많은 오버헤드 발생                             │
│                                                                 │
│  해결:                                                          │
│  - 작은 버퍼는 placed resource로 생성 (큰 Heap에 배치)          │
│  - ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED 플래그    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 코드 변경

```cpp
// wiGraphicsDevice_DX12.cpp - CreateDevice 또는 초기화 부분

D3D12MA::ALLOCATOR_DESC allocatorDesc = {};
allocatorDesc.pDevice = device;
allocatorDesc.pAdapter = adapter;

// ✅ Windows 11 최적화 플래그 추가
allocatorDesc.Flags |= D3D12MA::ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED;

D3D12MA::CreateAllocator(&allocatorDesc, &allocator);
```

### 플래그 동작

```
┌─────────────────────────────────────────────────────────────────┐
│    ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  플래그 OFF (기본값):                                           │
│  ──────────────────                                             │
│  작은 버퍼 (< 64KB) → CreateCommittedResource() 사용            │
│  → Windows 10에서 빠름, Windows 11에서 느림                     │
│                                                                 │
│  플래그 ON:                                                     │
│  ─────────                                                      │
│  작은 버퍼 → CreatePlacedResource() 사용 (Heap에 배치)          │
│  → Windows 10, 11 모두에서 일관된 성능                          │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Large Heap (D3D12MA 관리)                               │   │
│  │  ┌──────┬──────┬──────┬──────┬──────────────────────┐    │   │
│  │  │Small │Small │Small │Small │      Free Space      │    │   │
│  │  │Buf 1 │Buf 2 │Buf 3 │Buf 4 │                      │    │   │
│  │  └──────┴──────┴──────┴──────┴──────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  장점: 할당 오버헤드 감소, 메모리 단편화 감소                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Committed vs Placed Resource

### 차이점

```
┌─────────────────────────────────────────────────────────────────┐
│              Committed vs Placed Resource                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Committed Resource:                                            │
│  ──────────────────                                             │
│  device->CreateCommittedResource(...)                           │
│                                                                 │
│  - 리소스마다 개별 Heap 생성                                    │
│  - 간단하지만 오버헤드 있음                                     │
│  - 메모리 단편화 발생 가능                                      │
│                                                                 │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                               │
│  │Heap1│ │Heap2│ │Heap3│ │Heap4│  ← 각각 별도 Heap              │
│  │Res1 │ │Res2 │ │Res3 │ │Res4 │                               │
│  └─────┘ └─────┘ └─────┘ └─────┘                               │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Placed Resource:                                               │
│  ─────────────────                                              │
│  device->CreateHeap(...)                                        │
│  device->CreatePlacedResource(heap, offset, ...)                │
│                                                                 │
│  - 큰 Heap을 미리 생성                                          │
│  - 리소스를 Heap 내 특정 위치에 배치                            │
│  - 서브할당으로 효율적                                          │
│                                                                 │
│  ┌─────────────────────────────────────────────┐                │
│  │ Large Heap                                  │                │
│  │ ┌─────┬─────┬─────┬─────┬─────────────────┐ │                │
│  │ │Res1 │Res2 │Res3 │Res4 │    Free         │ │                │
│  │ └─────┴─────┴─────┴─────┴─────────────────┘ │                │
│  └─────────────────────────────────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 업그레이드 시 주의사항

### API 호환성

```cpp
// D3D12MA 3.x에서 일부 API 변경됨
// 대부분 하위 호환되지만 확인 필요

// 2.x
allocator->GetBudget(&budget);

// 3.x (변경된 API)
allocator->GetBudget(&budget, nullptr);  // 추가 파라미터
```

### 파일 교체

```
D3D12MemAlloc.h   ← 전체 교체
D3D12MemAlloc.cpp ← 전체 교체

라이브러리 파일이므로 수정 없이 전체 교체 권장
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| D3D12MemAlloc.h | 버전 2.1.0 → 3.0.1 전체 교체 |
| D3D12MemAlloc.cpp | 버전 2.1.0 → 3.0.1 전체 교체 |
| GraphicsDevice_DX12.cpp | `ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED` 플래그 추가 |

### 코드 위치

```cpp
// GraphicsDevice_DX12.cpp - Allocator 초기화 부분
D3D12MA::ALLOCATOR_DESC allocatorDesc = {};
allocatorDesc.pDevice = device.Get();
allocatorDesc.pAdapter = dxgiAdapter.Get();
allocatorDesc.Flags |= D3D12MA::ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED;

hr = D3D12MA::CreateAllocator(&allocatorDesc, &allocationhandler->allocator);
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 변경 1 | D3D12MemAlloc 버전 업데이트 (2.1.0 → 3.0.1) |
| 변경 2 | Windows 11 최적화 플래그 추가 |
| 효과 | 2년치 버그 수정, Windows 11 성능 개선 |
| VizMotive | ✅ 적용 완료 |

### 핵심 교훈

> **외부 라이브러리 정기 업데이트**
>
> D3D12MemAlloc 같은 GPU 메모리 관리 라이브러리는
> OS/드라이버 변경에 따른 최적화가 포함되므로
> 정기적으로 업데이트하는 것이 좋음.

> **플랫폼별 최적화 플래그 확인**
>
> Windows 11에서 동작이 변경된 경우가 있으므로
> 라이브러리 릴리스 노트에서 플랫폼별 최적화 확인.
> `ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED`처럼
> 특정 플랫폼 문제를 해결하는 플래그 적용.
