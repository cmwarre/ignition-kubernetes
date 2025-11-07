# Ignition on Kubernetes

A complete Kubernetes deployment for Inductive Automation's Ignition SCADA platform with PostgreSQL database backend.

## Overview

This repository contains Kubernetes manifests to deploy:
- **Ignition Gateway** (8.1) - SCADA platform
- **PostgreSQL** (15-alpine) - Database backend
- **Traefik IngressRoute** - External access via `my-new-ignition.k8s.local`
- **Prometheus ServiceMonitor** - Metrics collection
- **ArgoCD Application** - GitOps deployment

## Architecture

```
├── k8s/
│   ├── namespace.yaml              # Ignition namespace
│   ├── ingress.yaml                # Traefik IngressRoute
│   ├── monitoring.yaml             # Prometheus ServiceMonitor
│   ├── postgresql/
│   │   ├── secret.yaml             # Database credentials
│   │   ├── pvc.yaml                # 10Gi persistent storage
│   │   ├── deployment.yaml         # PostgreSQL deployment
│   │   └── service.yaml            # ClusterIP service
│   └── ignition/
│       ├── pvc.yaml                # 5Gi persistent storage
│       ├── deployment.yaml         # Ignition deployment
│       └── service.yaml            # ClusterIP service
└── argocd/
    └── application.yaml            # ArgoCD application manifest
```

## Prerequisites

- Kubernetes cluster with:
  - **Traefik** ingress controller
  - **Prometheus Operator** (for ServiceMonitor)
  - **ArgoCD** (optional, for GitOps)
  - **CoreDNS** configured for `.k8s.local` domain
- `kubectl` configured to access your cluster
- Git and `gh` CLI (for repository setup)

## Quick Start

### Option 1: Manual Deployment with kubectl

1. **Apply all manifests:**
   ```bash
   kubectl apply -f k8s/namespace.yaml
   kubectl apply -f k8s/postgresql/
   kubectl apply -f k8s/ignition/
   kubectl apply -f k8s/ingress.yaml
   kubectl apply -f k8s/monitoring.yaml
   ```

2. **Verify deployment:**
   ```bash
   kubectl get pods -n ignition
   kubectl get pvc -n ignition
   kubectl get svc -n ignition
   ```

3. **Check Ignition logs:**
   ```bash
   kubectl logs -n ignition -l app=ignition -f
   ```

### Option 2: GitOps Deployment with ArgoCD

1. **Update the repository URL** in `argocd/application.yaml`:
   ```yaml
   repoURL: https://github.com/YOUR_USERNAME/ignition-kubernetes.git
   ```

2. **Apply the ArgoCD Application:**
   ```bash
   kubectl apply -f argocd/application.yaml
   ```

3. **Monitor sync status:**
   ```bash
   kubectl get application -n argocd ignition
   ```
   
   Or via ArgoCD UI:
   ```bash
   # Port-forward to ArgoCD (if not exposed via ingress)
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   # Navigate to https://localhost:8080
   ```

## Accessing Ignition

### Configure DNS/Hosts

Add the following to your `/etc/hosts` file (or configure CoreDNS):
```
<TRAEFIK_INGRESS_IP>  my-new-ignition.k8s.local
```

To find your Traefik ingress IP:
```bash
kubectl get svc -n traefik  # Look for LoadBalancer or NodePort service
```

### Access the Gateway

- **HTTP:** http://my-new-ignition.k8s.local
- **HTTPS:** https://my-new-ignition.k8s.local

Default credentials (configured in `k8s/ignition/deployment.yaml`):
- **Username:** admin
- **Password:** password

**⚠️ IMPORTANT:** Change default passwords in production!

## Database Configuration

Connect Ignition to PostgreSQL:

1. Navigate to **Config > Databases** in the Ignition Gateway
2. Create a new database connection:
   - **Name:** ignition-db
   - **Driver:** PostgreSQL
   - **Connection URL:** `jdbc:postgresql://postgresql.ignition.svc.cluster.local:5432/ignition`
   - **Username:** ignition
   - **Password:** ignition123

