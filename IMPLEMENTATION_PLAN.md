# Jenkins Ansible EKS Deployment Pipeline - Implementation Plan

This document outlines the complete implementation of the Jenkins-Ansible-EKS deployment pipeline based on the detailed planning document.

## Implementation Status

✅ **Complete** - All pipeline components have been implemented

## Directory Structure Created

```
Kubernetes-crash-course-2025/
├── ansible/                             # Ansible deployment automation
│   ├── inventory.ini                    # Multi-environment inventory
│   ├── ansible.cfg                      # Ansible configuration
│   ├── bootstrap-deploy.yml            # One-time cluster setup
│   ├── app-deploy.yml                  # Application release pipeline
│   ├── rollback-deploy.yml             # Rollback to previous versions
│   ├── group_vars/
│   │   ├── dev.yml                     # Development environment config
│   │   ├── stage.yml                   # Staging environment config
│   │   └── prod.yml                    # Production environment config
│   ├── roles/
│   │   ├── prerequisites/              # Validate cluster prerequisites
│   │   ├── namespace_and_secrets/      # Namespace and credential setup
│   │   ├── database/                   # PostgreSQL deployment
│   │   ├── workload_deployment/        # Microservices deployment
│   │   ├── gateway_and_routing/        # Ingress and TLS setup
│   │   ├── monitoring/                 # Monitoring integration
│   │   └── validation/                 # Post-deployment verification
│   └── README.md                        # Complete Ansible documentation
│
└── jenkins/                             # Jenkins CI/CD configuration
    ├── Jenkinsfile                      # Pipeline definition
    └── README.md                        # Jenkins setup guide
```

## Components Implemented

### 1. Ansible Playbooks

#### Bootstrap Pipeline (`bootstrap-deploy.yml`)
**Purpose**: One-time cluster setup (cluster prerequisites, database initialization)

**Stages**:
1. Prerequisites validation (kubectl, AWS, ECR, CRDs)
2. Namespace creation
3. PostgreSQL credentials secret creation
4. PostgreSQL cluster deployment
5. Database readiness verification

**Usage**:
```bash
ansible-playbook -i ansible/inventory.ini ansible/bootstrap-deploy.yml \
  -e "env_name=prod"
```

#### Application Deployment Playbook (`app-deploy.yml`)
**Purpose**: Repeatable application releases

**Stages**:
1. Prerequisites validation
2. Namespace and secret management
3. Workload deployment (auth, game, frontend)
4. Gateway and HTTPRoute configuration
5. TLS certificate setup
6. Monitoring integration (ServiceMonitor)
7. Post-deployment validation

**Usage**:
```bash
ansible-playbook -i ansible/inventory.ini ansible/app-deploy.yml \
  -e "env_name=prod" \
  -e "frontend_image_tag=abc123" \
  -e "auth_image_tag=abc123" \
  -e "game_image_tag=abc123"
```

#### Rollback Playbook (`rollback-deploy.yml`)
**Purpose**: Safe rollback to previous versions

**Usage**:
```bash
ansible-playbook -i ansible/inventory.ini ansible/rollback-deploy.yml \
  -e "env_name=prod" \
  -e "previous_frontend_image_tag=old_sha" \
  -e "previous_auth_image_tag=old_sha" \
  -e "previous_game_image_tag=old_sha"
```

### 2. Ansible Roles

| Role | Purpose | Key Tasks |
|------|---------|-----------|
| `prerequisites` | Validate cluster readiness | Check kubectl access, AWS auth, CRDs |
| `namespace_and_secrets` | Setup namespace and credentials | Create namespace, database secret |
| `database` | Deploy PostgreSQL | Apply cluster, configmap, wait for readiness |
| `workload_deployment` | Deploy services | Render manifests with image tags, apply, verify |
| `gateway_and_routing` | Configure ingress | Apply gateway, routes, TLS, wait for readiness |
| `monitoring` | Setup metrics collection | Deploy ServiceMonitor if Prometheus available |
| `validation` | Verify deployment health | Check deployment status, endpoints, gateway |

### 3. Environment Configuration

Multi-environment support with **separate EKS clusters** and **shared application namespace**:

**Architecture**: Each environment has its own cluster (`dev-eks`, `stage-eks`, `prod-eks`) for isolation, but all deploy to the same namespace (`app`) since cluster separation provides environment separation.

**Development** (`group_vars/dev.yml`):
- Namespace: `app` (shared across environments)
- Cluster Context: `dev-eks`
- Domain: `app-dev.example.com`
- Monitoring: Disabled
- Timeouts: Standard

