# Ansible Deployment Pipeline Documentation

## Overview

This Ansible-based deployment pipeline coordinates:
- **Bootstrap Pipeline**: One-time cluster and database setup
- **Application Release Pipeline**: Repeatable application deployments
- **Rollback Pipeline**: Safe rollback to previous versions

## Directory Structure

```
ansible/
├── inventory.ini                 # Ansible inventory (dev, stage, prod)
├── ansible.cfg                   # Ansible configuration
├── bootstrap-deploy.yml          # Bootstrap playbook (cluster prerequisites)
├── app-deploy.yml               # Application release playbook
├── rollback-deploy.yml          # Rollback playbook
├── group_vars/
│   ├── dev.yml                  # Development environment variables
│   ├── stage.yml                # Stage environment variables
│   └── prod.yml                 # Production environment variables
└── roles/
    ├── prerequisites/           # Pre-flight checks and validation
    ├── namespace_and_secrets/   # Namespace and secret management
    ├── database/                # PostgreSQL cluster deployment
    ├── workload_deployment/     # Auth, game, frontend deployments
    ├── gateway_and_routing/     # Gateway, HTTPRoute, and TLS setup
    ├── monitoring/              # ServiceMonitor and metrics setup
    └── validation/              # Final verification and health checks
```

## Environment Configuration

Each environment (dev, stage, prod) has its own configuration file in `group_vars/`:

- `AWS_ACCOUNT_ID`: AWS account ID for ECR access
- `K8S_NAMESPACE`: Kubernetes namespace for deployment (**Common across all environments**: `app`)
- `K8S_CONTEXT`: kubectl context for different clusters (e.g., `dev-eks`, `stage-eks`, `prod-eks`)
- `APP_DOMAIN`: Application domain name (environment-specific)
- `CERT_EMAIL`: Email for Let's Encrypt certificates (environment-specific)
- `IMAGE_TAGS`: Container image tags to deploy
- `FEATURE_FLAGS`: Enable/disable monitoring, TLS, etc.

**Architecture Note**: This setup uses **separate EKS clusters per environment** for isolation. Since cluster separation provides environment isolation, the application namespace is **shared** (`app`) across all environments. The kubeconfig context (`k8s_context`) points to the correct cluster.

## Prerequisite Setup

Before running any pipeline, ensure:

1. **EKS Cluster Access**:
   ```bash
   aws eks update-kubeconfig --name <cluster-name> --region <region>
   kubectl config current-context
   ```

2. **Cluster Prerequisites**:
   - cert-manager installed
   - CloudNativePG operator installed
   - Gateway API or kgateway installed
   - Prometheus operator (optional for monitoring)
   - AWS EBS CSI driver installed

3. **AWS ECR Repositories**:
   ```bash
   aws ecr create-repository --repository-name frontend --region us-east-1
   aws ecr create-repository --repository-name auth --region us-east-1
   aws ecr create-repository --repository-name game --region us-east-1
   ```

4. **Ansible Installation**:
   ```bash
   pip install ansible
   ```

## Bootstrap Pipeline

Run once during initial cluster setup:

```bash
# Development environment
ansible-playbook -i ansible/inventory.ini ansible/bootstrap-deploy.yml \
  -e "env_name=dev" \
  -e "frontend_image_tag=<SHA>" \
  -e "auth_image_tag=<SHA>" \
  -e "game_image_tag=<SHA>"

# Production environment (with manual approval)
ansible-playbook -i ansible/inventory.ini ansible/bootstrap-deploy.yml \
  -e "env_name=prod" \
  -e "frontend_image_tag=<SHA>" \
  -e "auth_image_tag=<SHA>" \
  -e "game_image_tag=<SHA>" \
  --ask-vault-pass
```

**What it does**:
1. Validates kubectl and AWS access
2. Verifies required CRDs are installed (cert-manager, CloudNativePG, Gateway API)
3. Creates namespace and application secret
4. Deploys PostgreSQL cluster
5. Waits for database readiness

**Manual steps after bootstrap**:
```bash
# Verify database initialization
kubectl exec -it -n default $(kubectl get pod -l app.kubernetes.io/name=postgresql -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U postgres -d app -c "SELECT 1"
```

## Application Release Pipeline

Run for each application deployment:

```bash
# Development environment
ansible-playbook -i ansible/inventory.ini ansible/app-deploy.yml \
  -e "env_name=dev" \
  -e "frontend_image_tag=abc123" \
  -e "auth_image_tag=abc123" \
  -e "game_image_tag=abc123"

# Production environment
ansible-playbook -i ansible/inventory.ini ansible/app-deploy.yml \
  -e "env_name=prod" \
  -e "frontend_image_tag=abc123" \
  -e "auth_image_tag=abc123" \
  -e "game_image_tag=abc123" \
  --ask-vault-pass
```

**What it does**:
1. Validates prerequisites
2. Creates/updates namespace and secrets
3. Deploys auth, game, frontend workloads with image tags
4. Applies Gateway and HTTPRoute for ingress
5. Waits for gateway to be programmed
6. Verifies certificate issuance
7. Validates all deployments are ready
8. Performs post-deployment checks

