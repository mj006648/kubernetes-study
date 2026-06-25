# TwinX Kubernetes 1.35 Upgrade Plan

> 작성일: 2026-06-25
> 상태: 계획 문서. 실제 업그레이드, 재시작, drain, rollout은 아직 수행하지 않음.
> 최신 점검: 2026-06-25 읽기 전용 클러스터 감사 결과 반영.

이 문서는 TwinX 클러스터를 Kubernetes 1.35 계열로 업그레이드하면서 Hubble 관측 기능과 DRA 기반 GPU 자원 할당 실험을 함께 준비하기 위한 작업 메모이다.

## 1. 현재 확인된 상태

2026-06-25 기준 감사 스냅샷은 `/tmp/twinx-cluster-audit-20260625073732`에 보관했다. 이 감사에서는 `kubectl get/describe`, Ceph status, Cilium health, ArgoCD 상태 등 읽기 전용 조회만 수행했고 업그레이드, drain, restart, rollout은 수행하지 않았다.

### Kubernetes

현재 `kubectl get nodes -o wide` 기준 모든 노드는 Kubernetes `v1.33.3`이다.

| 구분 | 상태 |
| --- | --- |
| Control plane | `control1`, `control2`, `control3` 모두 `v1.33.3` |
| Worker / GPU nodes | 모든 노드 `v1.33.3` |
| OS | 대부분 Ubuntu 24.04.x, `sv4000-1`만 Ubuntu 22.04.4 LTS |
| Container runtime | containerd 2.1.3 |
| 특이사항 | `edgebox1~4`는 `SchedulingDisabled` 상태 |

### 2026-06-25 주요 리스크 요약

내일 작업의 기본 판단은 **전체 클러스터 일괄 업그레이드가 아니라, 사전 점검 후 저위험 구간부터 단계적으로 진행**하는 것이다.

| 구분 | 현재 관찰 | 판단 |
| --- | --- | --- |
| Rook-Ceph | `HEALTH_WARN`, inactive/undersized PG 33개, replica 없는 pool 18개, OSD 3개 전부 `l40s` | 가장 큰 리스크. `l40s`, `sv4000-1`은 마지막 별도 작업 |
| Harbor | registry PVC 500Gi RWO가 `rm352-1`에 attach, 새 registry pod는 `l40s`에서 `ContainerCreating` | `rm352-1` drain 전 Harbor 상태 정리 필요 |
| `l40s` runtime | Grafana/Prometheus/Milvus pod kill 실패, `DeadlineExceeded`, `RST_STREAM` 이벤트 반복 | drain 중 pod 종료가 오래 걸리거나 stuck 가능 |
| PDB | `keycloak-db-primary`, `nessiedb-primary`, `opensearch` PDB allowed disruptions 0 | stateful workload eviction 실패 가능 |
| Partridge | `sv4000-2` 단일 전용 label/taint, Partridge pod 모두 `sv4000-2` | drain 시 Partridge 중단/Pending 가능 |
| Cilium | 현재 v1.17.3, health는 12/12 reachable, Hubble disabled | 현재 통신은 OK. 1.19 계열 점프와 Hubble enable은 분리 권장 |
| control-plane | kube-apiserver/controller/scheduler는 Running이지만 일부 restart count 높음 | etcd health/snapshot과 비대화형 SSH 확인 후 진행 |
| Metrics | metrics-server 없음, `kubectl top nodes` 불가 | hard blocker는 아니지만 관측성 부족 |

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
- [ ] Ceph `HEALTH_WARN` 세부 원인 확인 및 `l40s`/`sv4000-1` 보류 여부 결정
- [ ] Harbor registry stuck pod와 RWO RBD attachment 상태 확인
- [ ] `l40s`의 FailedKillPod / containerd 응답 지연 이벤트 재확인
- [ ] `sv4000-2` Partridge downtime 가능 여부 확인
- [ ] control-plane SSH OTP 우회/해제 및 작업 후 원복 계획 확정
- [ ] ArgoCD auto-sync가 업그레이드 중 민감 앱을 건드리지 않는지 확인

## 7. 참고 링크