**Staging** (`group_vars/stage.yml`):
- Namespace: `app` (shared across environments)
- Cluster Context: `stage-eks`
- Domain: `app-stage.example.com`
- Monitoring: Enabled
- Timeouts: Standard

**Production** (`group_vars/prod.yml`):
- Namespace: `app` (shared across environments)
- Cluster Context: `prod-eks`
- Domain: `app.example.com`
- Monitoring: Enabled
- Timeouts: Extended for stability

**Key Benefits**:
- Cluster isolation provides strong environment separation
- Single namespace simplifies deployment logic
- Same manifests deploy to all clusters
- No namespace creation overhead per environment

### 4. Jenkins Pipeline (`jenkins/Jenkinsfile`)

Complete CI/CD pipeline with following stages:

1. **Validate Parameters** - Confirm environment and inputs
2. **Checkout Source** - Clone repository
3. **Build Images** - Docker build for frontend, auth, game
4. **Authenticate AWS** - Setup AWS credentials
5. **Push Images to ECR** - Push to Amazon ECR registry
6. **Run Bootstrap Pipeline** - Optional cluster setup
7. **Run Application Deployment** - Execute Ansible deployment
8. **Verify Deployment** - Health checks and validation
9. **Archive Artifacts** - Save manifests and metadata

**Jenkins Parameters**:
- `ENVIRONMENT`: dev, stage, or prod
- `IMAGE_TAG`: Docker image tag (auto-generated from commit SHA)
- `SKIP_BUILD`: Skip building images (use existing)
- `SKIP_DATABASE`: Skip database deployment
- `RUN_BOOTSTRAP`: Run cluster bootstrap pipeline

## Phase Mapping to Implementation

### Phase 1: Define Pipeline Boundaries ✅
- **Implemented in**: Jenkinsfile, app-deploy.yml
- **Details**: Separated bootstrap (prerequisites) from application release
- **Result**: Two distinct pipelines with clear responsibilities

