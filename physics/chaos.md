# UE Chaos Vehicle 성능 평가 지침

언리얼엔진 네이티브 **Chaos Vehicle** 물리의 성능을 측정·분석하기 위한 방법론. 다중 차량(구동계) 추가 시 병목을 식별하고, Genesis 등 외부 물리 백엔드와 레퍼런스 비교할 때 사용한다.

---

## 1. 측정 지표 약어 풀이

`stat unit` 및 CSV 프로파일러 출력의 컬럼 의미.

| 약어 | 정식 명칭 | 의미 |
|---|---|---|
| FPS | Frames Per Second | 초당 프레임 수 |
| Frame (ms) | Total Frame Time | 한 프레임 전체 소요 시간 |
| Game (ms) | Game Thread Time | 게임 스레드(메인 CPU 로직) 시간 |
| Draw (ms) | Render(Draw) Thread Time | 렌더 스레드(드로우콜 준비) 시간 |
| RHIT (ms) | RHI Thread Time | Render Hardware Interface 스레드(그래픽 API 변환) 시간 |
| GPU Time (ms) | GPU Execution Time | GPU 실제 연산 시간 |
| Input (ms) | Input Latency | 입력→반영 지연. 단, 프레임을 초과하면 **물리 스레드 블로킹 대기**가 섞인 것 의심 |
| Mem (GB) | Memory Usage | 메모리 사용량 |
| Draws | Draw Calls | 드로우콜 횟수 |
| Prims | Primitives (Triangles) | 렌더링 삼각형 수 |
| SyncBodies | Sync Physics Bodies | 물리엔진→게임 바디 트랜스폼 동기화 |
| CreateExternalAccelerationStructure | — | 브로드페이즈 가속 구조(공간 분할) 생성 |
| Update Kinematics On Deferred SkelMeshes | — | 지연 처리된 스켈레탈 메쉬 본 키네마틱 갱신 |
| Phys SetBodyTransform | Set Physics Body Transform | 물리 바디 트랜스폼 강제 설정 (kinematic) |
| Query PhysicalMaterialMask Hit | — | 표면 물리재질 마스크 조회 (휠 레이캐스트 단위) |

---

## 2. 빠른 진단 — `stat` 콘솔 명령

런타임/PIE 콘솔(`~`)에서 즉시 사용.

```
stat unit          # Frame / Game / Draw / GPU / RHIT 분해 — 1차 병목 위치 파악
stat chaos         # Chaos 솔버 전체 비용 (휠 솔버 포함)
stat physics       # 씬 쿼리 / 바디 동기화 비용
stat startfile     # 이후 구간을 stat 파일로 덤프 시작
stat stopfile      # 덤프 종료
```

**판독 규칙**
- `Frame ≈ Game` 이고 Draw/GPU/RHIT가 한참 낮음 → **게임 스레드(CPU) 바운드**. 렌더/GPU 최적화는 무의미.
- `stat chaos` 가 게임 스레드 비용과 별개로 크게 나옴 → 휠 솔버가 물리 워커에서 무겁다는 뜻.
- 물리 동기화(SyncBodies 등)가 1ms 미만인데 Frame이 큼 → 비용은 **Vehicle Movement Component tick + async physics 대기**에 있음.

---

## 3. 정밀 분석 — Unreal Insights 워크플로우

### 3-1. 트레이스 켜고 실행

실행 인자:
```
YourGame.exe -trace=default,cpu,frame,bookmark,counters -statnamedevents
```
- `cpu` 채널이 핵심 — 전 스레드(Game / Physics / RHI) 함수별 타임라인 캡처
- `-statnamedevents` 가 있어야 `STAT_` 매크로가 이름으로 표시됨

에디터 PIE에서 동적으로:
```
Trace.Start default,cpu,frame
Trace.Stop
```
Insights를 먼저 띄워두고 자동 연결하려면 `-tracehost=127.0.0.1` 추가.

### 3-2. Insights 열기

`Engine/Binaries/Win64/UnrealInsights.exe` 실행 → 생성된 `.utrace` 열기.
경로: `ProjectName/Saved/Profiling/` 또는 `%LOCALAPPDATA%/UnrealEngine/Common/...`

### 3-3. Timing Insights에서 봐야 할 것

