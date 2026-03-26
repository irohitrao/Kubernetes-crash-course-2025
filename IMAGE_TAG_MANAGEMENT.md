# Image Tag Management Strategy

## Overview

This document explains how image tags are managed across development, staging, and production environments using the Jenkins-Ansible deployment pipeline.

## Problem & Solution

### Problem
Previously, Kubernetes manifests contained **hardcoded image tags** (e.g., `v1`), which meant:
- No traceability to actual deployment versions
- Manual manifest updates needed for each deployment
- Risk of deploying wrong versions to wrong environments
- Difficult to rollback or pin specific versions

### Solution
Manifests now use **dynamic placeholders** that are rendered at deployment time:

```yaml
# In Git (static, reusable)
image: IMAGE_REGISTRY/auth-service:IMAGE_TAG

# At deployment time (rendered by Ansible)
image: 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:abc1234d
```

## How It Works

### 1. Manifest Structure (In Git)

All deployment manifests use placeholders:

```yaml
# manifests/auth-deploy.yaml
spec:
  containers:
  - name: auth-service
    image: IMAGE_REGISTRY/auth-service:IMAGE_TAG     # ← Placeholder
    imagePullPolicy: Always
```

**What gets checked into Git:**
- `IMAGE_REGISTRY` - ECR registry URL placeholder
- `IMAGE_TAG` - Version tag placeholder
- **Never** hardcoded registry or version

**Why?**
- Single source of truth for all environments
- No drift between dev/stage/prod manifests
- Safe to commit to Git (no secrets)

### 2. Tag Generation (In Jenkins)

Jenkins generates the appropriate tag based on the branch:

```groovy
// Jenkinsfile
if (!SKIP_BUILD && !IMAGE_TAG) {
    IMAGE_TAG = sh(
        script: 'git rev-parse --short HEAD',
        returnStdout: true
    ).trim()
    echo "Generated image tag: ${IMAGE_TAG}"
}
```

| Branch | Generated Tag | Type | Example |
|--------|---------------|------|---------|
| `feature/*` | Commit SHA | Commit hash | `abc1234d` |
| `develop` | Commit SHA | Commit hash | `def5678e` |
| `release/1.2.0` | Release version | Semantic | `1.2.0-rc1` |
| `main` + tag v1.2.0 | Git tag | Version | `v1.2.0` |

### 3. Image Build & Push (In Jenkins)

Images are built and pushed with the generated tag:

```bash
# In Jenkinsfile: 'Build Images' stage
cd ${WORKSPACE}/auth-service
docker build -t ${ECR_REGISTRY}/auth-service:${IMAGE_TAG} .
docker push ${ECR_REGISTRY}/auth-service:${IMAGE_TAG}
```

**Result in ECR:**
```
ECR Repository: auth-service
  - abc1234d (latest dev)
  - def5678e (previous dev)
  - 1.2.0-rc1 (stage candidate)
  - v1.2.0 (production release)
  - v1.2.1 (production release)
```

### 4. Template Rendering (In Ansible)

Ansible renders manifests at deployment time using sed:

```yaml
# ansible/roles/workload_deployment/tasks/main.yml
- name: Render and apply auth deployment
  shell: |
    cat /tmp/manifests/auth-deploy.yaml | \
    sed "s|IMAGE_REGISTRY|{{ ecr_repository_url }}|g" | \
    sed "s|IMAGE_TAG|{{ auth_image_tag }}|g" | \
    kubectl apply -f - --namespace={{ k8s_namespace }} --context={{ k8s_context }}
```

**Transformation:**
```yaml
# Input (from Git)
image: IMAGE_REGISTRY/auth-service:IMAGE_TAG

# After sed replacement
image: 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:abc1234d

# Applied to cluster
kubectl apply -f rendered-manifest.yaml
```

### 5. Deployment Verification

After applying, verification ensures correct images are running:

