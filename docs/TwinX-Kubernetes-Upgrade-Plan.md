# TwinX Kubernetes 1.35 Upgrade Plan

> 작성일: 2026-06-25  
> 상태: 계획 문서. 실제 업그레이드, 재시작, drain, rollout은 아직 수행하지 않음.

이 문서는 TwinX 클러스터를 Kubernetes 1.35 계열로 업그레이드하면서 Hubble 관측 기능과 DRA 기반 GPU 자원 할당 실험을 함께 준비하기 위한 작업 메모이다.

## 1. 현재 확인된 상태

### Kubernetes

현재 `kubectl get nodes -o wide` 기준 모든 노드는 Kubernetes `v1.33.3`이다.

| 구분 | 상태 |
| --- | --- |
| Control plane | `control1`, `control2`, `control3` 모두 `v1.33.3` |
| Worker / GPU nodes | 모든 노드 `v1.33.3` |
| OS | 대부분 Ubuntu 24.04.x, `sv4000-1`만 Ubuntu 22.04.4 LTS |
| Container runtime | containerd 2.1.3 |
| 특이사항 | `edgebox1~4`는 `SchedulingDisabled` 상태 |

### Kubespray

- 현재 로컬 Kubespray 체크아웃의 checksum 정의는 `1.33.3`까지만 확인된다.
- 최신 정식 릴리스 `v2.31.0`의 component version은 Kubernetes `1.35.4`이다.
- `master` 브랜치에는 1.36 계열 정의가 보이지만, 운영 클러스터는 정식 릴리스 기준으로 접근한다.

따라서 현재 운영 목표는 `1.36.1`이 아니라 **Kubernetes 1.35.4**로 잡는다.

## 2. 업그레이드 목표와 순서

목표 버전:

```text
current: v1.33.3
target:  v1.35.4
```

Kubernetes와 Kubespray 모두 마이너 버전 점프를 피하는 것이 안전하다.

```text
v1.33.3
→ v1.34.x
→ v1.35.4
```

권장 실행 원칙:

1. etcd snapshot 및 `/etc/kubernetes` 주요 설정 백업
2. Kubespray 릴리스 노트와 inventory removed vars 검토
3. control-plane / etcd 우선 업그레이드
4. worker 노드는 drain / upgrade / uncordon 순서로 처리
5. 가능하면 `serial=1`로 한 노드씩 진행
6. 각 단계 후 API server, CoreDNS, Cilium, GPU Operator, 주요 워크로드 확인

## 3. Kubespray v2.31.0 주요 주의사항

Kubespray v2.31.0 릴리스 노트에서 특히 확인해야 할 항목은 다음과 같다.

| 항목 | 영향 |
| --- | --- |
| Kubernetes 1.35.4 default | 이번 목표 버전 |
| cgroup v1 미지원 | Ubuntu 24.04는 보통 cgroup v2라 유리하지만 실제 노드 확인 필요 |
| ingress-nginx addon 제거 | Kubespray addon으로 사용 중이면 제거 또는 별도 관리 필요 |
| Kubernetes Dashboard addon 제거 | `dashboard_enabled: true` 사용 여부 확인 필요 |
| removed vars validation | 예전 inventory 변수가 있으면 playbook 초반에 중단될 수 있음 |
| etcd 3.6.10 | 기존 etcd가 너무 낮으면 중간 버전 경유 필요 |
| Cilium 1.19.3 | 현재 TwinX Cilium 1.17.3에서 점프가 발생할 수 있어 별도 검토 필요 |

## 4. Cilium / Hubble 계획

현재 TwinX Cilium 상태:

```text
cilium:          v1.17.3
cilium-operator: v1.17.3
hubble:          disabled
```

요청사항은 TwinX Cilium에서 Hubble을 사용할 수 있게 하는 것이다. 목표는 우선 **Hubble Relay + Metrics**만 켜고 UI는 제외하는 방향이 안전하다.

Kubespray inventory 기준 최소 설정안:

```yaml
# inventory/mycluster/group_vars/k8s_cluster/k8s-net-cilium.yml

cilium_enable_hubble: true
cilium_enable_hubble_ui: false

cilium_hubble_metrics:
  - drop
  - tcp
  - flow
  - icmp
  - httpV2:exemplars=true

cilium_install_extra_flags: >-
  --set hubble.eventQueueSize=8191
  --set hubble.metrics.enableOpenMetrics=true
  --set hubble.metrics.port=9965
```