Database credentials are stored in `k8s/postgresql/secret.yaml`.

## Monitoring

### Prometheus Integration

The ServiceMonitor (`k8s/monitoring.yaml`) is configured to scrape Ignition metrics on port `9088`.

Verify the ServiceMonitor:
```bash
kubectl get servicemonitor -n ignition
```

### View Metrics in Grafana

1. Access your Grafana instance
2. Add queries for Ignition metrics (namespace: `ignition`)
3. Create dashboards for:
   - Gateway health
   - Thread usage
   - Database connections
   - Tag metrics

## Persistence

Two PersistentVolumeClaims are created:

| PVC | Size | Purpose |
|-----|------|---------|
| `postgresql-pvc` | 10Gi | PostgreSQL data |
| `ignition-data-pvc` | 5Gi | Ignition gateway data |

**Note:** Adjust storage sizes in the PVC manifests based on your requirements.

## Updating Configuration

### Update PostgreSQL Password

1. Edit `k8s/postgresql/secret.yaml`
2. Update the `POSTGRES_PASSWORD` value
3. Apply changes:
   ```bash
   kubectl apply -f k8s/postgresql/secret.yaml
   kubectl rollout restart deployment/postgresql -n ignition
   ```

### Scale Ignition (Not Recommended)

Ignition gateway is stateful and typically runs as a single instance. If you need high availability, consider Ignition's built-in redundancy features rather than horizontal scaling.

### Update Ignition Version

Edit `k8s/ignition/deployment.yaml`:
```yaml
image: inductiveautomation/ignition:8.1.X  # Update version
```

Apply and restart:
```bash
kubectl apply -f k8s/ignition/deployment.yaml
kubectl rollout restart deployment/ignition -n ignition
```

## Troubleshooting

### Pods not starting

```bash
kubectl describe pod -n ignition <pod-name>
kubectl logs -n ignition <pod-name>
```

### Database connection issues

Verify PostgreSQL is ready:
```bash
kubectl exec -it -n ignition deployment/postgresql -- psql -U ignition -d ignition -c '\l'
```

### Ingress not working

Check Traefik IngressRoute:
```bash
kubectl get ingressroute -n ignition
kubectl describe ingressroute ignition-http -n ignition
```

Verify Traefik can route to the service:
```bash
kubectl get svc -n ignition ignition
```

### ArgoCD sync issues

Check application status:
```bash
kubectl get application -n argocd ignition -o yaml
```

View sync errors in ArgoCD UI or:
```bash
argocd app get ignition
```

## Cleanup

### Remove via kubectl

```bash
kubectl delete -f k8s/
```

### Remove via ArgoCD

```bash
kubectl delete -f argocd/application.yaml
# Or via argocd CLI:
argocd app delete ignition
```

## Security Considerations

⚠️ **Before deploying to production:**

1. **Change default passwords:**
   - Ignition admin password (in `k8s/ignition/deployment.yaml`)
   - PostgreSQL password (in `k8s/postgresql/secret.yaml`)

2. **Use Kubernetes Secrets properly:**
   - Consider using sealed-secrets or external secret managers
   - Don't commit sensitive secrets to git

3. **Enable TLS:**
   - Configure Traefik with proper TLS certificates
   - Use cert-manager for automated certificate management

4. **Network policies:**
   - Restrict traffic between pods
   - Limit external access

5. **Resource limits:**
   - Adjust CPU/memory limits based on your workload
   - Set up monitoring and alerts

## License

This deployment configuration is provided as-is. Ignition is a commercial product by Inductive Automation - ensure you have proper licensing before use.

## Support

For Ignition-specific questions, visit:
- [Inductive Automation Forums](https://forum.inductiveautomation.com/)
- [Ignition Documentation](https://docs.inductiveautomation.com/)

For Kubernetes deployment questions, please open an issue in this repository.