좌측 **Frames** 패널에서 가장 무거운 프레임 클릭 → 줌인. 트랙 비교:

| 트랙 | 확인 포인트 |
|---|---|
| **GameThread** | 실제 연산 vs 빈 구간(스톨) 비율. `FTickFunctionTask` / `UChaosVehicleMovementComponent::TickComponent` 폭 |
| **PhysicsThread / Chaos** (워커 트랙) | 휠 솔버(`FChaosVehicleManager`, suspension raycast)가 게임 프레임보다 길게 삐져나오는지 |
| 두 트랙 정렬 | GameThread가 멈춘 동안 Physics가 돌고 있으면 → **블로킹 대기 확정** (`Input` 초과 현상의 정체) |

판별 핵심: GameThread에 `WaitFor...` / 빈 갭이 보이고 그 구간이 Physics 워커 작업과 겹치면 = 게임 스레드가 휠 솔버를 기다리는 스톨.

### 3-4. 함수별 비용 정량화

하단 **Timers** 패널 → 해당 프레임 범위 선택 → Inclusive time 내림차순. Chaos Vehicle 상위 용의자:
- `FChaosVehicleManager::Update` / `ParallelUpdateVehicles`
- `UChaosWheeledVehicleSimulation::UpdateSimulation`
- `FSuspensionSimModule` / scene query(raycast)

---

## 4. async physics 설정 확인

```
p.Chaos.Solver.Async        # 1이면 async on (물리 워커 스레드), 0이면 게임 스레드에서 동기 실행
```
- Async **off** → Chaos Vehicle이 통째로 게임 스레드에서 돌아 `Frame==Game` 격차가 더 심해짐
- Async **on** → 휠 솔버는 워커에서 돌지만, 게임 스레드가 결과를 대기하며 스톨할 수 있음 (`Input` 컬럼 비대화)

---

## 5. 해석 체크리스트

1. `Frame == Game` ? → 게임 스레드 바운드 확정. GPU/렌더 헤드룸 확인.
2. 차량 수 대비 Frame 증분이 선형(O(N))인가? → 차량당 고정 비용 존재 = 배칭 없음.
3. `stat physics` 동기화 비용이 작은데 Frame이 큰가? → 비용은 Vehicle tick + async 대기.
4. `Input(ms) > Frame(ms)` ? → 물리 워커 오버런으로 인한 게임 스레드 스톨 의심 → Insights로 확정.
5. `Query PhysicalMaterialMask Hit` 콜수가 차량당 ~14회로 선형 증가? → 휠 서스펜션 레이캐스트가 차량마다 독립 발생.

---

## 6. 레퍼런스 실험 결과 (Chaos Vehicle, 단일 클라이언트)

차량 수(TankCount) 증가에 따른 측정:

| TankCount | FPS | Frame=Game (ms) | 차량당 증분 |
|---|---|---|---|
| 1  | 68 | 14  | — |
| 5  | 33 | 29  | +3.75/대 |
| 10 | 21 | 47  | +3.6/대 |
| 15 | 15 | 65  | +3.6/대 |
| 20 | 11 | 86  | +3.5/대 |
| 30 | 8  | 122 | +3.7/대 |

**목표 FPS별 대략 한계 차량 수**

| 목표 FPS | 프레임 예산 | 한계 차량 수 |
|---|---|---|
| 60 FPS | 16ms | ~1대 (1대가 이미 14ms) |
| 30 FPS | 33ms | ~5대 |
| 15 FPS | 65ms | ~15대 |
| 8 FPS  | — | ~30대 |

**결론**
- 네이티브 최적화된 UE Chaos Vehicle도 **차량당 ~3.7ms, O(N) 선형, 게임 스레드 바운드**. 차량 간 배칭/벡터화 없음.
- GPU/렌더/씬 물리 동기화는 병목이 아님. 비용은 Vehicle Movement Component tick + async physics 대기.
- 다중 차량(수백~수천 대)은 엔진 종류의 문제가 아니라 **구조적 문제**. Chaos·Genesis 모두 단일 씬 다중 차량에 불리.
- 대응: **멀티프로세스 / 배칭(n_envs) / 서버 분산**이 필수. 단일 차량 충실도 비교 시 Chaos Vehicle 1대 = 14ms가 기준선.