- Kubespray releases: https://github.com/kubernetes-sigs/kubespray/releases
- Kubespray upgrade guide: https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/upgrades.md
- Kubernetes DRA docs: https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
- NVIDIA DRA Driver for GPUs: https://dra-driver-nvidia-gpu.sigs.k8s.io/docs/
- Cilium upgrade guide: https://docs.cilium.io/en/stable/operations/upgrade/

## 8. TwinX 노드별 drain 전략

2026-06-25 읽기 전용 점검 기준, TwinX는 일반적인 전체 `drain=true` 전략을 그대로 적용하기 어렵다. Rook-Ceph, DB, OpenSearch, Redis 같은 stateful workload와 PDB가 일부 노드에 집중되어 있기 때문이다.

현재 Ceph 상태도 `HEALTH_OK`가 아니라 `HEALTH_WARN`이다.

```text
health: HEALTH_WARN
mons d,e,f are using a lot of disk space
mon f is low on available space
Reduced data availability: 33 pgs inactive
Degraded data redundancy: 33 pgs undersized
```

따라서 `drain_nodes: false`는 전체 기본값이 아니라 **Ceph-heavy 노드에 대한 예외 우회책**으로만 사용한다. 특히 `sv4000-1`과 `l40s`는 Ceph health 경고를 먼저 줄인 뒤 마지막에 별도 작업으로 처리한다.

| Node | 주요 상태 | 권장 전략 |
| --- | --- | --- |
| `control1` | control-plane, PDB allowed=1 workload 있음 | `drain=true`, control-plane 한 대씩 순차 진행 |
| `control2` | control-plane, Rook CSI provisioner pod 있음 | 기본 `drain=true`, 막히면 원인 확인 후 예외 처리 |
| `control3` | control-plane, hard PDB 없음 | `drain=true`, control-plane 한 대씩 순차 진행 |
| `edgebox1~4` | T4 GPU node, 이미 `SchedulingDisabled`, non-DS pod 적음 | GPU job 확인 후 `drain=true` 가능성이 높음 |
| `rm352-1` | A10 GPU node, Harbor registry 500Gi RWO PVC가 attach된 상태 | **Harbor registry 정리 전 보류.** drain 시 Harbor registry 중단/볼륨 재attach 가능성 확인 |
| `rm352-2` | A10 GPU node, PDB allowed=1 workload 있음, 과거 Cilium 이슈 이력 | 표준 `drain=true` 후보. 전후 Cilium health와 pod/node 통신 검증 |
| `sv4000-2` | A100 GPU node, Partridge 전용 label/taint, Kyverno로 `partridge` namespace pod 고정 | **Partridge 전용 노드. 서비스 영향 공지 후 처리. 대체 전용 노드가 없으면 drain 시 Partridge pod는 Pending/중단 가능** |
| `l40s` | OSD 3개, MON/MGR, keycloak-db primary, nessiedb primary, redis master | **Ceph/DB-heavy 노드. Ceph WARN 해소 후 마지막에 처리. drain 실패 시 `drain_nodes=false` 예외 고려** |
| `sv4000-1` | MON 2개, MDS, RGW, Rook operator/tools, 많은 Trident pod, Ubuntu 22.04.4 | **가장 위험한 노드. OS 업그레이드와 K8s 업그레이드 분리. Ceph WARN 해소 후 별도 작업. 필요 시 `drain_nodes=false` 예외** |

### 권장 실행 순서

```text
0. 사전 점검
   - etcd snapshot
   - Kubespray inventory backup
   - Ceph status / PDB / workload 분포 확인

1. control-plane
   - control1 -> control2 -> control3
   - 기본 drain=true
   - 한 대씩 진행

2. 가벼운 GPU/worker 노드
   - edgebox1~4
   - rm352-2
   - 기본 drain=true

3. Harbor 영향 노드
   - rm352-1
   - Harbor registry 500Gi RWO PVC attachment 상태 정리 후 진행
   - registry pod가 안정적으로 한 곳에 떠 있고 stuck pod가 없는지 확인
   - Harbor가 중요하면 별도 짧은 중단 공지 후 drain

4. Partridge 전용 노드 별도 처리
   - sv4000-2
   - `twinx.dreamai.kr/dedicated-node=partridge` label/taint가 있는 유일한 노드
   - Kyverno `inject-partridge-node-selector` 정책이 partridge namespace pod를 이 노드로 고정
   - Partridge 작업자 중단 가능성을 공지하고, 필요하면 대체 노드에 동일 label/taint를 준비한 뒤 진행
   - 대체 노드가 없으면 `drain=true` 시 Partridge pod는 evict 후 sv4000-2가 uncordon될 때까지 Pending 가능
   - 짧은 kubelet 업그레이드만 하고 Partridge 중단을 피해야 하면 `drain_nodes=false` 예외를 검토하되, reboot/runtime/CNI 변경과는 묶지 않음

5. Ceph-heavy 노드 별도 처리
   - l40s
   - sv4000-1
   - Ceph HEALTH_WARN 원인 완화 후 진행
   - reboot, OS upgrade, containerd upgrade, Cilium major/minor upgrade와 동시에 묶지 않음
   - drain이 PDB/Rook-Ceph 때문에 막힐 경우에만 drain_nodes=false 예외 사용
```

