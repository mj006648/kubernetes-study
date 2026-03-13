# Kubernetes Resources: Dynamic Resource Allocation (DRA) 정밀 가이드

Dynamic Resource Allocation (DRA)은 GPU, FPGA 등 특수 하드웨어 가속기를 관리하기 위한 쿠버네티스의 차세대 리소스 할당 모델입니다.

## 1. DRA의 핵심 메커니즘
기존의 `nvidia.com/gpu: 1` 같은 단순 방식에서 벗어나, 하드웨어의 미세한 속성(VRAM 용량, 특정 하드웨어 버전 등)을 기반으로 자원을 요청하고 할당받습니다.

- **ResourceClaim**: 스토리지의 PVC와 유사하게 특정 하드웨어 자원을 예약합니다.
- **ResourceClass**: 자원의 종류와 할당 방식을 정의하는 추상화 계층입니다.

## 2. 상세 YAML 구현 예시

### A. ResourceClass 정의
클러스터에 어떤 하드웨어 풀이 있는지 정의합니다.

```yaml
apiVersion: resource.k8s.io/v1alpha3
kind: ResourceClass
metadata:
  name: nvidia-gpu-a100
driverName: gpu-driver.nvidia.com # 해당 하드웨어를 관리할 드라이버
parametersRef:
  kind: GpuClassParameters
  name: default-parameters
```

### B. ResourceClaim (개별 포드의 자원 요청)
특정 포드가 사용할 자원 명세입니다.

```yaml
apiVersion: resource.k8s.io/v1alpha3
kind: ResourceClaim
metadata:
  name: ai-training-gpu-claim
spec:
  resourceClassName: nvidia-gpu-a100
  request: # 세부 요구 사양 (드라이버가 해석)
    vram: "40Gi"
    cuda_version: "12.2"
```

### C. Pod 스펙에서 DRA 리소스 참조
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:12.2-base
    resources:
      claims:
      - name: gpu-resource # 아래의 resourceClaims 항목 참조
  resourceClaims:
  - name: gpu-resource
    source:
      resourceClaimName: ai-training-gpu-claim # 위에서 생성한 Claim 연결
```

## 3. 실무에서의 기대 효과
DRA를 사용하면 대규모 GPU 클러스터에서 자원 파편화를 방지하고, 특정 워크로드에 꼭 필요한 사양의 하드웨어를 정확하게 배정할 수 있어 **비용 최적화 및 성능 극대화**가 가능합니다.

---
*Reference: [Kubernetes Documentation - Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)*
