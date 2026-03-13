# Kubernetes Security: ServiceAccount and RBAC

ServiceAccount and Role-Based Access Control (RBAC) are the core mechanisms for managing identities and permissions within a Kubernetes cluster.

## ServiceAccount (SA)
A ServiceAccount provides an identity for processes that run in a Pod. When you (a human) access the cluster, the API server authenticates you as a particular User Account. Processes in containers inside pods can also contact the API server. When they do, they are authenticated as a particular ServiceAccount (for example, default).

### Key Characteristics
- **Namespaced**: ServiceAccounts are scoped to a specific namespace.
- **Default SA**: Every namespace has a 'default' ServiceAccount created automatically.
- **Token Projection (v1.24+)**: Since Kubernetes v1.24, ServiceAccount tokens are no longer stored in Secrets permanently. Instead, they are projected into the Pod via the TokenRequest API with a limited lifetime.

## Role-Based Access Control (RBAC)
RBAC is a method of regulating access to computer or network resources based on the roles of individual users within an enterprise.

### Core Resources
- **Role**: Defines permissions within a specific namespace.
- **ClusterRole**: Defines permissions across the entire cluster (e.g., node management).
- **RoleBinding**: Grants the permissions defined in a Role to a user or ServiceAccount within a specific namespace.
- **ClusterRoleBinding**: Grants the permissions defined in a ClusterRole to a user or ServiceAccount cluster-wide.

## Practical Implementation
To allow a Pod to list other Pods in its namespace:
1. Create a ServiceAccount.
2. Create a Role with 'list' and 'get' verbs for the 'pods' resource.
3. Create a RoleBinding to link the ServiceAccount and the Role.
4. Specify the 'serviceAccountName' in the Pod's specification.

---
*Reference: Kubernetes Official Documentation - Service Accounts*
