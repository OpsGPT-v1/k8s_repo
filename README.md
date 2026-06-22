# OpsGPT Kubernetes Manifests

This directory contains static, dry-run-ready Kubernetes manifests for deploying the OpsGPT microservices stack to **Azure Kubernetes Service (AKS)**.

## Directory Structure

```text
kubernetes/
├── dev/
│   ├── namespace.yaml
│   ├── serviceaccounts.yaml
│   ├── secretproviderclasses.yaml
│   ├── deployments.yaml
│   ├── services.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
├── prod/
│   ├── namespace.yaml
│   ├── serviceaccounts.yaml
│   ├── secretproviderclasses.yaml
│   ├── deployments.yaml
│   ├── services.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
└── README.md
```

---

## Architecture Design Decisions

### 1. Target Platform & Registry
The deployments are designed for **Azure AKS** utilizing container images pulled from **Azure Container Registry (ACR)**. Static manifests use placeholder registry paths (`<ACR_NAME>.azurecr.io/opsgpt/*`) that must be substituted during deployment.

### 2. Environment & Namespace Strategy
The stack is segregated into two namespaces to align with standard development lifecycles:
*   `dev`: Replicas count of 2 per service. Uses image tag `:dev`.
*   `prod`: Replicas count of 3 per service. Uses image tag `:prod`.

### 3. Ingress Routing (AGIC)
The entry point reverse-proxy is handled by **Azure Application Gateway Ingress Controller (AGIC)**, replacing the local Nginx proxy. We configure `ingressClassName: azure-application-gateway` alongside annotations targeting the AGIC engine. Path routing is configured with `pathType: Prefix`:
*   `/` → `frontend-service:3000`
*   `/api/core` → `core-api-service:8001`
*   `/api/alerts` → `alert-ingestion-service:8002`
*   `/api/analysis` → `ai-analysis-service:8003`
*   `/api/notifications` → `notification-service:8004`
*   `/alerts/webhook` → `alert-ingestion-service:8002`

### 4. Workload Identity & ServiceAccounts
For improved security posture, we employ **Azure Workload Identity**:
*   Each of the 5 services is assigned its own dedicated `ServiceAccount` (e.g. `core-api-service-sa`).
*   Service accounts are annotated with `azure.workload.identity/client-id` pointing to their respective Azure Managed Identity Client ID.
*   Pods are labeled with `azure.workload.identity/use: "true"` to trigger the Workload Identity webhook injector.

### 5. Secrets Management (Azure Key Vault CSI)
To follow security best practices and prevent credentials from leaking into Git repositories:
*   **No static Secrets or ConfigMaps** are declared in these manifests.
*   Secrets (e.g., `DATABASE_URL`, `JWT_SECRET_KEY`, `INTERNAL_API_KEY`, etc.) are injected directly from **Azure Key Vault** using the **Secrets Store CSI Driver** and a `SecretProviderClass` configured per service.
*   Workload Identity is used to authorize the CSI driver pods to authenticate to Azure Key Vault without client secrets.
*   Pods mount the secret volume at `/mnt/secrets-store`. Key Vault secrets are also synced to standard Kubernetes secret objects under the hood, allowing FastAPI backends to consume them as environment variables via `valueFrom.secretKeyRef`.

### 6. Stateless Pods (No PV/PVC)
OpsGPT's microservices are strictly stateless. No `PersistentVolume` (PV) or `PersistentVolumeClaim` (PVC) manifests are created. The database layer is decoupled and targets a managed database service (e.g., Azure Database for PostgreSQL Flexible Server) in production.

### 7. Resource Allocations
To guarantee pod scheduling and resource boundary enforcement:
*   `requests`: CPU `100m`, Memory `128Mi`
*   `limits`: CPU `500m`, Memory `512Mi`

### 8. Autoscaling (HPA)
Each microservice includes a Horizontal Pod Autoscaler using `autoscaling/v2`:
*   **Dev**: Min Replicas `2`, Max Replicas `4`, Target CPU `70%`.
*   **Prod**: Min Replicas `3`, Max Replicas `10`, Target CPU `70%`.

### 9. Pod Security Context
In compliance with requirements, no `podSecurityContext` or container `securityContext` parameters (such as `runAsNonRoot`, `readOnlyRootFilesystem`, etc.) are declared in these templates.
*   **Reason**: The backend and frontend containers are already pre-configured to run as the non-root system user `appuser` (or standard `nginx` user) in their respective Dockerfiles. Redundant K8s-level overrides are avoided.

---

## Compatibility Warnings

> [!WARNING]
> **Health Probe Endpoints Compatibility**:
> Kubernetes manifests configure `livenessProbe` and `readinessProbe` to hit `/healthz` and `/ready` paths.
> However, the microservices code currently only exposes a `GET /health` endpoint.
> Prior to deploy, you must either:
> 1. Update the application source code routes to support `/healthz` and `/ready`.
> 2. Override the values or manually edit the paths in these manifests to target `/health`.

---

## Execution & Dry-Run Validation

Use the following commands to validate the manifests locally before triggering deployment pipelines.

### Dev Dry-run Validation
```bash
kubectl apply -f kubernetes/dev/ --dry-run=client
```

### Prod Dry-run Validation
```bash
kubectl apply -f kubernetes/prod/ --dry-run=client
```
