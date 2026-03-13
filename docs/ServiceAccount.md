# Kubernetes Security: ServiceAccount 및 RBAC 심층 가이드

ServiceAccount와 RBAC(Role-Based Access Control)은 쿠버네티스 클러스터 내의 보안 경계를 정의하는 가장 중요한 요소입니다. 본 문서는 공식 보안 권고안을 바탕으로 작성되었습니다.

## 1. ServiceAccount (SA) 상세 분석

### 개념 및 내부 작동 원리
ServiceAccount는 '사람'이 아닌 '포드(Pod)'에서 실행되는 프로세스에 부여되는 신원(Identity)입니다. 포드가 API 서버에 요청을 보낼 때, 이 SA를 통해 인증(Authentication)과 인가(Authorization) 과정을 거칩니다.

### 주요 특징
- **자동 토큰 마운트**: 포드 생성 시 SA를 지정하면 `/var/run/secrets/kubernetes.io/serviceaccount` 경로에 토큰, CA 인증서, 네임스페이스 정보가 자동으로 마운트됩니다.
- **보안 강화 (v1.24+ Bound Service Account Tokens)**: 이전처럼 Secret에 토큰을 무기한 저장하지 않고, `TokenRequest` API를 통해 발급된 시한부 토큰을 볼륨 주입(Projected Volume) 방식으로 사용합니다.

### 실무용 YAML 예시: 전용 ServiceAccount 생성
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor-bot-sa
  namespace: default
automountServiceAccountToken: true # 자동으로 토큰을 마운트할지 여부 (보안상 필요한 경우만 true)
```

## 2. RBAC (Role-Based Access Control) 정밀 설정

RBAC은 네 가지 리소스를 통해 권한을 제어합니다.

### A. Role vs ClusterRole
- **Role**: 특정 네임스페이스 내의 리소스에 대한 권한입니다.
- **ClusterRole**: 클러스터 전체 자원(Node, PV 등)이나 모든 네임스페이스의 자원을 제어할 때 사용합니다.

### B. Rule 정의 (API Groups, Resources, Verbs)
권한은 '어떤 그룹의 어떤 자원에 대해 어떤 행위를 할 것인가'로 정의됩니다.

**YAML 예시: 상세 권한 정의 (Role)**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: deployment-manager
rules:
- apiGroups: ["apps", "extensions"] # Deployment가 속한 API 그룹
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""] # 코어 리소스 (Pod, ConfigMap 등)
  resources: ["pods", "configmaps"]
  verbs: ["get", "list"]
```

### C. RoleBinding vs ClusterRoleBinding
- **RoleBinding**: 특정 네임스페이스 안에서 주체(SA 등)에게 Role을 부여합니다.
- **ClusterRoleBinding**: 클러스터 전체에 걸쳐 ClusterRole의 권한을 부여합니다.

**YAML 예시: SA와 Role 연결**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-manager-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: monitor-bot-sa
  namespace: default
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

## 3. 실무 보안 팁
1. **최소 권한 원칙**: `default` SA에 권한을 주지 말고, 항상 전용 SA를 만드세요.
2. **automountServiceAccountToken: false**: API 서버와 통신이 필요 없는 포드는 토큰 마운트를 비활성화하여 공격 표면을 줄이세요.
3. **ClusterRole 주의**: 모든 네임스페이스의 Secret을 읽을 수 있는 ClusterRole 부여는 매우 위험합니다.

---
*Reference: [Kubernetes Documentation - RBAC Authorization](https://kubernetes.io/docs/reference/access-authn/rbac/)*
