# 커밋 #16: df69a706 - DX12 Root Signature Desc Lifetime Extender

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `df69a706` |
| 날짜 | 2025-06-14 |
| 작성자 | Turánszki János |
| 카테고리 | 안정성 / 메모리 안전 |
| 우선순위 | 높음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | `rootsig_desc_lifetime_extender` 필드 추가 및 설정 |

---

## 배경 지식: DX12 Root Signature

### Root Signature란?

**Root Signature**는 셰이더가 접근할 수 있는 리소스(버퍼, 텍스처, 샘플러 등)의 "레이아웃"을 정의합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Root Signature                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "이 셰이더는 다음 리소스들을 사용합니다"                        │
│                                                                 │
│  ┌──────────────┬──────────────┬──────────────┬────────────┐   │
│  │ Root Param 0 │ Root Param 1 │ Root Param 2 │ Root Param │   │
│  │ CBV (b0)     │ SRV Table    │ UAV (u0)     │ Sampler    │   │
│  │              │ (t0-t7)      │              │ (s0)       │   │
│  └──────────────┴──────────────┴──────────────┴────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Root Signature 생성 과정

```
┌──────────────────────────────────────────────────────────────────┐
│                   셰이더 컴파일 & Root Signature                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 셰이더 컴파일 (.hlsl → .cso 바이트코드)                      │
│     └─ 바이트코드 안에 Root Signature 정보 포함                  │
│                                                                  │
│  2. Root Signature 역직렬화                                      │
│     ┌─────────────────────────────────────────┐                  │
│     │ ID3D12VersionedRootSignatureDeserializer│                  │
│     │                                         │                  │
│     │  바이트코드 → D3D12_VERSIONED_ROOT_     │                  │
│     │               SIGNATURE_DESC 구조체     │                  │
│     └─────────────────────────────────────────┘                  │
│                                                                  │
│  3. Root Signature 객체 생성                                     │
│     device->CreateRootSignature(바이트코드)                      │
│     → ID3D12RootSignature*                                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 핵심 포인트: rootsig_desc 포인터

```cpp
// Shader 내부 구조체
struct Shader_Internal
{
    ComPtr<ID3D12VersionedRootSignatureDeserializer> rootsig_deserializer;
    const D3D12_VERSIONED_ROOT_SIGNATURE_DESC* rootsig_desc = nullptr;  // ← 포인터!
    ComPtr<ID3D12RootSignature> rootSignature;
    // ...
};
```

**중요**: `rootsig_desc`는 `rootsig_deserializer`가 **소유한 메모리**를 가리키는 **raw 포인터**입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        메모리 소유권                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  rootsig_deserializer ─────────────────┐                        │
│  (ComPtr, 메모리 소유)                  │                        │
│                                        ▼                        │
│                               ┌────────────────────┐            │
│                               │ Root Sig 메모리    │            │
│                               │ (파라미터 정보 등)  │            │
│                               └────────────────────┘            │
│                                        ▲                        │
│  rootsig_desc ─────────────────────────┘                        │
│  (raw pointer, 메모리 미소유)                                    │
│                                                                 │
│  deserializer가 해제되면 → rootsig_desc는 Dangling Pointer!     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 배경 지식: PSO와 Shader의 관계

### PSO (Pipeline State Object)

PSO는 여러 셰이더를 조합하여 만든 렌더링 파이프라인입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    PSO (Pipeline State Object)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │ Vertex    │  │ Hull      │  │ Domain    │  │ Geometry  │    │
│  │ Shader    │  │ Shader    │  │ Shader    │  │ Shader    │    │
│  │  (VS)     │  │  (HS)     │  │  (DS)     │  │  (GS)     │    │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘    │
│        │              │              │              │           │
│        └──────────────┴──────────────┴──────────────┘           │
│                              │                                  │
│                              ▼                                  │
│                   ┌───────────────────┐                         │
│                   │   Root Signature   │  ← 첫 번째 유효한      │
│                   │   (셰이더에서 복사) │     셰이더에서 가져옴   │
│                   └───────────────────┘                         │
│                                                                 │
│  + Rasterizer State                                             │
│  + Blend State                                                  │
│  + Depth/Stencil State                                          │
│  + ...                                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### PSO 생성 시 Root Signature 복사

```cpp
// CreatePipelineState() 내부
if (pso->desc.vs != nullptr)
{
    auto shader_internal = to_internal(pso->desc.vs);

    if (internal_state->rootSignature == nullptr)
    {
        // 셰이더에서 root signature 복사
        internal_state->rootSignature = shader_internal->rootSignature;  // ComPtr 복사 (OK)
        internal_state->rootsig_desc = shader_internal->rootsig_desc;    // 포인터 복사 (위험!)
    }
}
```

---

## 문제 상황: Dangling Pointer

### 시나리오

```
┌─────────────────────────────────────────────────────────────────┐
│                    Dangling Pointer 발생 과정                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 셰이더 생성                                                 │
│     ┌────────────────────────────────────────────┐              │
│     │ Shader_Internal                            │              │
│     │  rootsig_deserializer ──→ [메모리 블록]    │              │
│     │  rootsig_desc ───────────→ ↑              │              │
│     └────────────────────────────────────────────┘              │
│                                                                 │
│  2. PSO 생성 (셰이더 참조)                                      │
│     ┌────────────────────────────────────────────┐              │
│     │ PipelineState_DX12                         │              │
│     │  rootsig_desc ───────────→ [메모리 블록]   │  ← 같은 메모리│
│     └────────────────────────────────────────────┘              │
│                                                                 │
│  3. 셰이더 삭제 (hotreload, 메모리 정리 등)                     │
│     ┌────────────────────────────────────────────┐              │
│     │ Shader_Internal 해제됨                      │              │
│     │  rootsig_deserializer 해제 → [메모리 해제] │              │
│     └────────────────────────────────────────────┘              │
│                                                                 │
│  4. PSO에서 rootsig_desc 접근                                   │
│     ┌────────────────────────────────────────────┐              │
│     │ PipelineState_DX12                         │              │
│     │  rootsig_desc ───────────→ [???????????]   │  ← 해제된 메모리!
│     └────────────────────────────────────────────┘              │
│                                                                 │
│     💥 크래시 또는 Undefined Behavior!                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 언제 발생하는가?

