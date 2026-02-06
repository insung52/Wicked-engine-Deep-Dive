# 커밋 #11: 8582ea3d - Terrain determinism fixes (PSO 바인딩 최적화)

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `8582ea3d` |
| 날짜 | 2025-04-28 |
| 작성자 | Turánszki János |
| 카테고리 | 버그수정 / 최적화 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | PSO 바인딩 로직 최적화 |
| wiNoise.h | 크로스 플랫폼 결정론적 노이즈 (본래 목적) |

**참고**: 커밋 제목은 "Terrain determinism fixes"이지만, DX12 변경은 부수적인 **PSO 바인딩 최적화**입니다.

---

## 배경 지식: PSO (Pipeline State Object)

### PSO란?

**PSO (Pipeline State Object)**는 DX12에서 렌더링 파이프라인의 모든 상태를 하나의 객체로 묶은 것입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pipeline State Object (PSO)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Vertex      │  │ Pixel       │  │ Geometry    │              │
│  │ Shader      │  │ Shader      │  │ Shader      │  셰이더들    │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Blend       │  │ Rasterizer  │  │ DepthStencil│              │
│  │ State       │  │ State       │  │ State       │  상태들      │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Input       │  │ Primitive   │  │ Render      │              │
│  │ Layout      │  │ Topology    │  │ Target Fmt  │  포맷들      │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### DX11 vs DX12 상태 관리

**DX11 (개별 상태 설정)**:
```cpp
// 상태를 개별적으로 설정 - GPU가 매번 조합
context->VSSetShader(vertexShader, nullptr, 0);
context->PSSetShader(pixelShader, nullptr, 0);
context->RSSetState(rasterizerState);
context->OMSetBlendState(blendState, nullptr, 0xFFFFFFFF);
context->OMSetDepthStencilState(depthStencilState, 0);
// ... draw call ...
```

**DX12 (PSO 단위 설정)**:
```cpp
// 미리 컴파일된 PSO를 통째로 바인딩 - GPU가 즉시 사용
commandList->SetPipelineState(pso);
// ... draw call ...
```

### PSO의 장점

1. **성능**: 상태 조합을 미리 컴파일 → 런타임 오버헤드 감소
2. **검증**: 생성 시점에 유효성 검사 → 런타임 에러 감소
3. **드라이버 최적화**: 드라이버가 PSO를 미리 최적화 가능

### PSO의 단점

1. **조합 폭발**: 셰이더 × 블렌드 × 뎁스 × ... = 수천 개 PSO
2. **생성 비용**: PSO 생성은 무거운 작업 (수십~수백 ms)
3. **메모리**: 각 PSO가 GPU 메모리 차지

---

## 배경 지식: Static PSO vs Dynamic PSO

### Static PSO

**컴파일 타임에 완전히 결정**되는 PSO입니다.

```cpp
// 예: 기본 불투명 렌더링
PipelineStateDesc desc;
desc.vs = opaqueVS;
desc.ps = opaquePS;
desc.bs = BlendState::Opaque;      // 고정
desc.dss = DepthStencilState::Default;  // 고정
desc.rs = RasterizerState::CullBack;    // 고정
// → 항상 동일한 PSO
```

**특징**:
- 한 번 생성하면 변하지 않음
- 포인터 비교로 동일성 확인 가능
- 캐싱이 간단함

### Dynamic PSO

**런타임에 조합이 결정**되는 PSO입니다.

```cpp
// 예: RenderPass에 따라 달라지는 렌더링
PipelineStateDesc desc;
desc.vs = shader->vs;
desc.ps = shader->ps;
desc.bs = material->blendState;         // 머티리얼에 따라 다름
desc.dss = renderPass->depthState;      // RenderPass에 따라 다름
desc.rt_formats = renderPass->formats;  // RenderPass에 따라 다름
// → 조합에 따라 다른 PSO
```

**특징**:
- 런타임에 조합 결정
- 해시 계산으로 동일성 확인 필요
- PSO 캐시 테이블 필요

---