```bash
# In Jenkinsfile: 'Verify Deployment' stage
for deployment in auth game frontend; do
    status=$(kubectl get deployment $deployment -n $namespace \
        -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')
    if [ "$status" = "True" ]; then
        echo "✓ $deployment deployment is ready"
    fi
done

# Verify image is running
kubectl get deployment auth -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:abc1234d
```

## Environment-Specific Flows

### Development Environment

```
1. Developer pushes commit abc1234d to feature/new-auth
   ↓
2. Jenkins detects commit
   ↓
3. Generates IMAGE_TAG = abc1234d
   ↓
4. Builds: auth-service:abc1234d
   ↓
5. Pushes to ECR: auth-service:abc1234d
   ↓
6. Ansible renders manifests with IMAGE_TAG=abc1234d
   ↓
7. Deploy to dev-eks cluster in 'crash-course' namespace
   ↓
8. Verification: ✓ auth deployment ready with image abc1234d
```

**Result in Cluster:**
```yaml
image: 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:abc1234d
```

### Staging Environment

```
1. Release manager creates release/1.2.0 branch
   ↓
2. Jenkins detects release branch
   ↓
3. Generates IMAGE_TAG = 1.2.0-rc1 (version from release branch)
   ↓
4. Builds: auth-service:1.2.0-rc1
   ↓
5. Pushes to ECR: auth-service:1.2.0-rc1
   ↓
6. Ansible renders manifests with IMAGE_TAG=1.2.0-rc1
   ↓
7. Deploy to stage-eks cluster in 'crash-course' namespace
   ↓
8. Verification: ✓ auth deployment ready with image 1.2.0-rc1
   ↓
9. Testing team validates stage deployment
```

**Result in Cluster:**
```yaml
image: 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:1.2.0-rc1
```

### Production Environment

```
1. After stage approval, merge release/1.2.0 → main
   ↓
2. Create git tag v1.2.0
   ↓
3. Push tag to main branch
   ↓
4. Jenkins detects tag
   ↓
5. Generates IMAGE_TAG = v1.2.0
   ↓
6. Builds: auth-service:v1.2.0
   ↓
7. Pushes to ECR: auth-service:v1.2.0
   ↓
8. Waits for manual approval in Jenkins
   ↓
9. Ansible renders manifests with IMAGE_TAG=v1.2.0
   ↓
10. Deploy to prod-eks cluster in 'crash-course' namespace
    ↓
11. Verification: ✓ auth deployment ready with image v1.2.0
    ↓
12. Prod is running v1.2.0
```

**Result in Cluster:**
```yaml
image: 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:v1.2.0
```

## Manifest Structure

All deployment manifests follow this pattern:

```yaml
spec:
  containers:
  - name: service-name
    image: IMAGE_REGISTRY/service-name:IMAGE_TAG        # Placeholder
    imagePullPolicy: Always                              # Always pull
    ports:
    - containerPort: 8080
```

**Key Points:**
- `imagePullPolicy: Always` ensures fresh image pull on every deployment
- `IMAGE_REGISTRY` and `IMAGE_TAG` are placeholders
- Same manifest deploys to all environments
- No hardcoded registry URLs or version tags

## Tracking & Traceability

### View Deployed Images

```bash
# Check what image is actually running on prod
kubectl get deployment auth -n crash-course -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:v1.2.0

# See all images in a deployment
kubectl get deployment -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].image}{"\n"}{end}'
```

### Git History

```bash
# Find when v1.2.0 was deployed
git log --oneline --grep="v1.2.0"
# Shows commit that created the tag

# See what was in v1.2.0
git show v1.2.0:manifests/auth-deploy.yaml
# Shows manifest at that tag
```

### Jenkins Build History

Each Jenkins build archives:
- Build number and timestamp
- Environment deployed to
- Image tag used
- Git commit SHA and branch
- Deployment status

```
Build #42
├── Timestamp: 2026-03-26 14:30:00
├── Environment: prod
├── Image Tag: v1.2.0
├── Git Commit: abc1234def5678
├── Git Branch: main
└── Status: SUCCESS
```