1. **셰이더 핫리로드**: 개발 중 셰이더 수정 후 다시 로드
2. **리소스 정리**: 씬 언로드 시 셰이더 먼저 삭제, PSO 나중에 삭제
3. **메모리 부족**: 오래된 셰이더 캐시 정리

### 증상

```cpp
// PSO 바인딩 시 rootsig_desc 접근
void RefreshRootDescriptors()
{
    // optimizer가 rootsig_desc를 참조
    const D3D12_ROOT_PARAMETER1& param =
        pso_internal->rootsig_desc->Desc_1_1.pParameters[index];  // 💥 크래시!
}
```

- 랜덤한 크래시
- Debug Layer 경고 없음 (이미 해제된 메모리)
- 재현이 어려움 (타이밍 의존)

---

## 해결: Lifetime Extender

### 해결 원리

셰이더의 internal_state를 shared_ptr로 보관하여, PSO가 살아있는 동안 셰이더 메모리도 유지합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   Lifetime Extender 적용 후                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Shader_Internal (shared_ptr로 관리)                      │   │
│  │   rootsig_deserializer ──→ [메모리 블록]                 │   │
│  │   rootsig_desc ──────────→ ↑                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                    ▲                         ▲                  │
│                    │                         │                  │
│          ┌─────────┴─────────┐    ┌──────────┴──────────┐      │
│          │ Shader 객체        │    │ PSO 객체            │      │
│          │ (internal_state)  │    │ (lifetime_extender) │      │
│          └───────────────────┘    └─────────────────────┘      │
│                                                                 │
│  두 객체 모두 shared_ptr로 Shader_Internal을 참조               │
│  → 둘 다 해제될 때까지 메모리 유지!                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 코드 변경

**PipelineState_DX12에 필드 추가**:

```cpp
struct PipelineState_DX12
{
    // 기존 필드들
    ComPtr<ID3D12RootSignature> rootSignature;
    const D3D12_VERSIONED_ROOT_SIGNATURE_DESC* rootsig_desc = nullptr;

    // 새로 추가된 필드
    std::shared_ptr<void> rootsig_desc_lifetime_extender;  // ← 핵심!

    RootSignatureOptimizer rootsig_optimizer;
    // ...
};
```

**PSO 생성 시 lifetime extender 설정**:

```cpp
if (pso->desc.vs != nullptr)
{
    auto shader_internal = to_internal(pso->desc.vs);

    if (internal_state->rootSignature == nullptr)
    {
        internal_state->rootSignature = shader_internal->rootSignature;
        internal_state->rootsig_desc = shader_internal->rootsig_desc;

        // 셰이더의 internal_state를 shared_ptr로 보관
        internal_state->rootsig_desc_lifetime_extender = pso->desc.vs->internal_state;

        stream.stream1.ROOTSIG = internal_state->rootSignature.Get();
    }
}
```

### shared_ptr<void> 사용 이유

