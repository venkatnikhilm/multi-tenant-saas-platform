# Multi-Tenant SaaS Platform

A cost-optimized, production-ready multi-tenant SaaS platform built on AWS EKS with Kubernetes namespace isolation and PostgreSQL database per tenant.

## Overview

This project demonstrates a complete multi-tenant architecture where each tenant gets:
- Isolated Kubernetes namespace with resource quotas
- Dedicated PostgreSQL database with separate credentials
- Subdomain-based routing (e.g., `acme-corp.yourplatform.local`)
- Automated provisioning via shell scripts

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     AWS Cloud                            │
│  ┌────────────────────────────────────────────────────┐ │
│  │  EKS Cluster (Spot Instances - 70% cost savings)   │ │
│  │                                                     │ │
│  │  ┌──────────────────┐  ┌──────────────────┐       │ │
│  │  │ tenant-acme-corp │  │ tenant-beta-corp │       │ │
│  │  │  - App Pods      │  │  - App Pods      │       │ │
│  │  │  - Resource Quota│  │  - Resource Quota│       │ │
│  │  │  - Network Policy│  │  - Network Policy│       │ │
│  │  └──────────────────┘  └──────────────────┘       │ │
│  │                                                     │ │
│  │  Ingress Controller (subdomain routing)            │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  RDS PostgreSQL (db.t3.micro)                      │ │
│  │  - acme_corp_db (acme_corp_user)                   │ │
│  │  - beta_corp_db (beta_corp_user)                   │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## Features

### Multi-Tenancy
- **Namespace Isolation**: Each tenant runs in a dedicated Kubernetes namespace
- **Resource Quotas**: CPU, memory, and pod limits per tenant
- **Database Isolation**: Separate PostgreSQL database per tenant
- **Subdomain Routing**: Tenant-specific URLs via Ingress

### Cost Optimization
- **Spot Instances**: 70% cost savings on EKS worker nodes
- **Right-Sized Resources**: t3.micro instances for development
- **Minimal Storage**: 20GB RDS with auto-scaling to 100GB
- **Single-AZ Deployment**: Reduced costs for non-production environments

### Tenant Resolution
Multiple methods to identify tenants:
1. Hostname-based: `acme-corp.yourplatform.local`
2. JWT token: `tenant_id` claim
3. Custom header: `X-Tenant-ID`

## Prerequisites

- AWS CLI configured with appropriate credentials
- Terraform >= 1.0
- kubectl
- Docker (for building application images)
- bash shell

## Quick Start

### 1. Deploy Infrastructure

```bash
# Deploy VPC
cd terraform/vpc
terraform init
terraform apply

# Deploy RDS
cd ../rds
terraform init
terraform apply

# Deploy EKS Cluster
cd ../eks
terraform init
terraform apply

# Configure kubectl
aws eks update-kubeconfig --name multi-tenant-cluster --region us-east-1
```

### 2. Provision a New Tenant

```bash
cd scripts
./provision-tenant.sh acme-corp mySecurePassword123
```

This script automatically:
- Creates a Kubernetes namespace with resource quotas
- Creates a PostgreSQL database and user
- Stores credentials in Kubernetes secrets
- Deploys the application
- Configures the service

### 3. Verify Deployment

```bash
# Check tenant resources
kubectl get all -n tenant-acme-corp

# Check resource quotas
kubectl describe resourcequota -n tenant-acme-corp

# Get service endpoint
kubectl get svc -n tenant-acme-corp
```

## Project Structure

```
.
├── app/
│   ├── server.js                    # Express application
│   ├── middleware/
│   │   └── tenant-resolver.js       # Tenant identification logic
│   ├── Dockerfile                   # Container image
│   └── package.json
├── kubernetes/
│   ├── deployments/                 # Per-tenant deployments
│   ├── namespaces/
│   │   └── tenant-template.yaml     # Namespace template with quotas
│   ├── ingress.yaml                 # Subdomain routing
│   ├── autoscaling/                 # HPA and cluster autoscaler
│   └── monitoring/                  # Cost tracking queries
├── terraform/
│   ├── vpc/                         # VPC with public subnets
│   ├── eks/                         # EKS cluster with spot instances
│   └── rds/                         # PostgreSQL database
└── scripts/
    └── provision-tenant.sh          # Automated tenant provisioning
```

## Configuration

### Resource Quotas (per tenant)

Default limits defined in `kubernetes/namespaces/tenant-template.yaml`:
- CPU: 500m requests, 1 core limit
- Memory: 1Gi requests, 2Gi limit
- Pods: Maximum 20
- Services: Maximum 10
- PVCs: Maximum 5

### Database Configuration

Each tenant gets:
- Dedicated database: `{tenant_id}_db`
- Dedicated user: `{tenant_id}_user`
- Credentials stored in Kubernetes secrets

### Application Environment Variables

- `TENANT_NAME`: Tenant identifier
- `DB_HOST`: RDS endpoint (from secret)
- `DB_NAME`: Database name (from secret)
- `PORT`: Application port (default: 80)

## Scaling

### Horizontal Pod Autoscaling

HPA configuration available in `kubernetes/autoscaling/hpa-config.yaml`:
- Scales based on CPU utilization
- Min replicas: 1
- Max replicas: 10

### Cluster Autoscaling

Cluster autoscaler automatically adjusts node count based on pod demands:
- Configured in `kubernetes/autoscaling/cluster-autoscaler.yaml`
- Works with spot instance node groups

## Cost Optimization Tips

1. **Use Spot Instances**: Already configured (70% savings)
2. **Right-size Resources**: Adjust instance types based on actual usage
3. **Enable Autoscaling**: Scale down during off-hours
4. **Monitor Costs**: Use queries in `kubernetes/monitoring/cost-queries.yaml`
5. **Single-AZ for Dev**: Multi-AZ only for production

## Security Considerations

- Database credentials stored in Kubernetes secrets
- Network policies isolate tenant namespaces
- RDS encryption at rest enabled
- VPC security groups restrict database access
- No public RDS access

## Monitoring

Access tenant-specific metrics:
```bash
# Pod metrics
kubectl top pods -n tenant-acme-corp

# Resource quota usage
kubectl describe resourcequota -n tenant-acme-corp

# Application logs
kubectl logs -n tenant-acme-corp -l app=acme-corp
```

## Adding a New Tenant

```bash
# Provision tenant
./scripts/provision-tenant.sh new-tenant-id securePassword

# Update ingress for new subdomain
kubectl apply -f kubernetes/ingress.yaml
```

## Cleanup

```bash
# Delete tenant
kubectl delete namespace tenant-acme-corp

# Destroy infrastructure (in reverse order)
cd terraform/eks && terraform destroy
cd ../rds && terraform destroy
cd ../vpc && terraform destroy
```

## License

MIT

## Contributing

Contributions welcome! Please open an issue or submit a pull request.
