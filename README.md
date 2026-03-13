# Kubernetes Study Reference

본 저장소는 쿠버네티스 핵심 개념, 실무 인프라 구축, 그리고 운영 트러블슈팅 경험을 기록하는 전문 학습 레퍼런스입니다. 공식 문서를 바탕으로 한 심층 가이드와 실제 운영 중인 클러스터 정보를 포함하고 있습니다.

## Repository Structure

현재 이 저장소는 다음과 같은 구조로 운영되고 있습니다.

- **/docs/** : 쿠버네티스 핵심 오브젝트 및 최신 기능에 대한 심층 기술 가이드
- **/planned/** : (진행 예정) Kubespray, kube-vip, GitOps(ArgoCD) 등 실습 설정 파일

## Kubernetes Deep-Dive Guides

실무에서 가장 중요하게 다루어지는 핵심 기능들에 대한 상세 가이드입니다. 각 문서에는 상세한 개념 설명과 바로 사용 가능한 실전 YAML 예시가 포함되어 있습니다.

- **[ServiceAccount 및 RBAC 심층 가이드](docs/ServiceAccount.md)**: 클러스터 보안의 기초인 신원 증명과 정밀한 권한 제어 방법
- **[Gateway API 정밀 가이드](docs/GatewayAPI.md)**: Ingress를 대체하는 차세대 서비스 네트워킹 표준 및 역할 기반 설정
- **[Persistent Volumes (PV/PVC) 정밀 가이드](docs/PersistentVolumes.md)**: 데이터 영속성 보장을 위한 스토리지 추상화 및 동적 할당 실무
- **[Dynamic Resource Allocation (DRA) 정밀 가이드](docs/DynamicResourceAllocation.md)**: GPU 등 고성능 하드웨어를 세밀하게 할당하는 최신 리소스 모델

## Cluster Information

아래는 현재 학습 및 운영 환경으로 활용 중인 클러스터의 상세 정보입니다.

### TwinX Cluster
본 클러스터는 고가용성(HA) 환경으로 구성된 메인 실습 환경입니다.

| Node | 역할(Role) | MGMT IP | K8S IP | K8S Version | OS-Image | Container Runtime |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| control1 | control-plane | 10.38.38.8 | 10.197.64.9 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| control2 | control-plane | 10.38.38.16 | 10.197.64.10 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| control3 | control-plane | 10.38.38.24 | 10.197.64.11 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| edgebox1 | worker | 10.38.38.56 | 10.197.64.12 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| edgebox2 | worker | 10.38.38.64 | 10.197.64.13 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| edgebox3 | worker | 10.38.38.72 | 10.197.64.14 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| edgebox4 | worker | 10.38.38.80 | 10.197.64.15 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| SV4000-1 | worker | 10.38.38.32 | 10.197.64.16 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| SV4000-2 | worker | 10.38.38.40 | 10.197.64.17 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| RM352-1 | worker | 10.38.38.88 | 10.197.64.18 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |
| RM352-2 | worker | 10.38.38.48 | 10.197.64.19 | v1.33.2 | Ubuntu 24.04.2 | containerd://2.0.5 |

### MiniX Cluster
가벼운 테스트 및 엣지 컴퓨팅 시뮬레이션을 위한 서브 클러스터입니다.

| Node | 역할(Role) | MGMT IP | K8S IP | K8S Version | OS-Image | Container Runtime |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| com1 | control-plane | 10.34.48.100 | 10.34.48.100 | v1.32.2 | Ubuntu 24.04.2 | containerd://2.0.3 |
| com2 | worker1 | 10.34.48.101 | 10.34.48.101 | v1.32.2 | Ubuntu 24.04.2 | containerd://2.0.3 |
| com3 | worker2 | 10.34.48.102 | 10.34.48.102 | v1.32.2 | Ubuntu 24.04.2 | containerd://2.0.3 |
| com4 | worker3 | 10.34.48.103 | 10.34.48.103 | v1.32.2 | Ubuntu 24.04.2 | containerd://2.0.3 |

## Usage and Contributing

- 각 문서의 상세 내용은 해당 마크다운 파일 링크를 통해 확인하실 수 있습니다.
- 실전 운영 경험이나 트러블슈팅 사례, 개선 제안은 언제든 Issue나 PR을 통해 환영합니다.

---
*Maintained by [mj006648](https://github.com/mj006648)*