### `drain_nodes=false` 사용 기준

사용 가능 조건:

- Kubernetes 컴포넌트 업그레이드만 수행
- 노드 reboot 없음
- OS/kernel/containerd 변경 없음
- Cilium 버전 점프 없음
- Ceph 상태가 최소한 악화되지 않음
- 한 번에 한 노드만 진행

사용 금지 또는 보류 조건:

- Ceph `HEALTH_ERR`
- inactive/undersized PG가 증가 중
- monitor quorum 불안정
- OSD down/out 발생
- 노드 reboot 필요
- Cilium 1.17 → 1.19 같은 네트워크 플러그인 점프를 동시에 수행
- GPU driver/operator 변경을 동시에 수행

즉, TwinX에서는 다음 원칙을 따른다.

```text
기본값: drain_nodes=true
예외: l40s/sv4000-1 같은 Ceph-heavy 노드에서 drain이 PDB 때문에 막힐 때만 drain_nodes=false 고려
금지: 전체 클러스터를 drain_nodes=false로 일괄 업그레이드
```

### `sv4000-2` Partridge 전용 노드 주의사항

`sv4000-2`는 일반 A100 worker가 아니라 Partridge 전용 노드로 운영 중이다.

확인된 상태:

```text
node label: twinx.dreamai.kr/dedicated-node=partridge
node taint: twinx.dreamai.kr/dedicated-node=partridge:NoSchedule
Kyverno policy: inject-partridge-node-selector
Partridge labeled node: sv4000-2 단일 노드
```

Kyverno 정책은 `partridge` namespace의 pod에 다음 스케줄링 조건을 주입한다.

```yaml
nodeSelector:
  twinx.dreamai.kr/dedicated-node: partridge
tolerations:
  - key: twinx.dreamai.kr/dedicated-node
    operator: Equal
    value: partridge
    effect: NoSchedule
```

따라서 `sv4000-2`를 drain하면 Partridge pod는 다른 일반 GPU 노드로 이동하지 않는다. 대체로 다음 중 하나를 선택해야 한다.

| 선택지 | 의미 | 권장 상황 |
| --- | --- | --- |
| `drain=true` | Partridge pod를 정상 eviction. 대체 partridge 노드가 없으면 일시 중단/Pending 가능 | 서비스 중단 공지가 가능할 때 |
| 대체 노드 준비 | 다른 GPU 노드에 동일 label/taint를 임시 부여해 Partridge를 이동 가능하게 함 | Partridge 무중단 또는 짧은 중단이 필요할 때 |
| `drain_nodes=false` | pod를 evict하지 않고 kubelet 업그레이드만 시도 | reboot/runtime/CNI 변경이 없고 아주 짧은 영향만 허용할 때 |

현재 Partridge pod는 `1/2 Running` 상태로 보이므로, 업그레이드 전에는 애플리케이션 소유자와 현재 정상 동작 기준을 먼저 확인한다.

## 9. 추가 preflight: rm352 Cilium 이력과 control-plane OTP

### rm352-1 / rm352-2 Cilium 통신 이력

과거 `rm352-1`, `rm352-2`에서 Cilium 관련 문제로 pod-to-node 또는 node 통신이 깨진 이력이 있다. 현재 읽기 전용 점검에서는 두 노드의 Cilium agent와 Cilium health가 정상으로 보인다.

현재 확인 결과:

