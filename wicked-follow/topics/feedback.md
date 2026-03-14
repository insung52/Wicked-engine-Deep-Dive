header 에 전역으로 inline 파라미터 (class 나 structure 가 아닌) 가 선언시키는 것은 잘 안 하는 것인데
이건 제가 봤을 때, wicked 에서 지양해야 하는 방향으로 코드를 구성한 거 같군요. 물론 wicked 는 단일 lib 파일 구조 (static lib based) 로 구성하고자 이렇게 한 거 같은데...
제가 봤을 때는 포인터 구조는 wicked 처럼 가되,
해당 전역 변수 및 함수 정의 부분들 (전역 변수 참조하는 것들) 은
Engine.dll (앞으로 ShaderEngine 이 아님)  에 속하는 cpp 로 구현하면 모든 문제가 깔끔하게 해결되리라 봅니다.
지금 Allocator.h 는 구성 상, vizmotive 기준 UTILS 그룹에 속하며
필터 구성에서 Public 에 포함되어야 합니다. Allocator.cpp 는 Private 에 구성되면 됩니다.

위에서 Public (Header Only) 에 선언되어 있는 모든 함수들은 dll 경계로부터 자유로운 독립적 함수들로 구성되어 있습니다. (모두 inline 이며, 해당 코드가 참조되는 곳에 그대로 copy&paste 된다고 생각하면 편합니다.)
- inline 함수가 어떻게 dll 경계로부터 자유로울 수 있는거지?
  → inline 함수는 컴파일 시점에 호출하는 곳에 코드가 그대로 복사(copy-paste)된다.
     Engine.dll에서 inline 함수를 호출하면 그 코드가 Engine.dll 안에 컴파일되고,
     GBackendDX12.dll에서 호출하면 GBackendDX12.dll 안에 컴파일된다.
     즉, cross-DLL 함수 호출이 발생하지 않는다. 각 DLL이 자기 자신 안에서 처리하므로 DLL 경계를 넘지 않는다.
     반면 inline 변수는 다르다. 함수 코드는 여러 곳에 복사되어도 괜찮지만,
     변수는 하나의 메모리 주소에 있어야 하기 때문에 DLL마다 별도 인스턴스가 생기는 것이 문제가 된다.
그런데 Public (APIs managed by core) 부분은 Engine.dll 에 정의되어 있는 함수를 호출하는 구성이며, 다른 dll 에서 해당 함수를 호출하면 dll export 관점으로 허용되는 함수를 콜합니다. (engine.dll 에 정의되어 있는 내용으로 수행. 메모리 관련해서는 engine.dll 에서 관리)
이렇게 한다고 생각하고 구성해 보세요.

- 이 utils 그룹이 뭐고, 그룹 내 분류가 어떤 의미인지 찾아보기
  → Engine_SOURCE.vcxitems.filters 파일에 Visual Studio 필터 구조가 정의되어 있다.
     Utils (UTIL_EXPORT) 그룹은 EngineCore 안의 유틸리티 모음이며, 세 가지로 나뉜다:

     [Public (Header Only)]
       - 순수 헤더 온리. inline 함수/템플릿만 있어서 컴파일 시점에 코드가 복사됨 → DLL 경계 문제 없음
       - 예: Spinlock.h, Timer.h, Color.h, vzMath.h, Allocator.h (현재 여기)

     [Public (APIs managed by core)]
       - 헤더에 선언, Engine.dll의 .cpp에 구현. dllexport로 공개되어 다른 DLL에서 호출 가능
       - 예: JobSystem.h, Profiler.h, Backlog.h, Config.h
       - 교수님이 Allocator.h를 여기로 이동하라고 하신 것

     [Private]
       - Engine.dll 내부 구현. 외부 DLL에 노출 안 됨
       - 예: ECS.h, Helpers2.h, JobSystem.cpp, Profiler.cpp 등 (대응하는 .cpp들이 여기)
       - 교수님이 Allocator.cpp를 여기에 두라고 하신 것

- Utils 는 Engine.dll 에 포함되는건가? 아직 이 vizmotive 의 엔진 구조를 내가 완벽하게 이해하지 못하는 상황인거 같음. 그냥 dll 이 여러개다 정도만 이해한 느낌?
  → 맞다. EngineCore/ 폴더 전체가 Engine.dll로 빌드된다.
     Utils/는 EngineCore/ 안의 하위 폴더이므로 Utils도 Engine.dll에 포함된다.
     VizMotive의 DLL 구조 전체를 한 눈에 보고 싶으면 아래 appendix 문서 요청 참고.

- vizmotive 의 전체 구조를 설명하는 문서가 하나 있으면 좋을거 같다. (appendix 로)
  → appendix_vizmotive_structure.md 신규 작성 예정.

allocator.h 를 Utils 의 Public (APIs managed by core) 로
- 지금은 public (header only 로 가있는 상태)
inline 선언된 변수를 allocator.cpp 의 전역으로 두기 (기왕이면 static 으로 선언, 이 때는 당연하지만 inline 으로 할 필요 없음)
allocator.h 에서 선언된 함수 및 자료구조는 wicked 와 동일하게