## 배경 지식: PSO 캐싱 전략

### 왜 캐싱이 필요한가?

```
Draw Call 1: SetPSO(A) → Draw
Draw Call 2: SetPSO(A) → Draw  ← 같은 PSO인데 또 바인딩?
Draw Call 3: SetPSO(A) → Draw  ← 또?
Draw Call 4: SetPSO(B) → Draw
Draw Call 5: SetPSO(B) → Draw  ← 같은 PSO인데 또?
```

`SetPipelineState()`는 가벼운 호출이지만, 불필요한 호출을 피하면 CPU 오버헤드 감소.

### 캐싱 구현

```cpp
struct CommandList_DX12 {
    ID3D12PipelineState* active_pso = nullptr;     // 현재 바인딩된 PSO
    uint64_t prev_pipeline_hash = 0;               // 이전 PSO 해시
};

void BindPipelineState(PipelineState* pso, CommandList cmd)
{
    // 1. 이미 같은 PSO가 바인딩되어 있으면 스킵
    if (commandlist.active_pso == internal_pso) {
        return;  // early exit
    }

    // 2. 실제 바인딩
    commandlist.GetGraphicsCommandList()->SetPipelineState(internal_pso);
    commandlist.active_pso = internal_pso;
}
```

---

## 문제 상황

### Static PSO 경로 문제

```cpp
void GraphicsDevice_DX12::BindPipelineState(const PipelineState* pso, CommandList cmd)
{
    // Static PSO 체크
    if (internal_state->pso != nullptr)
    {
        // static PSO 바인딩
        if (commandlist.active_pso != internal_state->pso)
        {
            commandlist.GetGraphicsCommandList()->SetPipelineState(internal_state->pso);
            commandlist.active_pso = internal_state->pso;
        }
        // 여기서 return이 없음! → 아래 dynamic PSO 코드도 실행됨
    }

    // Dynamic PSO 코드... (불필요하게 실행)
    uint64_t pipeline_hash = ...;  // 해시 계산 (비용 발생)
}
```

**문제**: static PSO를 바인딩해도 아래의 dynamic PSO 해시 계산이 불필요하게 실행됨.

### Dynamic PSO 캐싱 문제

```cpp
// Dynamic PSO 경로
uint64_t pipeline_hash = ComputeHash(desc);

if (commandlist.prev_pipeline_hash == pipeline_hash)
{
    // 해시가 같으면 PSO도 같다고 가정 → early exit
    return;  // ⚠️ active_pso 갱신 안 함!
}

// 해시가 다르면 PSO 찾기/생성
auto pso = FindOrCreatePSO(pipeline_hash);
commandlist.GetGraphicsCommandList()->SetPipelineState(pso);
commandlist.active_pso = pso;
commandlist.prev_pipeline_hash = pipeline_hash;
```

**문제**: 해시가 같아서 early exit할 때 `active_pso`를 갱신하지 않음.

### 왜 문제인가?

```
시나리오:

1. BindPipelineState(PSO_A) 호출
   - pipeline_hash = 12345
   - active_pso = PSO_A
   - prev_pipeline_hash = 12345

2. 다른 곳에서 active_pso가 변경됨 (예: Compute PSO 바인딩)
   - active_pso = PSO_Compute

3. BindPipelineState(PSO_A) 다시 호출
   - pipeline_hash = 12345
   - prev_pipeline_hash == pipeline_hash → early exit!
   - active_pso는 여전히 PSO_Compute ❌

결과: 코드는 PSO_A가 바인딩되었다고 생각하지만,
      실제로는 PSO_Compute가 바인딩된 상태!
```

---

## 해결: 두 가지 수정

### 수정 1: Static PSO 조기 종료

```cpp
void GraphicsDevice_DX12::BindPipelineState(const PipelineState* pso, CommandList cmd)
{
    if (internal_state->pso != nullptr)
    {
        // static PSO 바인딩
        if (commandlist.active_pso != internal_state->pso)
        {
            commandlist.GetGraphicsCommandList()->SetPipelineState(internal_state->pso);
            commandlist.active_pso = internal_state->pso;
        }
        else
            return;  // ← 추가: 이미 바인딩되어 있으면 즉시 종료
    }

    // Dynamic PSO 코드는 static PSO일 때 실행되지 않음
}
```

