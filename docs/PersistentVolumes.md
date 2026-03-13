# Kubernetes Storage: Persistent Volumes (PV) & Claims (PVC) 정밀 가이드

쿠버네티스 스토리지는 데이터의 영속성(Persistence)을 보장하기 위해 실제 저장소 인프라를 추상화합니다.

## 1. 핵심 개념: PV, PVC, 그리고 StorageClass

- **PersistentVolume (PV)**: 클러스터 관리자가 확보해 둔 실제 저장 공간(NFS, Ceph, Cloud Disk 등). 노드와는 별개의 생명주기를 가집니다.
- **PersistentVolumeClaim (PVC)**: 사용자가 "이 정도 용량과 성능의 스토리지가 필요하다"라고 요청하는 명세서입니다.
- **StorageClass (SC)**: 관리자가 스토리지의 종류(예: 'Fast SSD', 'Cheap HDD')를 정의해 두면, PVC 요청 시 자동으로 PV가 생성(Dynamic Provisioning)되도록 돕는 템플릿입니다.

## 2. 정교한 YAML 예시 및 속성 설명

### A. StorageClass (동적 할당 정의)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd
provisioner: kubernetes.io/aws-ebs # 벤더 프로비저너
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Retain # 중요: PVC 삭제 시 데이터를 보존(Retain)할지 삭제(Delete)할지 결정
volumeBindingMode: WaitForFirstConsumer # 포드가 스케줄링된 노드의 가용 영역에 맞춰 볼륨 생성
```

### B. PersistentVolumeClaim (사용자 스토리지 요청)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce # 한 노드에서만 R/W (대부분의 블록 스토리지)
    # - ReadWriteMany # 여러 노드에서 동시 R/W (NFS 등 공유 파일시스템)
  resources:
    requests:
      storage: 50Gi
  storageClassName: premium-ssd
```

### C. 실전 활용: Deployment에 볼륨 마운트
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        volumeMounts:
        - name: storage-vol
          mountPath: /data # 컨테이너 내부 경로
      volumes:
      - name: storage-vol
        persistentVolumeClaim:
          claimName: app-data-pvc
```

## 3. 실무 운영 시 주의사항
- **AccessModes**: 사용하려는 물리 스토리지 솔루션이 지원하는 모드인지 반드시 확인해야 합니다. (예: EBS는 RWX 지원 불가)
- **Expand Persistent Volumes**: `allowVolumeExpansion: true` 설정이 SC에 있으면 서비스 중단 없이 용량 증설이 가능합니다.

---
*Reference: [Kubernetes Documentation - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)*
