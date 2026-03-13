# Kubernetes Resources: Dynamic Resource Allocation (DRA) 심층 가이드

Dynamic Resource Allocation (DRA)은 쿠버네티스 v1.26부터 도입된 차세대 자원 할당 모델입니다. 기존 'Device Plugins' 모델의 한계를 극복하고 NVIDIA GPU, FPGA 등 고성능 가속기 자원을 더욱 유연하게 관리할 수 있게 해줍니다.

## 1. DRA의 필요성과 이점
- **세밀한 제어**: 단순 개수 기반(예: GPU: 1) 할당이 아닌, 특정 성능 요구사항(VRAM 용량, CUDA 버전 등)을 구조화된 파라미터로 요청할 수 있습니다.
- **클레임 기반 관리**: 스토리지의 PVC 모델과 유사하게 자원을 미리 선점(Claim)하고 여러 포드가 공유하거나 특정 생명주기에 맞게 자원을 해제할 수 있습니다.

## 2. DRA 리소스 생성 실습 (v1.30+ 기준)

### ResourceClaimTemplate 생성
하드웨어 자원 요청을 위한 템플릿입니다.

```yaml
apiVersion: resource.k8s.io/v1alpha3
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim-template
spec:
  spec:
    resourceClassName: nvidia-gpu-class # 클러스터에 정의된 리소스 클래스
    parametersRef: # 세부 요구 사양 정의 가능
      kind: GpuParameters
      name: a100-profile
```

### 포드에서 자원 할당받기
포드 스펙에서 위 템플릿을 사용하여 자원을 동적으로 요청합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-pod
spec:
  containers:
  - name: training-container
    image: nvidia/cuda:12.0-base
    resources:
      claims:
      - name: my-gpu-resource # 아래의 resourceClaims 참조
  resourceClaims:
  - name: my-gpu-resource
    templateName: gpu-claim-template
```

---

# Kubernetes Storage: Persistent Volumes (PV/PVC) 심층 가이드

본 문서는 데이터 영속성을 보장하기 위한 쿠버네티스 스토리지 추상화 계층인 PV(Persistent Volume)와 PVC(Persistent Volume Claim)의 실무 활용법을 다룹니다.

## 1. PV/PVC의 작동 원리
인프라 관리자가 실제 스토리지(NFS, Ceph, Cloud EBS 등)를 **PV**로 등록해 두면, 개발자는 **PVC**라는 요청서를 통해 필요한 용량만큼의 스토리지를 할당받습니다.

## 2. 실전 스토리지 할당 (YAML 예시)

### A. StorageClass (동적 할당 설정)
클라우드나 Ceph 환경에서는 수동으로 PV를 만들지 않고 StorageClass를 통해 자동 할당합니다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd-storage
provisioner: kubernetes.io/aws-ebs # 클라우드 벤더의 프로비저너
parameters:
  type: gp3
reclaimPolicy: Retain # PVC 삭제 시 실제 데이터 보존 여부 결정 (Retain/Delete)
```

### B. PersistentVolumeClaim (사용자 요청)
실제 애플리케이션에서 사용할 저장 공간을 요청합니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce # 하나의 노드에서만 읽기/쓰기 가능
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd-storage # 위에서 만든 SC 지정
```

### C. Pod에서 볼륨 사용하기
포드 내의 특정 경로에 마운트합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
  - name: db-container
    image: postgres:15
    volumeMounts:
    - name: data-volume
      mountPath: /var/lib/postgresql/data # 컨테이너 내부 경로
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: database-pvc # 할당받은 PVC 이름
```

---
*참고: [Kubernetes 공식 문서 - 스토리지 관리](https://kubernetes.io/docs/concepts/storage/)*
