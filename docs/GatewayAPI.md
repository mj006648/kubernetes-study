# Kubernetes Networking: Gateway API 심층 분석 가이드

Gateway API는 기존 Ingress의 한계를 극복하기 위해 설계된 쿠버네티스의 차세대 서비스 네트워킹 표준입니다. 본 문서는 공식 명세(v1.0+)를 기반으로 실무 중심의 활용법을 다룹니다.

## 1. Ingress와 Gateway API의 차이점
기존 Ingress는 하나의 리소스 파일에 트래픽 규칙과 인프라 설정을 모두 담아야 했지만, Gateway API는 이를 분리하여 다중 팀(인프라 팀, 서비스 팀) 간의 협업을 최적화합니다.

- **표현력**: HTTP 헤더 기반 라우팅, 가중치 기반 트래픽 분산 등을 네이티브하게 지원합니다.
- **역할 분리**: 인프라 담당자는 `Gateway`를 생성하고, 애플리케이션 개발자는 `HTTPRoute`만 작성하면 됩니다.

## 2. Gateway API 구현 실습 (YAML 예시)

### A. Gateway 생성 (인프라/운영자 역할)
클러스터의 외부 진입점(Entry Point)을 정의합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gateway
  namespace: infrastructure-ns
spec:
  gatewayClassName: nginx-gateway-class # 환경에 맞는 GatewayClass 지정
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All # 모든 네임스페이스의 Route를 허용할 것인지 정의
```

### B. HTTPRoute 생성 (애플리케이션 개발자 역할)
실제 트래픽을 서비스로 전달하는 라우팅 규칙을 정의합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-routing
  namespace: app-ns
spec:
  parentRefs:
  - name: external-gateway # 위에서 생성한 Gateway 연결
    namespace: infrastructure-ns
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1/api # 경로 기반 라우팅
    backendRefs:
    - name: v1-service # 실제 백엔드 서비스
      port: 8080
      weight: 90 # 트래픽 가중치 (90%)
    - name: canary-service # 카나리 배포용 서비스
      port: 8080
      weight: 10 # 트래픽 가중치 (10%)
```

## 3. 실무에서의 활용 (Cross-Namespace)
Gateway API의 강력한 기능 중 하나는 네임스페이스를 넘나드는 라우팅입니다. 인프라 팀이 중앙 네임스페이스에서 `Gateway`를 관리하고, 각 서비스 팀은 자신의 네임스페이스에서 `HTTPRoute`만 관리하여 트래픽 진입로를 독립적으로 운영할 수 있습니다.

---
*참고: [Gateway API 공식 문서](https://gateway-api.sigs.k8s.io/)*
