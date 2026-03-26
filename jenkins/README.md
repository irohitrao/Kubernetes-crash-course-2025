# Jenkins EKS Deployment Pipeline

Complete CI/CD pipeline for building, testing, and deploying applications to Amazon EKS using Jenkins and Ansible.

## Quick Start

### 1. Prerequisites

- EKS cluster running with required operators installed
- Jenkins server configured with Docker and AWS credentials
- Ansible installed on Jenkins server
- kubectl configured with EKS cluster access

### 2. Configure Jenkins

1. Create a new Pipeline job in Jenkins
2. Set SCM to your Git repository
3. Set Pipeline script path to `jenkins/Jenkinsfile`
4. Configure Jenkins credentials:
   - `aws-jenkins-credentials`: AWS IAM user credentials for ECR/EKS access
   - `kubeconfig-dev`: kubeconfig for dev cluster
   - `kubeconfig-stage`: kubeconfig for stage cluster
   - `kubeconfig-prod`: kubeconfig for prod cluster

### 3. Run Pipeline

Trigger the Jenkins pipeline with parameters:

**Development Deployment**:
```
Environment: dev
Image Tag: (auto-generated from commit SHA)
Skip Build: false
Skip Database: true
Run Bootstrap: false
```

**Production Deployment**:
```
Environment: prod
Image Tag: v1.2.3
Skip Build: false
Skip Database: true
Run Bootstrap: false
(Requires manual approval)
```

## Pipeline Overview

```
┌─────────────────────────────────────────────────────────────┐
│              Jenkins Pipeline Stages                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Validate Parameters                                    │
│     └─> Check environment, image tags, approvals          │
│                                                             │
│  2. Checkout Source                                        │
│     └─> Clone repository with app source                  │
│                                                             │
│  3. Build Images (Frontend, Auth, Game)                   │
│     └─> Build Docker images from application source       │
│                                                             │
│  4. Authenticate AWS                                       │
│     └─> Verify AWS credentials and permissions            │
│                                                             │
│  5. Push Images to ECR                                     │
│     └─> Push images to Amazon ECR registry                │
│                                                             │
│  6. Run Bootstrap Pipeline (Optional)                      │
│     └─> Run Ansible bootstrap for first-time setup        │
│                                                             │
│  7. Run Application Deployment                            │
│     └─> Run Ansible playbook for app release              │
│         ├─> Validate prerequisites                        │
│         ├─> Create namespace & secrets                    │
│         ├─> Deploy workloads                              │
│         ├─> Configure gateway & routing                   │
│         ├─> Setup monitoring                              │
│         └─> Validate deployment                           │
│                                                             │
│  8. Verify Deployment                                      │
│     └─> Check deployment status and access                │
│                                                             │
│  9. Archive Artifacts                                      │
│     └─> Save manifests and metadata                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Input Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ENVIRONMENT` | choice | - | Target environment (dev, stage, prod) |
| `IMAGE_TAG` | string | auto | Docker image tag (commit SHA recommended) |
| `SKIP_BUILD` | boolean | false | Skip Docker build, use existing images |
| `SKIP_DATABASE` | boolean | true | Skip database deployment |
| `RUN_BOOTSTRAP` | boolean | false | Run cluster bootstrap pipeline |
| `FORCE_ROLLOUT` | boolean | false | Force pod restart even if spec unchanged (kubectl rollout restart) |

## Output Artifacts

- Docker images pushed to Amazon ECR
- Kubernetes manifests rendered with environment-specific values
- Deployment metadata including image tags and timestamps
- Build logs and verification reports

## Environment Configuration

Each environment (dev, stage, prod) has specific configuration in `ansible/group_vars/` with **separate EKS clusters** and **shared application namespace**:

| Environment | Cluster Context | Namespace | Domain | Monitoring |
|-------------|-----------------|-----------|--------|-----------|
| dev | dev-eks | app | app-dev.example.com | disabled |
| stage | stage-eks | app | app-stage.example.com | enabled |
| prod | prod-eks | app | app.example.com | enabled |

**Architecture**: Cluster isolation provides environment separation. The shared namespace (`app`) simplifies deployment while kubeconfig context (`k8s_context`) ensures correct cluster targeting.

## Multi-Environment Support

The pipeline supports three environments with separate clusters:

### Development
- Dedicated EKS cluster (`dev-eks`)
- Fast iteration cycle
- No monitoring required
- Automatic deployment from branches

### Stage
- Dedicated EKS cluster (`stage-eks`)
- Production-like setup
- Monitoring enabled
- Manual deployment from release candidates

### Production
- Dedicated EKS cluster (`prod-eks`)
- High availability
- Full monitoring and observability
- Manual approval required before deployment

## Deployment Topology