**효과**: static PSO 경로에서 불필요한 해시 계산 회피

### 수정 2: Dynamic PSO active_pso 갱신

```cpp
// Dynamic PSO 경로
uint64_t pipeline_hash = ComputeHash(desc);

if (commandlist.prev_pipeline_hash == pipeline_hash)
{
    commandlist.active_pso = pso;  // ← 추가: active_pso 갱신!
    return;
}

// 해시가 다르면 PSO 찾기/생성
auto pso = FindOrCreatePSO(pipeline_hash);
commandlist.GetGraphicsCommandList()->SetPipelineState(pso);
commandlist.active_pso = pso;
commandlist.prev_pipeline_hash = pipeline_hash;
```

**효과**: 해시 캐시 히트 시에도 active_pso 상태 일관성 보장

---

## 수정 전후 비교

### 수정 전

```cpp
if (internal_state->pso != nullptr)
{
    if (commandlist.active_pso != internal_state->pso)
    {
        commandlist.GetGraphicsCommandList()->SetPipelineState(internal_state->pso);
        commandlist.active_pso = internal_state->pso;
    }
    // return 없음 → 아래 코드 계속 실행
}

// dynamic PSO...
if (commandlist.prev_pipeline_hash == pipeline_hash)
{
    return;  // active_pso 갱신 없이 종료
}
```

### 수정 후

```cpp
if (internal_state->pso != nullptr)
{
    if (commandlist.active_pso != internal_state->pso)
    {
        commandlist.GetGraphicsCommandList()->SetPipelineState(internal_state->pso);
        commandlist.active_pso = internal_state->pso;
    }
    else
        return;  // ← 추가: static PSO 조기 종료
}

// dynamic PSO...
if (commandlist.prev_pipeline_hash == pipeline_hash)
{
    commandlist.active_pso = pso;  // ← 추가: active_pso 갱신
    return;
}
```

---

## VizMotive 적용

### 적용 일자
2026-01-27

### VizMotive 커밋
`2b18b324`

### 적용 내용

**GraphicsDevice_DX12.cpp**:

```cpp
// 1. static PSO 조기 종료 추가 (line 6843)
if (commandlist.active_pso != internal_state->pso)
{
    commandlist.GetGraphicsCommandList()->SetPipelineState(internal_state->pso);
    commandlist.active_pso = internal_state->pso;
}
else
    return;  // early exit for static pso

// 2. dynamic PSO 캐싱 개선 (line 6855-6857)
if (commandlist.prev_pipeline_hash == pipeline_hash)
{
    commandlist.active_pso = pso;  // 누락되었던 부분 추가
    return;  // early exit for dynamic pso|renderpass
}
```

---

## 효과

### 성능 개선
- static PSO 경로에서 불필요한 해시 계산 제거
- CPU 오버헤드 감소 (특히 draw call이 많은 씬에서)

### 버그 수정
- PSO 해시 캐시 히트 시에도 `active_pso` 상태 일관성 보장
- 다른 PSO 타입(Graphics/Compute) 전환 후 잘못된 PSO 참조 방지

---

## 요약

| 변경 항목 | 변경 전 | 변경 후 |
|-----------|---------|---------|
| static PSO 경로 | return 없음 → dynamic 코드 실행 | `else return` → 즉시 종료 |
| dynamic PSO 캐시 히트 | `active_pso` 갱신 안 함 | `active_pso = pso` 갱신 |
| 상태 일관성 | 잠재적 불일치 | 항상 일치 보장 |

### 핵심 교훈

> **캐싱 최적화에서도 상태 일관성을 유지해야 함**
>
> "해시가 같으면 PSO도 같다"는 맞지만,
> `active_pso` 변수도 갱신해야 다른 코드에서 올바른 상태를 볼 수 있음
