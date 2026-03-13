# Kubernetes Security: ServiceAccount 및 RBAC 심층 가이드

쿠버네티스 클러스터 보안의 핵심은 권한 관리입니다. 본 가이드는 공식 문서를 바탕으로 ServiceAccount와 RBAC(Role-Based Access Control)의 실무 적용 방법을 다룹니다.

## 1. ServiceAccount (SA)
ServiceAccount는 포드에서 실행되는 프로세스가 쿠버네티스 API 서버와 통신할 때 사용하는 '신원(Identity)'입니다. 

### 실무 활용 시나리오
일반적으로 애플리케이션 포드는 외부와 통신하지만, 특정 관리형 포드(예: 모니터링 에이전트, CI/CD 러너)는 클러스터의 상태를 조회하거나 리소스를 생성해야 합니다. 이때 전 전용 SA를 생성하여 최소 권한 원칙을 준수해야 합니다.

### YAML 예시: ServiceAccount 생성
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-explorer-sa
  namespace: default
```

## 2. RBAC (Role-Based Access Control)
RBAC은 '누가(Subject)', '어디에서(Namespace)', '어떤 자원에(Resource)', '어떤 작업(Verb)'을 할 수 있는지 정의합니다.

### Role vs ClusterRole
- **Role**: 특정 네임스페이스에 한정된 권한을 정의합니다.
- **ClusterRole**: 클러스터 전체 자원(노드, 모든 네임스페이스의 포드 등)에 대한 권한을 정의합니다.

### YAML 예시: 특정 네임스페이스의 포드 조회 권한 (Role)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # ""은 코어 API 그룹을 의미함
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

### YAML 예시: SA와 Role 연결 (RoleBinding)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: api-explorer-sa # 위에서 생성한 SA
  namespace: default
roleRef:
  kind: Role
  name: pod-reader # 위에서 생성한 Role
  apiGroup: rbac.authorization.k8s.io
```

## 3. 포드에 ServiceAccount 적용하기
포드가 실행될 때 특정 SA의 권한을 가지도록 설정합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  serviceAccountName: api-explorer-sa
  containers:
  - name: app-container
    image: nginx
```

---
*참고: [Kubernetes 공식 문서 - RBAC 구성](https://kubernetes.io/docs/reference/access-authn/rbac/)*
