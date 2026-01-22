Api python 스크립트화

mcp 화

해상도가 커지면 fps 감소하는 문제

렌더링 창에서 오브젝트 선택 후 끌어서 이동,회전,스케일 구현

gui 에서 스크롤 해도 렌더링 창이 같이 확대 축소됨.

scene 저장

개발 문서 최신화



wicked engine 은 imgui 를 이미 포함

imgui 없이
dpg 유지 (8bit)
엔진 초기화 시 타겟 display output 을 정할 수 있도록 (default 10bit)


WINDOW handler 

32 - 24   8비트 남음
-> R11G11B10 으로 분산
material color 중 alpha 가 중요하면 rgba8
그렇지 않으면 r11g11b10 사용
r10g10b10a2 는 alpha 가 적은 정확도로 필요할때
최종 색감 필터가 있음
swap chain 

pyimgui : opengl 만 가능함
같은 gpu 디바이스로 memory share 가능?
논리 체계는 다를 수 있음
렌더 out - gpu 메모리에 texture handler -> imgui 가 그걸 읽어서 바로 출력

디바이스 자체를 하나로 (engine device - imgui device)
둘중 하나가 다른 device 를 불러와서 사용

파이썬은 윈도우 맥 리눅스 따로
pyimgui 는 어떻게 백엔드를 선택? 


렌더링 -> swap chain

중간 버퍼를 cpu 에 다운 -> jpg 로 출력
command list?

staging texture 메모리를 계속 유지하면서 사용 (mcp 용도로)

stagingTex 를 따로 멤버로
