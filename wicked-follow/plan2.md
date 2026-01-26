# dx12 6개월 단위로 follow

* Graphics Backend 
  - GBackendDX12 를 Wicked engine DX12 backend update 를 그대로 따라서 진행.

![alt text](image-1.png)
GBackendMetal 새로 프로젝트 만들기
follow 하면서 정리하면서 하기

# 최신버전 기준으로 

* RayTracing 
  - VizMotive 에서는 PathTracing 으로 RenderPath3D 를 상속받아 구현

* GI Shaders
  - VoxelGI, SurfelGI 추가

* 물리엔진 (Jolt)
  - Jolt engine 과 이것의 wrapper APIs VizMotive 에 이식
  - 이 부분에서 VizMotive physics layer 가 향후 Genesis Physics 와 연동될 것을 염두에 두고 진행





* Direct Volume Rendering
  - Medical Viewer 용 (회사 지원)