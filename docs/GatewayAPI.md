# Kubernetes Networking: Gateway API (Next-Gen Ingress)

The Gateway API is an open-source project managed by the Kubernetes SIG-Network community. It is a collection of API resources that model service networking in Kubernetes.

## Why Gateway API over Ingress?
- **Ingress**: A single resource where all traffic rules are combined. It lacks a clear separation between infrastructure and application developers and relies heavily on proprietary annotations.
- **Gateway API**: Designed for the separation of duties. It offers a more expressive and extensible model for traffic routing and management.

## Core Resource Models
The Gateway API defines three main roles and their corresponding resources:
1.  **GatewayClass (Infrastructure Provider)**: Defines the underlying network infrastructure (e.g., GCP LB, NGINX, Istio).
2.  **Gateway (Cluster Operator)**: Defines a physical or virtual entry point for traffic (IP, ports, TLS configuration).
3.  **HTTPRoute / GRPCRoute / TCPRoute (Application Developer)**: Defines the actual traffic routing rules.

## Key Features
- **Header-based Routing**: Advanced routing based on HTTP headers without custom code.
- **Weighted Traffic Splitting**: Built-in support for canary deployments and A/B testing (weights).
- **Cross-Namespace Routing**: Allows a single Gateway to route traffic to services across different namespaces securely.
- **Native Support for Service Mesh**: Seamlessly integrates with modern service mesh implementations.

## Practical Usage
Gateway API is becoming the standard for modern Kubernetes clusters. It allows cluster operators to manage the infrastructure while giving developers the flexibility to define their own routing rules.

---
*Reference: Gateway API Documentation (gateway-api.sigs.k8s.io)*