```cpp
// 왜 void인가?
std::shared_ptr<void> rootsig_desc_lifetime_extender;

// 이유:
// 1. 타입에 무관하게 어떤 셰이더든 보관 가능
// 2. VS, HS, DS, GS, PS, MS, AS 중 어느 것이든 가능
// 3. 실제 타입을 알 필요 없음 - 그냥 살려두기만 하면 됨
// 4. shared_ptr<void>는 원래 deleter를 기억함 → 올바른 소멸자 호출
```

---

## 비유: 도서관과 복사본

```
┌─────────────────────────────────────────────────────────────────┐
│                    도서관 비유                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  변경 전:                                                       │
│  ─────────                                                      │
│  도서관(Shader)에서 책 목차(rootsig_desc)를 메모해 감           │
│  "목차는 3층 A섹션 5번 선반에 있어"                              │
│                                                                 │
│  → 도서관이 폐쇄되면 목차 위치가 무의미해짐                      │
│  → 그 위치에 가면 다른 건물이 들어서 있을 수도 있음              │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  변경 후:                                                       │
│  ─────────                                                      │
│  도서관 자체를 빌림 (lifetime_extender)                         │
│  "이 도서관은 내가 다 볼 때까지 폐쇄하지 마세요"                 │
│                                                                 │
│  → 내가 다 볼 때까지(PSO 해제) 도서관(Shader) 유지               │
│  → 목차 위치가 항상 유효                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-27

### 적용 내용

**1. PipelineState_DX12 구조체** (line 1425):

```cpp
struct PipelineState_DX12
{
    // ...
    ComPtr<ID3D12VersionedRootSignatureDeserializer> rootsig_deserializer;
    const D3D12_VERSIONED_ROOT_SIGNATURE_DESC* rootsig_desc = nullptr;
    vz::allocator::shared_ptr<void> rootsig_desc_lifetime_extender;  // 추가됨
    RootSignatureOptimizer rootsig_optimizer;
    // ...
};
```

**2. 각 셰이더 타입별 lifetime extender 설정**:

| 셰이더 | 라인 | 코드 |
|--------|------|------|
| VS | 4116 | `internal_state->rootsig_desc_lifetime_extender = pso->desc.vs->internal_state;` |
| HS | 4128 | `internal_state->rootsig_desc_lifetime_extender = pso->desc.hs->internal_state;` |
| DS | 4140 | `internal_state->rootsig_desc_lifetime_extender = pso->desc.ds->internal_state;` |
| GS | 4152 | `internal_state->rootsig_desc_lifetime_extender = pso->desc.gs->internal_state;` |
| PS | 4164 | `internal_state->rootsig_desc_lifetime_extender = pso->desc.ps->internal_state;` |
| MS | 4177 | `internal_state->rootsig_desc_lifetime_extender = pso->desc.ms->internal_state;` |
| AS | 4189 | `internal_state->rootsig_desc_lifetime_extender = pso->desc.as->internal_state;` |

---

## 수정 효과

### 해결된 문제

| 문제 | 해결 |
|------|------|
| 셰이더 핫리로드 시 크래시 | ✅ 셰이더 삭제해도 PSO가 참조 유지 |
| 리소스 정리 순서 의존성 | ✅ 순서 무관하게 안전 |
| 랜덤 크래시 | ✅ 메모리 항상 유효 |
| 디버깅 어려움 | ✅ 문제 자체가 발생 안 함 |

### 성능 영향

- **메모리**: 셰이더가 조금 더 오래 살아있을 수 있음 (무시 가능)
- **CPU**: shared_ptr 참조 카운팅 오버헤드 (무시 가능)
- **전체적으로 무시 가능한 수준**

---

## 요약

| 항목 | 내용 |
|------|------|
| 문제 | PSO가 셰이더의 rootsig_desc 포인터를 복사 → 셰이더 삭제 시 dangling pointer |
| 원인 | rootsig_desc는 셰이더의 rootsig_deserializer가 소유한 메모리를 가리킴 |
| 해결 | shared_ptr<void>로 셰이더 internal_state 참조 → 메모리 수명 연장 |
| 효과 | 셰이더 핫리로드/정리 시 크래시 방지 |
| VizMotive | ✅ 적용 완료 (2026-01-27) |

### 핵심 교훈

> **raw 포인터를 복사할 때는 해당 메모리의 수명을 고려해야 함**
>
> 포인터가 가리키는 메모리의 소유자가 누구인지,
> 그 소유자가 언제 해제될 수 있는지 항상 확인할 것.
> 필요하다면 shared_ptr로 수명을 연장하여 안전성 확보.