```
┌──────────────────────────────────────────────────────────┐
│                    AWS Account                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │            Amazon EKS Cluster                    │  │
│  ├──────────────────────────────────────────────────┤  │
│  │                                                  │  │
│  │  ┌────────────────────────────────────────────┐ │  │
│  │  │  Kubernetes Namespace: production         │ │  │
│  │  ├────────────────────────────────────────────┤ │  │
│  │  │                                            │ │  │
│  │  │  ┌─────────────┐  ┌──────────┐  ┌──────┐  │ │  │
│  │  │  │   Frontend  │  │  Auth    │  │ Game │  │ │  │
│  │  │  │ Deployment  │  │Deployment│  │Deploy│  │ │  │
│  │  │  └─────────────┘  └──────────┘  └──────┘  │ │  │
│  │  │                                            │ │  │
│  │  │  ┌─────────────────────────────────────┐  │ │  │
│  │  │  │   PostgreSQL CloudNativePG         │  │ │  │
│  │  │  │   Cluster                          │  │ │  │
│  │  │  └─────────────────────────────────────┘  │ │  │
│  │  │                                            │ │  │
│  │  │  ┌──────────────────────────────────────┐ │ │  │
│  │  │  │  Gateway API / kgateway             │ │ │  │
│  │  │  │  ├─ HTTPRoute (routing rules)       │ │ │  │
│  │  │  │  ├─ TLS (cert-manager/Let's Encrypt)│ │ │  │
│  │  │  │  └─ Load Balancer (external IP)     │ │ │  │
│  │  │  └──────────────────────────────────────┘ │ │  │
│  │  │                                            │ │  │
│  │  │  ┌──────────────────────────────────────┐ │ │  │
│  │  │  │  Monitoring (ServiceMonitor)        │ │ │  │
│  │  │  │  └─ Prometheus scrape targets       │ │ │  │
│  │  │  └──────────────────────────────────────┘ │ │  │
│  │  │                                            │ │  │
│  │  └────────────────────────────────────────────┘ │  │
│  │                                                  │  │
│  └──────────────────────────────────────────────────┘  │
│                          ↑                              │
│                          │ kubectl apply               │
│                          │                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │        Amazon ECR (Container Registry)           │  │
│  │  ├─ frontend:<TAG>                              │  │
│  │  ├─ auth:<TAG>                                  │  │
│  │  └─ game:<TAG>                                  │  │
│  └──────────────────────────────────────────────────┘  │
│                          ↑                              │
│                          │ docker push                 │
│                          │                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │        Jenkins (CI/CD Orchestration)             │  │
│  │  └─ Build → Push → Deploy → Verify              │  │
│  └──────────────────────────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ↓
    ┌─────────────────────────┐
    │  Users Access via DNS   │
    │  https://app.example.com│
    └─────────────────────────┘
```

## File Structure

```
jenkins/
├── Jenkinsfile                          # Main pipeline definition
└── README.md                            # Jenkins documentation

ansible/
├── README.md                            # Ansible documentation
├── ansible.cfg                          # Ansible configuration
├── inventory.ini                        # Environment inventory
├── bootstrap-deploy.yml                 # Bootstrap playbook
├── app-deploy.yml                       # Application deployment playbook
├── rollback-deploy.yml                  # Rollback playbook
├── group_vars/
│   ├── dev.yml                          # Dev environment config
│   ├── stage.yml                        # Stage environment config
│   └── prod.yml                         # Prod environment config
└── roles/
    ├── prerequisites/                   # Pre-flight checks
    ├── namespace_and_secrets/           # Namespace and secrets
    ├── database/                        # Database deployment
    ├── workload_deployment/             # Workload deployment
    ├── gateway_and_routing/             # Gateway configuration
    ├── monitoring/                      # Monitoring setup
    └── validation/                      # Validation checks

manifests/
├── *.yaml                               # Kubernetes manifests
└── README.md                            # Manifest documentation
```

## Deployment Examples

### Deploy Development Build
```bash
# In Jenkins UI:
# - Environment: dev
# - Image Tag: (auto-generate)
# - Click "Build"
```

### Deploy Specific Version to Production
```bash
# In Jenkins UI:
# - Environment: prod
# - Image Tag: v1.2.3
# - Click "Build Now"
# - Approve deployment when prompted
```

### Rollback Production Deployment
```bash
# Using Ansible CLI:
cd ansible
ansible-playbook rollback-deploy.yml \
  -e "env_name=prod" \
  -e "previous_frontend_image_tag=abc123" \
  -e "previous_auth_image_tag=abc123" \
  -e "previous_game_image_tag=abc123"
```

### Force Rollout (Redeploy Without Spec Changes)
```bash
# In Jenkins UI - when image content changed but tag is same:
# - Environment: prod
# - Image Tag: abc123 (same as before)
# - Skip Build: true (or false)
# - FORCE_ROLLOUT: true  ← Enable pod restart
# - Click "Build Now"
```

This scenario occurs when:
- You rebuilt an image from the same commit (same tag)
- Manual redeployment needed without manifest changes
- Image pull policy should be `Always`

**Result**: All pods are restarted without modifying manifests.

## Status Checks

After deployment, verify:

```bash
# Check all deployments
kubectl get deployments -n production

# Check services
kubectl get services -n production

# Check gateway status
kubectl get gateway -n production

# Check certificate
kubectl get certificate -n production

# Check routes
kubectl get httproute -n production

# Check pod status
kubectl get pods -n production

# View deployment events
kubectl get events -n production --sort-by='.lastTimestamp'
```

## Troubleshooting

See [Ansible README](./ansible/README.md#monitoring-and-troubleshooting) for detailed troubleshooting guide.

## Next Steps

1. Review [Ansible documentation](./ansible/README.md)
2. Configure Jenkins credentials
3. Test deployment in dev environment
4. Promote to prod with approval workflow

## Support

For issues or questions, refer to:
- [Ansible Deployment Guide](./ansible/README.md)
- [Kubernetes Manifests](./manifests/README.md)
- Jenkins logs: `Jenkins UI → Logs Console`
- Deployment logs: `kubectl logs -f deployment/<name> -n <namespace>`
