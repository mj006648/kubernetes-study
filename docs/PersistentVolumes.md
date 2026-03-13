# Kubernetes Resources: Dynamic Resource Allocation (DRA)

Dynamic Resource Allocation (DRA) is a new resource allocation model in Kubernetes (alpha since v1.26, evolving in v1.30+). It provides a more flexible way to request and allocate hardware resources like GPUs and FPGAs.

## Core Concepts
Unlike the traditional 'device plugins' model, DRA allows resources to be managed and allocated dynamically with more structured parameters.

### Why DRA?
- **More Expressive Requests**: Developers can specify exactly what hardware features they need (e.g., specific CUDA versions or GPU memory types).
- **Flexible Lifecycle**: Resources can be claimed and released independently of a Pod's lifecycle.
- **Resource Sharing**: It enables more efficient sharing of high-performance resources like NVIDIA GPUs.

## Major Components
- **ResourceClaim**: A request for a specific resource, similar to a PersistentVolumeClaim but for hardware accelerators.
- **ResourceClass**: Defines the parameters for a resource type (e.g., 'NVIDIA-GPU-A100').
- **ResourceClaimTemplate**: Defines a template for generating ResourceClaims.

## Implementation Details
DRA is the future of hardware acceleration in Kubernetes. It allows for smarter scheduling and better resource utilization, especially for AI/ML workloads.

---
*Reference: Kubernetes Official Blog - v1.30: Uwubernetes*

---

# Kubernetes Storage: Persistent Volumes and Claims

Kubernetes provides a powerful abstraction layer for managing persistent storage.

## Key Resources
- **PersistentVolume (PV)**: A piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes.
- **PersistentVolumeClaim (PVC)**: A request for storage by a user (developer).
- **StorageClass**: Allows administrators to describe the "classes" of storage they offer.

## Why Use PV/PVC?
- **Abstraction**: Developers don't need to know the underlying storage infrastructure (NFS, EBS, Ceph, etc.).
- **Lifecycle Management**: Storage remains persistent even if the Pod is deleted or rescheduled.

## Dynamic Provisioning
Most modern clusters use 'StorageClasses' for dynamic provisioning. When a PVC is created, the StorageClass automatically creates a PV and the underlying cloud/hardware storage.

---
*Reference: Kubernetes Official Documentation - Storage*