## Rollback Pipeline

In case of deployment issues, rollback to previous versions:

```bash
ansible-playbook -i ansible/inventory.ini ansible/rollback-deploy.yml \
  -e "env_name=prod" \
  -e "previous_frontend_image_tag=<old_SHA>" \
  -e "previous_auth_image_tag=<old_SHA>" \
  -e "previous_game_image_tag=<old_SHA>"
```

**What it does**:
1. Redeployss workloads with previous image versions
2. Waits for rollback to complete
3. Validates all deployments are ready

## Jenkins Integration

The `Jenkinsfile` orchestrates the complete pipeline:

```groovy
Stage 1: Validate Parameters
  - Confirm environment, image tags, and approvals

Stage 2: Checkout Source
  - Clone repository with application source

Stage 3: Build Images
  - Build Docker images for frontend, auth, game services

Stage 4: Push Images to ECR
  - Authenticate to AWS ECR and push images

Stage 5: Run Bootstrap Pipeline (optional)
  - Execute cluster prerequisites

Stage 6: Run Application Deployment
  - Execute Ansible deployment playbook

Stage 7: Verify Deployment
  - Check deployment readiness and gateway status

Stage 8: Archive Artifacts
  - Save rendered manifests and metadata
```

### Jenkins Pipeline Usage

**Development Deployment**:
```
Environment: dev
Image Tag: <leave empty for auto-generate from commit SHA>
Skip Build: false
Skip Database: true
Run Bootstrap: false
```

**Production Deployment** (with approval):
```
Environment: prod
Image Tag: v1.2.3
Skip Build: false
Skip Database: true
Run Bootstrap: false
(Requires manual approval in Jenkins)
```

**Bootstrap Production Cluster** (rare):
```
Environment: prod
Image Tag: v1.2.3
Skip Build: false
Skip Database: false
Run Bootstrap: true
(Requires manual approval in Jenkins)
```

## Role Descriptions

### prerequisites
- Validates kubectl and AWS access
- Checks for required CRDs (cert-manager, CloudNativePG, Gateway API)
- Creates namespace if needed

### namespace_and_secrets
- Ensures namespace exists
- Creates PostgreSQL credentials secret

### database
- Deploys ConfigMap for database initialization
- Deploys PostgreSQL Cluster resource
- Waits for cluster readiness
- Verifies database connectivity

### workload_deployment
- Templating and applies auth, game, frontend deployments
- Applies corresponding services
- Waits for deployment readiness
- Verifies service endpoints

### gateway_and_routing
- Applies ClusterIssuer for TLS
- Deploys Gateway resource
- Deploys HTTPRoute rules
- Applies HTTP redirect if enabled
- Waits for gateway to be programmed
- Verifies certificate issuance

### monitoring
- Checks for Prometheus operator
- Deploys ServiceMonitor if monitoring is enabled
- Verifies ServiceMonitor acceptance

### validation
- Final checks for all deployments, services, gateway, routes, and TLS
- Prints comprehensive deployment summary

## Monitoring and Troubleshooting

### Check Deployment Status
```bash
# List all deployments
kubectl get deployments -n <namespace>

# Check deployment details
kubectl describe deployment <name> -n <namespace>

# View deployment events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### View Logs
```bash
# Current deployment logs
kubectl logs -f deployment/auth -n <namespace>
kubectl logs -f deployment/game -n <namespace>
kubectl logs -f deployment/frontend -n <namespace>

# Previous deployment logs (if crashed)
kubectl logs -f deployment/auth -n <namespace> --previous
```

### Check Gateway Status
```bash
# Gateway status
kubectl get gateway -n <namespace> -o wide

# HTTPRoute status
kubectl get httproute -n <namespace> -o wide

# Gateway events
kubectl describe gateway <name> -n <namespace>
```

### Verify TLS Certificate
```bash
# List certificates
kubectl get certificate -n <namespace>

# Check certificate details
kubectl describe certificate <name> -n <namespace>

# View secret
kubectl get secret <tls-secret> -n <namespace> -o yaml
```

### Database Health
```bash
# Get PostgreSQL cluster status
kubectl get postgresqlclusters -n <namespace>

# Check database pod
kubectl get pods -l app.kubernetes.io/name=postgresql -n <namespace>