### Phase 2: Organize Repository and Pipeline Inputs ✅
- **Implemented in**: ansible/group_vars/*, Jenkinsfile
- **Details**: Environment-specific variables in group_vars/
- **Result**: Single deployment logic, environment differences externalized

### Phase 3: Build and Publish Images ✅
- **Implemented in**: Jenkinsfile stages 2-5
- **Details**: Build Docker images, push to ECR
- **Result**: Image tags captured and passed to deployment

### Phase 4: Prepare Ansible Deployment Logic ✅
- **Implemented in**: ansible/roles/*, ansible/*.yml
- **Details**: Multiple roles for different deployment concerns
- **Result**: Modular, reusable deployment automation

### Phase 5: Readiness Gates and Rollout Checks ✅
- **Implemented in**: workload_deployment, gateway_and_routing, validation roles
- **Details**: Wait conditions, status checks, timeout handling
- **Result**: Safe deployments with verification

### Phase 6: Multi-Environment Support ✅
- **Implemented in**: group_vars/*, Jenkinsfile parameters
- **Details**: One pipeline, multiple environments
- **Result**: Consistent deployments across dev/stage/prod

### Phase 7: Rollback and Repeatability ✅
- **Implemented in**: rollback-deploy.yml, app-deploy.yml design
- **Details**: Idempotent playbooks, previous tag support
- **Result**: Safe rollback mechanism, repeatable deployments

### Phase 8: Verification and Reporting ✅
- **Implemented in**: validation role, Jenkins post-tasks, archive artifacts
- **Details**: Comprehensive health checks, artifact preservation
- **Result**: Traced deployments with detailed reporting

## Quick Start Guide

### 1. Prerequisites Setup

```bash
# Verify EKS cluster access
aws eks update-kubeconfig --name <cluster-name> --region us-east-1
kubectl get nodes

# Verify required operators are installed
kubectl get crd | grep -E "cert-manager|postgresql|gateway"

# Create ECR repositories
aws ecr create-repository --repository-name frontend
aws ecr create-repository --repository-name auth
aws ecr create-repository --repository-name game
```

### 2. Configure Environments

Edit `ansible/group_vars/` files to set:
- AWS account IDs
- Kubernetes namespaces
- Application domains
- Database credentials
- Feature flags

### 3. Setup Jenkins

1. Create Jenkins Pipeline job pointing to `jenkins/Jenkinsfile`
2. Add credentials:
   - `aws-jenkins-credentials` - AWS IAM user
   - `kubeconfig-dev`: dev cluster kubeconfig
   - `kubeconfig-stage`: stage cluster kubeconfig
   - `kubeconfig-prod`: prod cluster kubeconfig
3. Enable CloudBees or similar for approval gates (production)

### 4. Test Deployment

**Development (fully automated)**:
```
Environment: dev
Image Tag: (auto-generate)
Skip Build: false
→ Builds, pushes, and deploys automatically
```

**Production (manual approval)**:
```
Environment: prod
Image Tag: v1.2.3
Skip Build: false
→ Builds, pushes, waits for manual approval, then deploys
```

## Key Design Decisions

### 1. Separated Bootstrap from Application Release
- **Rationale**: Cluster prerequisites rarely change; app updates are frequent
- **Benefit**: Safer, faster, focused deployment pipelines

### 2. Single Deployment Logic, Environment Variables
- **Rationale**: Avoid manifest duplication, reduce maintenance burden
- **Benefit**: Consistent deployments, easier to maintain

### 3. Idempotent Playbooks
- **Rationale**: Re-running should be safe (no accidental deletions)
- **Benefit**: Safe rollback, repeatable deployments

### 4. Manifest Templating via CLI
- **Rationale**: Simple, no additional tools needed
- **Benefit**: Easy debugging, visible in logs

### 5. Image Tags as Primary Rollback Mechanism
- **Rationale**: Database-safe (no schema rollback complexity)
- **Benefit**: Quick, predictable rollback

## Validation Checklist

Before production deployment:

- [ ] EKS cluster accessible with kubectl
- [ ] AWS ECR repositories created and accessible
- [ ] cert-manager installed (`kubectl get crd certificates.cert-manager.io`)
- [ ] CloudNativePG operator installed (`kubectl get crd clusters.postgresql.cnpg.io`)
- [ ] Gateway API installed (`kubectl get crd gateways.gateway.networking.k8s.io`)
- [ ] Ansible installed locally (`ansible --version`)
- [ ] Jenkins configured with required credentials
- [ ] DNS domain configured or Route53 alias created
- [ ] Database password stored securely in Jenkins credentials
- [ ] Test deployment successful in dev environment
- [ ] Approval workflow configured in Jenkins (for prod)

## Next Steps

1. **Copy Manifests**: Ensure all Kubernetes manifests in `manifests/` directory
2. **Configure Environments**: Update `ansible/group_vars/` with actual values
3. **Setup Jenkins**: Create pipeline job and configure credentials
4. **Test Bootstrap**: Run bootstrap pipeline in dev environment
5. **Test Deployment**: Run application deployment in dev environment
6. **Promote to Stage**: Test complete flow in staging
7. **Deploy to Production**: Final production deployment with approvals

## Troubleshooting

### Ansible Connection Issues
```bash
# Test inventory
ansible-inventory -i ansible/inventory.ini --list

# Test connectivity
ansible all -i ansible/inventory.ini -m ping
```

### Kubernetes Permission Issues
```bash
# Verify context
kubectl config current-context

# Check permissions
kubectl auth can-i create deployments --as system:serviceaccount:default:default
```

### ECR Push Failures
```bash
# Verify ECR login
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com

# Check repository access
aws ecr describe-repositories
```

## File Reference

### Key Files Implemented

**Playbooks**:
- `ansible/bootstrap-deploy.yml` - Cluster setup
- `ansible/app-deploy.yml` - Application deployment
- `ansible/rollback-deploy.yml` - Rollback mechanism

**Roles** (all in `ansible/roles/`):
- `prerequisites/tasks/main.yml` - Pre-flight checks
- `namespace_and_secrets/tasks/main.yml` - Namespace setup
- `database/tasks/main.yml` - Database deployment
- `workload_deployment/tasks/main.yml` - Microservices
- `gateway_and_routing/tasks/main.yml` - Ingress setup
- `monitoring/tasks/main.yml` - Logging/metrics
- `validation/tasks/main.yml` - Health checks

**Configuration**:
- `ansible/inventory.ini` - Inventory definition
- `ansible/ansible.cfg` - Ansible settings
- `ansible/group_vars/dev.yml` - Development config
- `ansible/group_vars/stage.yml` - Staging config
- `ansible/group_vars/prod.yml` - Production config

**Jenkins**:
- `jenkins/Jenkinsfile` - Pipeline definition
- `jenkins/README.md` - Setup guide

**Documentation**:
- `ansible/README.md` - Comprehensive Ansible guide
- `IMPLEMENTATION_PLAN.md` - This document

## Support Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [CloudNativePG Documentation](https://cloudnative-pg.io/)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [Jenkins Documentation](https://www.jenkins.io/doc/)

---

**Implementation Date**: March 2026
**Status**: Complete and Ready for Testing
**Next Phase**: Integration Testing with Live EKS Cluster