## Rollback Strategy

If production deployment fails:

```bash
# Find last known good image tag from Jenkins history
# Previous prod deployment used: v1.1.9

# Redeploy with previous tag
ENVIRONMENT=prod IMAGE_TAG=v1.1.9 ansible-playbook app-deploy.yml

# Or trigger Jenkins manually:
# Environment: prod
# Image Tag: v1.1.9
# Click "Build Now"

# Result: Cluster reverts to v1.1.9
```

## Key Benefits

✅ **Traceability** - Every image tag can be traced to exact Git commit/tag  
✅ **Repeatability** - Same manifests work for all environments  
✅ **Auditability** - Full history in Git, Jenkins, and ECR  
✅ **Safety** - Manifests in Git never change, only tags change  
✅ **Rollback** - Quick rollback by specifying previous tag  
✅ **Immutability** - Each tag is immutable once pushed to ECR  

## Secrets Management

### What Goes in Manifests
✅ Environment variables (non-sensitive)  
✅ Resource requests/limits  
✅ Port configurations  
✅ Image pull policy  

### What Stays Out
❌ Database passwords  
❌ API keys  
❌ AWS credentials  
❌ TLS certificates  

**Where secrets go:**
- **Kubernetes Secrets** (referenced in manifests via `valueFrom.secretKeyRef`)
- **Ansible Vault** (encrypted files for pipeline credentials)
- **Jenkins Credentials** (AWS keys, kubeconfig)

Example in manifest:
```yaml
env:
- name: POSTGRES_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-app              # Secret created by Ansible
      key: password
```

## Workflow Summary

```
┌─ Developer commits code
│           ↓
├─ Jenkins auto-detects (webhook)
│           ↓
├─ Generate IMAGE_TAG from commit SHA/branch/tag
│           ↓
├─ Build Docker image with tag
│           ↓
├─ Push to ECR with tag
│           ↓
├─ Ansible loads manifests from Git
│           ↓
├─ Render manifests (sed replaces placeholders)
│           ↓
├─ kubectl apply rendered manifest
│           ↓
├─ Pods pull image from ECR (imagePullPolicy: Always)
│           ↓
└─ Deployment ready with traced version
```

## Best Practices

1. **Commit SHA for dev** - Changes with every commit, shows rapid iteration
2. **RC tags for stage** - Stable for testing period
3. **Version tags for prod** - Semantic versioning for releases
4. **Always use placeholders** - Never hardcode registry/tag in Git
5. **imagePullPolicy: Always** - Ensures fresh pull on every deployment
6. **Git tags for prod** - Create git tag before production deployment
7. **Archive build metadata** - Save IMAGE_TAG in Jenkins artifacts for traceability

## Troubleshooting

### Deployment stuck with wrong image
```bash
# Check what image is running
kubectl get deployment auth -o jsonpath='{.spec.template.spec.containers[0].image}'

# If wrong, check if image exists in ECR
aws ecr describe-images --repository-name auth-service

# Redeploy with correct tag
# Use force_rollout=true to force pod restart
```

### Image not found in ECR
```bash
# Verify push succeeded
aws ecr describe-images --repository-name auth-service --query 'imageDetails[].[imageTags]'

# Check Jenkins build log for push errors
# Re-run Jenkins build or manually push:
docker push 408009925873.dkr.ecr.ap-south-1.amazonaws.com/auth-service:v1.2.0
```

### Manifest placeholder not replaced
```bash
# Check Ansible logs for sed errors
# Make sure IMAGE_TAG and IMAGE_REGISTRY are passed correctly
ansible-playbook -e "IMAGE_TAG=abc1234" -e "IMAGE_REGISTRY=ECR_URL" ...

# Manually test sed replacement
sed "s|IMAGE_REGISTRY|408009925873.dkr.ecr.ap-south-1.amazonaws.com|g" manifests/auth-deploy.yaml
```