# Database logs
kubectl logs -f -l app.kubernetes.io/name=postgresql -n <namespace>
```

## Variable Reference

### Required Variables (passed from Jenkins/CLI)
- `env_name`: Environment name (dev, stage, prod)
- `frontend_image_tag`: Frontend image tag/SHA
- `auth_image_tag`: Auth service image tag/SHA
- `game_image_tag`: Game service image tag/SHA

### Environment-Specific Variables (from group_vars)
- `k8s_namespace`: Kubernetes namespace
- `k8s_context`: kubectl context name
- `app_domain`: Application domain
- `cert_email`: Certificate email
- `db_username`: Database username
- `db_password`: Database password (secret)
- `enable_monitoring`: Enable Prometheus monitoring
- `enable_tls`: Enable TLS certificates
- `enable_http_redirect`: Enable HTTP to HTTPS redirect
- `workload_ready_timeout`: Deployment readiness timeout
- `database_ready_timeout`: Database readiness timeout
- `gateway_ready_timeout`: Gateway readiness timeout

## Best Practices

1. **Always use immutable image tags**: Use commit SHAs rather than 'latest'
2. **Test in dev first**: Always deploy to dev/stage before production
3. **Review manifests**: Check rendered manifests before applying to prod
4. **Monitor rollout**: Watch deployment status in real-time during changes
5. **Keep rollback ready**: Document previous image tags for quick rollback
6. **Backup database**: Ensure PostgreSQL backups before major updates
7. **Use GitOps**: Store all configuration in git for auditability
8. **Stagger deployments**: Space out production deployments to catch issues early

## Vault Integration (Optional)

For sensitive data like database passwords:

```bash
# Create vault-encrypted group_vars
ansible-vault create group_vars/prod_secrets.yml

# Run playbook with vault prompt
ansible-playbook ... --ask-vault-pass

# Or use vault password file
ansible-playbook ... --vault-password-file=.vault_pass
```

## Advanced Usage

### Deployment with Skip Steps
```bash
# Deploy only application workloads, skip database and gateway
ansible-playbook ... --skip-tags="database,gateway"

# Deploy only validation
ansible-playbook ... --tags="validation"
```

### Dry-run Mode (Check-only)
```bash
# Preview changes without applying
ansible-playbook ... --check --diff
```

### Verbose Output
```bash
# Detailed debugging
ansible-playbook ... -vvv
```

## Handling Rollouts When Spec Hasn't Changed

### Problem Scenario

`kubectl apply` doesn't trigger a rollout when the manifest spec is unchanged. This occurs when:
- **Image tag is the same** but image content changed (rebuilt from same commit)
- **Manifests are identical** but you want to force a fresh deployment
- **Manual redeploy** needed without modifying any YAML

### Solution 1: Force Rollout Option (Recommended)

Use the `force_rollout` parameter to restart all pods:

**Via Jenkins**:
```
FORCE_ROLLOUT: true
```

**Via Ansible CLI**:
```bash
ansible-playbook -i ansible/inventory.ini ansible/app-deploy.yml \
  -e "env_name=prod" \
  -e "frontend_image_tag=abc123" \
  -e "auth_image_tag=abc123" \
  -e "game_image_tag=abc123" \
  -e "force_rollout=true"
```

This uses `kubectl rollout restart` to restart all pods without reapplying manifests.

### Solution 2: Image Pull Policy

Ensure deployments use `imagePullPolicy: Always`:

```yaml
spec:
  template:
    spec:
      containers:
      - name: auth
        image: {{ auth_image }}:{{ auth_image_tag }}
        imagePullPolicy: Always  # Forces fresh image pull every deployment
```

This guarantees Kubernetes pulls the latest image even if tag is unchanged.

### Solution 3: Pod Annotation Trick

Patch deployment with timestamp annotation to force spec change:

```bash
kubectl patch deployment auth -n {{ k8s_namespace }} \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"restartedAt\":\"$(date +%s)\"}}}}}"
```

The deployment spec changes (annotation updated), triggering pod recreation.

### Solution 4: Strategy Pattern

Temporarily change deployment strategy to force redeployment:

```bash
# Change to Recreate for one cycle
kubectl patch deployment auth -n {{ k8s_namespace }} \
  -p '{"spec":{"strategy":{"type":"Recreate"}}}'

# Then apply
kubectl apply -f auth-deploy.yaml -n {{ k8s_namespace }}

# Revert strategy
kubectl patch deployment auth -n {{ k8s_namespace }} \
  -p '{"spec":{"strategy":{"type":"RollingUpdate"}}}'
```

### Comparison of Approaches

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **Force Rollout** | Simple, clean, no manifest changes | Requires new parameter | Standard redeployments |
| **imagePullPolicy: Always** | Per-deployment setting | Always pulls images (slower) | Development environments |
| **Pod Annotation** | Works with `kubectl apply` | Modifies spec (audit trail) | Immutable deployments |
| **Strategy Patch** | Controlled rolling restart | Complex, multiple steps | Advanced scenarios |

### Real-World Example

**Scenario**: You rebuilt the image from the same commit (same tag), but want to deploy the new image.

**Process**:

1. Jenkins detects image content changed (via image digest)
2. Jenkins triggers deployment with `FORCE_ROLLOUT=true`
3. Ansible playbook:
   - Applies manifests (no change in spec)
   - Sees `force_rollout: true`
   - Runs `kubectl rollout restart` for each deployment
   - Pods are terminated and recreated
   - New pods pull latest image (imagePullPolicy: Always)
   - Deployment validation confirms readiness

**Result**: Fresh deployment without manifest changes.

## Support and Troubleshooting

For more information, see:
- Ansible Documentation: https://docs.ansible.com/
- Kubernetes: https://kubernetes.io/docs/
- CloudNativePG: https://cloudnative-pg.io/
- Gateway API: https://gateway-api.sigs.k8s.io/
