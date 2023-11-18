# MLOps

현재 진행하는 실습은 EKS 환경에서 진행하였으며, `Deploy the vanilla version of Kubeflow on AWS Using Kustomize`를 사용하여 Kubeflow를 배포하였다.

목차

- [Practitioners guide to MLOps](./mlops-component.md)
- [Deploy Kubernetes with GPU enabled](./setup_kubernetes.md)
- [Deploy Kubeflow on BareMetal Using Kustomize](./Kubeflow_BareMetal.md)
- [Deploy the vanilla version of Kubeflow on AWS Using Kustomize](./Kubeflow_EKS.md)

## Kubeflow

Kubeflow는 Kubernetes 용 기계 학습 툴킷이다. ML 워크플로우에 필요한 서비스를 만드는 것이 아닌, 각 영역에서 가장 적합한 오픈 소스 시스템들을 제공하는 것이다.

ML Workflow

1. 데이터 준비
2. 모델 교육
3. 예측 제공
4. 서비스 관리

Kubeflow는 Kubernetes가 하기의 기능을 수행하도록 하여 ML모델을 확장하고 Production에 배포하는 작업을 간단하게 한다.

- 다양한 인프라에서 쉽고 반복 가능하며 이식 가능한 배포
- 느슨하게 결합된 마이크로서비스 배포 및 관리
- 수요에 따른 확장

#### 컴포넌트

![experimental phase](./img/experimental_phase.png)

![production phase](./img/production_phase.png)

- 쥬피터 노트북
- 메인 대시보드
- 하이퍼파라미터 튜닝
- 파이프라인
- 서빙
- 학습
- etc