```text
rm352-1 cilium status: OK
rm352-2 cilium status: OK
cilium-health: 12/12 reachable
routing-mode: tunnel
 tunnel-protocol: vxlan
kube-proxy-replacement: false
```

하지만 과거 장애 이력이 있으므로 `rm352-1`, `rm352-2`는 일반 worker이지만 다음 검증을 반드시 전후로 수행한다.

업그레이드 전:

```bash
kubectl -n kube-system exec <rm352-1-cilium-pod> -c cilium-agent -- cilium status --brief
kubectl -n kube-system exec <rm352-1-cilium-pod> -c cilium-agent -- cilium-health status
kubectl -n kube-system exec <rm352-2-cilium-pod> -c cilium-agent -- cilium status --brief
kubectl -n kube-system exec <rm352-2-cilium-pod> -c cilium-agent -- cilium-health status
kubectl -n kube-system get pods -o wide | grep -E 'rm352-1|rm352-2|cilium'
```

업그레이드 후:

```bash
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system exec <rm352-1-cilium-pod> -c cilium-agent -- cilium-health status
kubectl -n kube-system exec <rm352-2-cilium-pod> -c cilium-agent -- cilium-health status
kubectl get nodes rm352-1 rm352-2 -o wide
kubectl get pods -A -o wide | grep -E 'rm352-1|rm352-2'
```

가능하면 실제 workload 기준으로 다음도 확인한다.

- rm352 위 pod에서 Kubernetes API 접근 가능 여부
- rm352 위 pod에서 node IP 접근 가능 여부
- 다른 노드 pod에서 rm352 위 pod 접근 가능 여부
- rm352 위 pod에서 ClusterIP 서비스 접근 가능 여부

중단 기준:

```text
rm352 중 하나라도 cilium-health가 12/12 reachable이 아니면 다음 노드로 진행하지 않는다.
Pod -> Node 통신이 다시 깨지면 Cilium/Hubble/DRA 작업은 중단하고 네트워크 복구를 우선한다.
```

### control-plane SSH OTP / Google Authenticator

control-plane 노드에 Google OTP 기반 SSH 2FA가 걸려 있으면 Kubespray/Ansible 자동화가 중간에 멈출 수 있다. Kubespray는 여러 노드에 반복 SSH 접속하므로, 작업 시간 동안 Ansible 실행 계정은 **비대화형 SSH 접속**이 가능해야 한다.

권장 방식:

1. 작업 전 control-plane 3대에 대해 Ansible 계정 SSH 접속 테스트
2. 작업 시간 동안만 Ansible 실행 계정의 OTP 요구를 비활성화하거나, OTP가 적용되지 않는 별도 자동화용 SSH key 경로 사용
3. root/sudo 권한이 password/OTP prompt 없이 동작하는지 확인
4. 작업 종료 후 OTP 정책 원복

사전 확인 예시:

```bash
ansible -i inventory/mycluster/inventory.ini kube_control_plane -m ping
ansible -i inventory/mycluster/inventory.ini kube_node -m ping
ansible -i inventory/mycluster/inventory.ini all -m command -a 'sudo -n true'
```

주의:

```text
OTP가 남아 있으면 업그레이드 도중 특정 control-plane에서 Ansible이 멈출 수 있다.
OTP를 해제하더라도 작업창 동안만 적용하고, 완료 후 반드시 원복한다.
```

## 10. 2026-06-25 상세 감사 기반 내일 판단

### 결론

내일 바로 `upgrade-cluster.yml`을 전체 노드 대상으로 끝까지 밀어붙이는 방식은 권장하지 않는다. 현재 클러스터에는 업그레이드 자체보다 **drain, storage detach/attach, stateful workload eviction, CNI rollout**에서 걸릴 요소가 많다.

내일 목표는 다음처럼 잡는다.

```text
1차 목표: 사전 점검 + control-plane + 저위험 worker 일부 안정 업그레이드
보류 목표: Ceph-heavy 노드, Harbor 영향 노드, Partridge 전용 노드
추가 목표: Hubble/DRA는 Kubernetes 안정화 이후 별도 적용
```

### 강한 blocker

