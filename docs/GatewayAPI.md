# Kubernetes Networking: Gateway API 정밀 가이드

Gateway API는 쿠버네티스 서비스 네트워킹을 표현력이 풍부하고 유연하며 역할 중심적으로 발전시키기 위한 오픈소스 프로젝트입니다.

## 1. 핵심 철학: 역할 기반 분리 (Separation of Concerns)
기존 Ingress는 개발자가 인프라 설정까지 건드려야 했으나, Gateway API는 세 가지 역할로 나뉩니다.
- **인프라 제공자**: `GatewayClass`를 통해 어떤 L7 로드밸런서를 쓸지 결정.
- **클러스터 운영자**: `Gateway`를 통해 실제 도메인, 포트, 인증서 설정.
- **애플리케이션 개발자**: `HTTPRoute`를 통해 자신의 서비스로 트래픽을 라우팅하는 규칙만 작성.

## 2. 상세 리소스 구성 및 YAML 예시

### A. GatewayClass (네트워크 인프라 정의)
클라우드 벤더(AWS, GCP)나 설치형(NGINX, Istio) 컨트롤러를 정의합니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: prod-lb-class
spec:
  controllerName: example.com/gateway-controller
```

### B. Gateway (트래픽 진입점 설정)
실제 외부 IP가 할당되는 지점입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: infra-ns
spec:
  gatewayClassName: prod-lb-class
  listeners:
  - name: https-listener
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: site-cert-secret
    allowedRoutes: # 보안: 어떤 네임스페이스의 Route만 허용할지 정의
      namespaces:
        from: Selector
        selector:
          matchLabels:
            env: production
```

### C. HTTPRoute (정밀한 라우팅 규칙)
가장 많이 쓰이는 라우팅 설정입니다.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: user-service-route
  namespace: user-app-ns
spec:
  parentRefs: # 어떤 Gateway에 붙을지 명시
  - name: main-gateway
    namespace: infra-ns
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1/users
    filters: # 헤더 조작 등 복잡한 필터링 가능
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: x-service-id
          value: user-v1
    backendRefs:
    - name: user-svc-v1
      port: 8080
```

## 3. Gateway API만의 차별화 기능
- **가중치 기반 분산**: 별도 메쉬 도구 없이도 `backendRefs`에서 `weight`를 주어 카나리 배포 가능.
- **조건부 라우팅**: HTTP 메소드, 헤더, 쿼리 파라미터 기반의 정교한 매칭 지원.
- **공식 표준**: 특정 벤더의 Annotation 지옥에서 벗어나 표준 API로 인프라 이동이 자유로움.

---
*Reference: [Gateway API Specification](https://gateway-api.sigs.k8s.io/)*