주의:

- 첨부받은 DataX 예시는 Cilium `1.17.5` 기준이므로 TwinX에 그대로 복사하지 않는다.
- Relay image tag/digest는 직접 고정하지 않고 Kubespray의 `cilium_version`에 맞게 관리한다.
- Hubble enable 자체는 datapath 변경보다 위험이 낮지만, Cilium ConfigMap 변경과 agent rollout이 발생할 수 있으므로 작업창에서 수행한다.
- Cilium 버전 업그레이드는 Hubble enable과 분리해서 판단한다. Cilium 공식 가이드는 마이너 업그레이드를 순차적으로 수행하는 것을 권장한다.

검증 명령:

```bash
kubectl -n kube-system get pods,svc | grep -Ei 'cilium|hubble'
kubectl -n kube-system get cm cilium-config -o yaml | grep -i hubble
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system rollout status deploy/hubble-relay
```

## 5. DRA 계획

DRA는 GPU 같은 특수 장치를 `ResourceClaim`, `ResourceClaimTemplate`, `DeviceClass`, `ResourceSlice` 기반으로 더 세밀하게 할당하기 위한 Kubernetes 기능이다.

현재 상태:

```text
kubectl get deviceclasses  -> resource type 없음
kubectl get resourceslices -> resource type 없음
```

즉 현재 `v1.33.3` 클러스터에서는 DRA API가 활성화되어 있지 않다.

Kubernetes 1.35에서는 DRA가 stable이고 기본 활성화 상태이므로, 1.35.4 업그레이드 후 DRA 실험을 진행하는 것이 적절하다.

NVIDIA DRA Driver 관점에서 현재 긍정적인 조건:

| 항목 | 현재 상태 |
| --- | --- |
| GPU Operator | `v25.3.4` 확인 |
| NVIDIA Driver | 노드 라벨 기준 `580.126.09`, DRA GPU allocation 요구사항 충족 |
| containerd | 2.1.3, CDI 사용에 유리 |
| NFD / GFD | 배포되어 있음 |
| GPU nodes | A100, L40S, A10, T4 계열 노드 존재 |

권장 순서:

1. Kubernetes 1.35.4 업그레이드 완료
2. DRA API 확인
   ```bash
   kubectl get deviceclasses
   kubectl get resourceslices
   ```
3. NVIDIA DRA Driver 설치 전 GPU Operator / 기존 device plugin 전환 방식 검토
4. 테스트 GPU 노드 1~2개에서 DRA Driver 검증
5. `ResourceClaimTemplate` 기반 샘플 워크로드로 GPU 할당 확인

주의:

- 기존 `nvidia-device-plugin-daemonset`과 DRA Driver의 역할이 겹칠 수 있다.
- NVIDIA 문서에서는 GPU Operator와 함께 사용할 때 device plugin 비활성화 및 DRA Driver 설치 경로를 별도로 안내한다.
- 운영 전체 전환 전 테스트 노드에서 검증한다.

## 6. 작업 전 체크리스트

- [ ] Kubespray target tag 확정
- [ ] inventory 내 removed vars 사전 점검
- [ ] `dashboard_enabled`, `ingress_nginx_enabled` 사용 여부 확인
- [ ] cgroup v2 확인
- [ ] etcd snapshot
- [ ] control-plane kubeconfig 백업
- [ ] Cilium 현재 Helm values 백업
- [ ] Cilium 1.17 → 1.19 처리 전략 결정
- [ ] Hubble relay / metrics only 설정 확정
- [ ] GPU Operator 및 NVIDIA device plugin 전환 전략 결정
- [ ] DRA 테스트 대상 GPU 노드 선정

## 7. 참고 링크

- Kubespray releases: https://github.com/kubernetes-sigs/kubespray/releases
- Kubespray upgrade guide: https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/upgrades.md
- Kubernetes DRA docs: https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
- NVIDIA DRA Driver for GPUs: https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/
- Cilium upgrade guide: https://docs.cilium.io/en/stable/operations/upgrade/