| Blocker | 증거 | 내일 대응 |
| --- | --- | --- |
| Ceph `HEALTH_WARN` | inactive/undersized PG 33개, replica 없는 pool 18개, OSD 전부 `l40s` | Ceph health 확인 전 `l40s`, `sv4000-1` 작업 금지 |
| Harbor registry PVC | `habor-harbor-registry` 500Gi RWO PV가 `rm352-1`에 attach, 새 pod는 `l40s`에서 `ContainerCreating` | `rm352-1` drain 전 registry rollout/stuck pod 정리 |
| `l40s` pod kill 실패 | `FailedKillPod`, `DeadlineExceeded`, `RST_STREAM` 이벤트 반복 | `l40s`는 마지막. 필요 시 containerd/kubelet 상태 별도 점검 |
| PDB allowed=0 | keycloak DB, Nessie DB, OpenSearch PDB | drain 실패 시 PDB별로 원인 확인, 전역 `drain_nodes=false` 금지 |
| `sv4000-2` 단일 Partridge 노드 | label/taint가 `sv4000-2`에만 존재, Partridge pod 전부 해당 노드 | 중단 공지 또는 대체 label/taint 노드 준비 후 진행 |
| control-plane OTP | Ansible 비대화형 SSH가 막힐 수 있음 | 작업 전 OTP 우회/해제, 작업 후 원복 |

### 내일 실제 순서

```text
0. 변경 금지 상태에서 preflight
   - ansible ping / sudo -n true
   - etcd health + snapshot
   - ceph status / ceph health detail
   - kubectl get pdb -A
   - kubectl get pods -A -o wide
   - kubectl -n kube-system cilium-health 확인
   - harbor registry pod/PVC/VolumeAttachment 확인

1. 진행 여부 판단
   - etcd 불안정: upgrade 중단
   - Ceph HEALTH_ERR 또는 악화: Ceph-heavy 노드 제외
   - Harbor registry stuck 지속: rm352-1 제외
   - Cilium health 12/12 미달: Cilium/Hubble/DRA 모두 중단

2. 가능한 경우 control-plane부터 한 대씩
   - control1 -> control2 -> control3
   - 각 노드 후 API server, scheduler, controller, Cilium 확인

3. 저위험 worker
   - edgebox1~4
   - rm352-2
   - 각 노드 후 Cilium health와 GPU Operator daemonset 확인

4. 보류 또는 별도 작업
   - rm352-1: Harbor registry 안정화 후
   - sv4000-2: Partridge downtime 합의 후
   - l40s/sv4000-1: Ceph health 정리 후
```

### 시간 예상

| 범위 | 예상 시간 | 판단 |
| --- | --- | --- |
| preflight + 상태 판단 | 1~2시간 | 반드시 필요 |
| control-plane 3대 | 1~2시간 이상 | etcd/OTP 문제 없을 때 |
| edgebox/rm352-2 일부 worker | 1~3시간 | drain 상황에 따라 변동 |
| 전체 노드 완료 | 반나절~하루 이상 | 현재 상태에서는 무리하지 않는 편이 안전 |
| Ceph/Harbor/Partridge 포함 완전 완료 | 별도 작업창 권장 | 내일 한 번에 묶지 않기 |

### 중단 기준

다음 중 하나라도 발생하면 다음 노드로 진행하지 않는다.

```text
- etcd endpoint health 실패
- kube-apiserver 3대 중 2대 이상 불안정
- Cilium health 12/12 reachable 미달
- Ceph inactive/undersized PG 증가
- Harbor registry가 Running 0개가 됨
- RBD volume detach/attach가 stuck
- drain이 PDB로 막혔는데 영향 workload 소유자가 불명확함
- Partridge pod가 의도치 않게 Pending/CrashLoop 상태로 전환
```

### Hubble / DRA 적용 위치

Hubble과 DRA는 이번 업그레이드의 최종 목표에 포함될 수 있지만, 내일 첫 작업에서는 Kubernetes 안정화보다 앞세우지 않는다.

```text
Kubernetes 1.35.4 안정화
→ Cilium 상태 확인
→ Hubble relay/metrics only enable
→ DRA API 확인
→ NVIDIA DRA Driver 테스트 노드 적용
```

Hubble은 Cilium datapath 변경보다는 낮은 위험이지만, ConfigMap/agent rollout이 생길 수 있다. DRA는 Kubernetes 1.35 이후 API가 보이는지 확인한 뒤 테스트 노드에서만 시작한다.
