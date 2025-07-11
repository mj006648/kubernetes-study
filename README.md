---

## 👋 Welcome!

이 저장소는 Kubernetes 기반 인프라 실습, 오픈소스 도구 활용, 운영 트러블슈팅, 그리고 실전 배포 경험을 한 곳에 모으기 위해 만들어졌습니다.
실습 환경, 인프라 자동화, GitOps, 고가용성(HA) 구성, 운영 노하우, 문제 해결 경험까지 모두 기록합니다.

---

## 📚 Repository Structure

- `/kubespray/` : Kubespray를 활용한 클러스터 자동화 배포 실습
- `/kube-vip/` : kube-vip를 통한 Control Plane 고가용성(VIP) 구성
- `/kubeadm/` : kubeadm 기반 클러스터 설치 및 관리
- `/gitops/` : ArgoCD, Flux 등 GitOps 실습 및 선언적 배포 사례
- `/docs/` : 공통 개념, 운영 노트, 트러블슈팅, 비교 분석 등

---

## 🚀 Example Cluster Info

아래는 실제 실습/운영 중인 Kubernetes 클러스터의 노드 정보 예시입니다.

<summary><b>TwinX 클러스터</b></summary>

<table>
  <thead>
    <tr>
      <th>Node Name</th>
      <th>역할(Role)</th>
      <th>MGMT IP</th>
      <th>K8S IP</th>
      <th>OS-Image</th>
      <th>Kernel Version</th>
      <th>Container Runtime</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>control1</td>
      <td>control-plane</td>
      <td>10.38.38.8</td>
      <td>10.197.64.9</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.8.0-62-generic</td>
      <td>containerd://2.0.5</td>
    </tr>
    <tr>
      <td>control2</td>
      <td>control-plane</td>
      <td>10.38.38.16</td>
      <td>10.197.64.10</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.8.0-53-generic</td>
      <td>containerd://2.0.5</td>
    </tr>
    <tr>
      <td>control3</td>
      <td>control-plane</td>
      <td>10.38.38.24</td>
      <td>10.197.64.11</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.8.0-62-generic</td>
      <td>containerd://2.0.5</td>
    </tr>
    <tr>
      <td>edgebox1</td>
      <td>worker</td>
      <td>10.38.38.56</td>
      <td>10.197.64.12</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.11.0-26-generic</td>
      <td>containerd://2.0.5</td>
    </tr>
    <tr>
      <td>edgebox2</td>
      <td>worker</td>
      <td>10.38.38.64</td>
      <td>10.197.64.13</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.11.0-26-generic</td>
      <td>containerd://2.0.5</td>
    </tr>
    <tr>
      <td>edgebox3</td>
      <td>worker</td>
      <td>10.38.38.72</td>
      <td>10.197.64.14</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.11.0-26-generic</td>
      <td>containerd://2.0.5</td>
    </tr>
    <tr>
      <td>edgebox4</td>
      <td>worker</td>
      <td>10.38.38.80</td>
      <td>10.197.64.15</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.11.0-26-generic</td>
      <td>containerd://2.0.5</td>
    </tr>
    <tr>
      <td>SV4000-1</td>
      <td>worker</td>
      <td>10.38.38.32</td>
      <td>10.197.64.16</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>-</td>
      <td>-</td>
    </tr>
    <tr>
      <td>SV4000-2</td>
      <td>worker</td>
      <td>10.38.38.40</td>
      <td>10.197.64.17</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>-</td>
      <td>-</td>
    </tr>
    <tr>
      <td>RM352-1</td>
      <td>worker</td>
      <td>10.38.38.88</td>
      <td>10.197.64.18</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>-</td>
      <td>-</td>
    </tr>
    <tr>
      <td>RM352-2</td>
      <td>worker</td>
      <td>10.38.38.48</td>
      <td>10.197.64.19</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>-</td>
      <td>-</td>
    </tr>
  </tbody>
</table>


<summary><b>클러스터2 (MiniX)</b></summary>

<table>
  <thead>
    <tr>
      <th>Node Name</th>
      <th>역할(Role)</th>
      <th>Internal IP</th>
      <th>OS-Image</th>
      <th>Kernel Version</th>
      <th>Container Runtime</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>com1</td>
      <td>control-plane</td>
      <td>10.34.48.100</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.8.0-59-generic</td>
      <td>containerd://2.0.3</td>
    </tr>
    <tr>
      <td>com2</td>
      <td>worker1</td>
      <td>10.34.48.101</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.8.0-63-generic</td>
      <td>containerd://2.0.3</td>
    </tr>
    <tr>
      <td>com3</td>
      <td>worker2</td>
      <td>10.34.48.102</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.8.0-53-generic</td>
      <td>containerd://2.0.3</td>
    </tr>
    <tr>
      <td>com4</td>
      <td>worker3</td>
      <td>10.34.48.103</td>
      <td>Ubuntu 24.04.2 LTS</td>
      <td>6.8.0-55-generic</td>
      <td>containerd://2.0.3</td>
    </tr>
  </tbody>
</table>

</details>
