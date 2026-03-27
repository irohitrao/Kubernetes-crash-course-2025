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

### 3. Setup Git Webhook for Automatic Triggers

The pipeline automatically builds and deploys on code changes via Git webhook. No manual trigger needed for dev/stage!

**Configure webhook in GitHub:**
1. Go to Repository Settings → Webhooks → Add webhook
2. Payload URL: `http://<jenkins-url>/generic-webhook-trigger/invoke?token=kubernetes-deploy-webhook`
3. Content type: `application/json`
4. Events: Push events
5. Click "Add webhook"

**Auto-deployment flow:**
```
feature/my-feature pushed
        ↓
Webhook triggers pipeline (automatic)
        ↓
ENVIRONMENT auto-set to dev
IMAGE_TAG generated from commit SHA (abc1234d)
        ↓
Build Docker images using code from feature/my-feature
        ↓
Push images to ECR with tag abc1234d
        ↓
Deploy to dev-eks cluster
        ↓
Services restart with new images
```

### 4. Run Pipeline

The pipeline automatically generates image tags based on Git branches. Manually trigger only if needed:

**Development Build** (automatic on commit to feature/* or develop):
```
Trigger: Git webhook (automatic)
Environment: AUTO-DETECTED as dev
Image Tag: AUTO-GENERATED from commit SHA
Result: Images built from feature branch → deployed to dev-eks
Example: Commit abc1234d on feature/auth-fix → builds & deploys instantly
```

**Staging Release** (automatic on merge to release/x.y.z):
```
Trigger: Git webhook (automatic)
Environment: AUTO-DETECTED as stage
Image Tag: AUTO-GENERATED from release branch (e.g., 1.2.0-rc1)
Result: Images built from release branch → deployed to stage-eks
Example: Commit on release/1.2.0 → builds v1.2.0-rc1 → deploys to stage
```

**Production Deployment** (manual only, requires git tag):
```
Trigger: MANUAL ONLY (no webhook) - requires deliberate action for safety
Environment: MANUAL selection (prod)
Image Tag: AUTO-GENERATED from git tag (requires semantic version tag)
Result: Images built from tagged commit → deployed to prod-eks
Example: git tag v1.2.0 on main → manual trigger → deploys v1.2.0 to prod
```

### Manual Trigger Combinations

When you manually trigger the pipeline, you control IMAGE_TAG and SKIP_BUILD. Here are the three options:

**Option 1: Auto-Generate (Recommended for simplicity)**
```
ENVIRONMENT: stage (or prod)
IMAGE_TAG: (leave EMPTY)
SKIP_BUILD: false (default)

Result:
├─ Jenkinsfile auto-detects branch
├─ Generates appropriate IMAGE_TAG:
│  ├─ release/1.2.0 → 1.2.0-rc43
│  ├─ main (with tag) → v1.2.0
│  └─ main (no tag) → ERROR (production requires tag)
├─ Builds fresh images
└─ Deploys
```

**Option 2: Rebuild with Custom Tag (Rare - rebuilds code)**
```
ENVIRONMENT: prod
IMAGE_TAG: my-custom-tag
SKIP_BUILD: false (default)

⚠️ WARNING: This rebuilds images with your custom tag
- Will DELETE old images with same tag
- Only use if you know what you're doing
- Jenkins will rebuild from current branch code
```

**Option 3: Reuse Existing Tested Images (Best for Promotion/Rollback)**
```
ENVIRONMENT: prod
IMAGE_TAG: 1.2.0-rc42 (MUST exist in ECR - the exact tag you tested!)
SKIP_BUILD: true (CRITICAL)

⚠️ IMPORTANT: Tag must EXACTLY match what's in ECR
✅ Examples that work:
 • IMAGE_TAG=1.2.0-rc42 (RC tested on stage)
 • IMAGE_TAG=v1.2.0 (semantic version built on main)
 • IMAGE_TAG=abc1234d (old development build)

❌ Examples that DON'T work:
 • IMAGE_TAG=1.2.0 (doesn't exist unless rebuilt)
 • IMAGE_TAG=latest (ECR images are tagged with SHA/version, not latest)

Result:
├─ No checkout, no build (fast!)
├─ Pulls IMAGE_TAG from ECR
├─ Deploys directly
└─ Usually paired with FORCE_ROLLOUT=true
```

**Common Mistake & How Jenkins Now Helps:**
```
You manually trigger with:
ENVIRONMENT = stage
IMAGE_TAG = 1.2.0    ← Incomplete! Should be 1.2.0-rc42
SKIP_BUILD = true

Jenkins now outputs:
❌ WARNING: Tag '1.2.0' looks incomplete!
   Did you mean one of these?
   - Rollback to RC: 1.2.0-rc42
   - Deploy tagged release: v1.2.0
```

**Manual Override** (when needed):
```
Trigger: Manual with parameters
Environment: Manually select dev/stage/prod
Image Tag: Manually specify exact ECR tag
Result: Uses specified images (with correct tag!)
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

## How Images Are Built and Deployed

### Image Build Process

**Step 1: Git Webhook Triggers Pipeline**
```
You push code to feature/auth-fix branch
        ↓
GitHub webhook calls Jenkins webhook endpoint
        ↓
Jenkins receives push notification
        ↓
Pipeline starts automatically
```

**Step 2: Checkout and Build Docker Images** (Stage 3)
```
def currentBranch = sh('git rev-parse --abbrev-ref HEAD').trim()  // Gets 'feature/auth-fix'
git clone <your-repo>
cd auth-service && docker build -t ECR/auth:abc1234d .            // Builds from YOUR changed code
cd frontend && docker build -t ECR/frontend:abc1234d .
cd game-service && docker build -t ECR/game:abc1234d .
```

**Step 3: Push to ECR** (Stage 5)
```
docker push ECR/auth:abc1234d                                      // Pushes to Amazon ECR
docker push ECR/frontend:abc1234d
docker push ECR/game:abc1234d
```

**Step 4: Deploy to dev-eks** (Stage 7)
```
ansible-playbook app-deploy.yml \
  -e "env_name=dev" \
  -e "frontend_image_tag=abc1234d" \                               // Uses YOUR images
  -e "auth_image_tag=abc1234d" \
  -e "game_image_tag=abc1234d"
        ↓
Kubernetes pulls images from ECR with tag abc1234d (YOUR code)
        ↓
Pods restart → Services available at dev domain
```

### Environment Auto-Detection

The pipeline automatically detects which environment to deploy to based on the Git branch:

```groovy
def currentBranch = sh('git rev-parse --abbrev-ref HEAD').trim()

if (currentBranch == 'main') {
    ENVIRONMENT = 'prod'     // Main branch → production
} 
else if (currentBranch =~ /^release\//) {
    ENVIRONMENT = 'stage'    // release/1.2.0 → staging
}
else {
    ENVIRONMENT = 'dev'      // feature/* or develop → dev
}
```

**Result**: No manual environment selection needed for dev. Push code → images auto-built and deployed to dev-eks!

## Input Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ENVIRONMENT` | choice | auto | Target environment (auto-detected from branch, or manual override) |
| `IMAGE_TAG` | string | auto | Docker image tag - auto-generated based on branch (see below) or manually specified |
| `SKIP_BUILD` | boolean | false | Skip Docker build, use existing images |
| `SKIP_DATABASE` | boolean | true | Skip database deployment |
| `RUN_BOOTSTRAP` | boolean | false | Run cluster bootstrap pipeline |
| `FORCE_ROLLOUT` | boolean | false | Force pod restart even if spec unchanged (kubectl rollout restart) |

### Branch-Based Image Tag Strategy

When `IMAGE_TAG` is not manually specified, the pipeline **automatically generates** the tag based on the Git branch:

| Branch | Generated Tag | Type | Used For | Example |
|--------|---------------|------|----------|---------|
| `feature/*` | Commit SHA | Short hash | Development | `abc1234d` |
| `develop` | Commit SHA | Short hash | Development | `def5678e` |
| `release/x.y.z` | Version-RC | Release candidate | Staging | `1.2.0-rc1` |
| `main` (with git tag) | Git tag | Semantic version | Production | `v1.2.0` |
| `main` (no tag) | **❌ FAILS** | **Required** | **Production only** | **Build fails** |

⚠️ **PRODUCTION REQUIREMENT**: Deployments to production (`main` branch) **MUST** have a git semantic version tag (e.g., `v1.2.0`). Without a tag, the pipeline will fail with an error message directing you to:
```bash
git tag v1.2.0
git push origin v1.2.0
```

**Benefits:**
- ✅ **Dev/Feature**: Changes with every commit, enables fast iteration
- ✅ **Release**: Stable during RC testing phase
- ✅ **Production**: Semantic versioning for releases, full traceability

### Examples

**Development (auto-generates from commit SHA):**
```groovy
// Branch: feature/new-auth
// Commit: abc1234d7f9e2c1b5a8ced3f6
// Generated: IMAGE_TAG = abc1234d
// Result: auth-service:abc1234d
```

**Staging (auto-generates from release branch):**
```groovy
// Branch: release/1.2.0
// Generated: IMAGE_TAG = 1.2.0-rc1 (or rc2, rc3 from build number)
// Result: auth-service:1.2.0-rc1
```

**Production (auto-generates from git tag):**
```groovy
// Branch: main
// Git Tag: v1.2.0
// Generated: IMAGE_TAG = v1.2.0
// Result: auth-service:v1.2.0
```

**Production without Git Tag (FAILS):**
```groovy
// Branch: main (no git tag)
// ERROR: ❌ PRODUCTION DEPLOYMENT REQUIRES A GIT TAG
// Current commit has no git tag.
// Solution: Create semantic version tag with:
//   git tag v1.2.0
//   git push origin v1.2.0
// Then trigger pipeline again
```

**Manual Override (always works):**
```groovy
// Manually specify IMAGE_TAG parameter
// IMAGE_TAG = custom-tag-123
// Pipeline uses: auth-service:custom-tag-123
```

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

## Best Practices for Image Tagging

### 1. Use Branch-Based Auto-Generation
✅ **DO**: Let the pipeline auto-generate tags based on branch
- Feature branches: Always use auto-generated commit SHA
- Release branches: Always use auto-generated RC tags
- Main with git tag: Always use auto-generated semantic version

❌ **DON'T**: Manually override tags for dev/stage (breaks traceability)

```bash
# Good - relies on auto-generation
Environment: dev, IMAGE_TAG: (empty)

# Avoid - loses traceability
Environment: dev, IMAGE_TAG: v1.2.0
```

### 2. Git Tagging for Production

Create semantic version git tags for production releases:

```bash
# After final testing in stage, tag in git
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin v1.2.0

# Then manually deploy to prod with auto-generated tag
Environment: prod, IMAGE_TAG: (empty)
# Pipeline will detect v1.2.0 tag and use it
```

### 3. Release Candidate Testing

Use release branches for staging validation:

```bash
# Create release branch
git checkout -b release/1.2.0 develop

# Update version files, CHANGELOG
# Push to release branch

# Jenkins auto-generates: 1.2.0-rc1, 1.2.0-rc2, etc.
# Each RC is independent for testing

# After approval, merge to main and create git tag
```

### 4. Rollback Strategy

To rollback to a previous version, manually specify the IMAGE_TAG:

```groovy
// Previous prod was v1.1.9
Environment: prod
IMAGE_TAG: v1.1.9  // Manually specify previous version
FORCE_ROLLOUT: true  // Ensure pods restart
```

### 5. Build Artifact Retention

Jenkins saves build metadata including IMAGE_TAG in artifacts:

```bash
# Find all production deployments
ls -la deployment-artifacts/*/metadata.txt

# Check what was deployed in build #42
cat deployment-artifacts/42/metadata.txt
# Shows: Image Tag: v1.2.0, Environment: prod, timestamp, etc.
```

## Setting Up Webhook Triggers

### Prerequisites

Install the **Generic Webhook Trigger** plugin in Jenkins:
1. Go to Jenkins → Manage Jenkins → Manage Plugins
2. Search for "Generic Webhook Trigger"
3. Install and restart Jenkins

### Configure GitHub Webhook

**Step 1: Get Jenkins Webhook URL**
- Jenkins URL: `http://<jenkins-hostname>:8080`
- Webhook endpoint: `http://<jenkins-hostname>:8080/generic-webhook-trigger/invoke?token=kubernetes-deploy-webhook`

**Step 2: Add webhook in GitHub**
1. Go to your repository → Settings → Webhooks → Add webhook
2. **Payload URL**: `http://<jenkins-hostname>:8080/generic-webhook-trigger/invoke?token=kubernetes-deploy-webhook`
3. **Content type**: `application/json`
4. **Events**: Select "Push events"
5. Click "Add webhook"

**Step 3: Test webhook**
```bash
# Push code to feature branch
git checkout -b feature/test-webhook
echo "test" >> README.md
git add .
git commit -m "test webhook"
git push origin feature/test-webhook

# Jenkins should automatically trigger build
# Check Jenkins → Blue Ocean or Build History
```

### How Webhook Triggers Work

```
┌─────────────────────────┐
│  You push to feature/*  │
│  git push origin ...    │
└────────────┬────────────┘
             │
             ↓
┌─────────────────────────┐
│  GitHub Webhook Fires   │
│  Match: refs/heads/feature/...
└────────────┬────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│  Jenkins Generic Webhook Trigger    │
│  ├─ Checks token validity           │
│  ├─ Extracts branch info            │
│  └─ Starts pipeline                 │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────┐
│ Validate Parameters     │
│ ├─ Branch detected      │
│ ├─ ENVIRONMENT = dev    │
│ ├─ IMAGE_TAG based on DA... "
└────────────┬────────────┘
             │
             ↓
┌─────────────────────────┐
│ Build Docker Images     │
│ ├─ Uses YOUR code       │
│ ├─ Tags: abc123d (SHA)  │
│ └─ Push to ECR          │
└────────────┬────────────┘
             │
             ↓
┌─────────────────────────┐
│ Deploy to dev-eks       │
│ ├─ Render manifests     │
│ ├─ kubectl apply        │
│ └─ Verify pods ready    │
└─────────────────────────┘
```

### Webhook Trigger Patterns

The Jenkinsfile webhook is configured to trigger on:

**Development (Automatic):**
- `feature/*` branches - any feature branch with code changes
- `develop` branch - development mainline
- **Result**: Auto-build and deploy to dev-eks

**Staging (Automatic):**
- `release/*` branches - release candidate branches
- **Result**: Auto-build and deploy to stage-eks

**Production (Manual only):**
- `main` branch webhooks are DISABLED for safety
- **Requirement**: Must manually trigger pipeline with git tag
- **Result**: Requires deliberate action to deploy to prod-eks

### Debugging Webhook Issues

**Issue: Pipeline not triggering on push**
```bash
# Check GitHub webhook delivery logs
GitHub → Repository → Settings → Webhooks → kubernetes-deploy-webhook → Recent Deliveries

# Verify Jenkins webhook endpoint is reachable
curl -X POST "http://<jenkins-hostname>:8080/generic-webhook-trigger/invoke?token=kubernetes-deploy-webhook" \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/feature/test"}'

# Check Jenkins logs
tail -f /var/log/jenkins/jenkins.log | grep webhook
```

**Issue: Branch detected incorrectly**
```bash
# Jenkins shows "Unknown branch" or wrong environment
# Verify in Jenkins console output:
#   echo "Environment: ${ENVIRONMENT}"
#   echo "Branch: feature/my-feature"

# If branch looks wrong, check:
cd /var/lib/jenkins/workspace/<job-name>
git rev-parse --abbrev-ref HEAD
```

## Release Management & RC Promotion

### How to Find the Latest RC Tag

When you push to `release/1.2.0` branch, each push creates an RC tag. **How do you know which RC number to promote to production?**

**Option 1: Check Jenkins Console Output (Easiest)**

After your release branch builds and deploys to stage-eks, check the Jenkins build output. At the end, you'll see:

```
=========================================
DEPLOYMENT COMPLETED SUCCESSFULLY
=========================================

📋 DEPLOYMENT INFO
   Environment: stage
   Image Tag: 1.2.0-rc42
   Build Number: 42
   Git Branch: release/1.2.0
   Git Commit: abc1234d

🔗 REFERENCE THIS DEPLOYMENT
   To rollback to this build:
   jenkins/Jenkinsfile build #42: IMAGE_TAG=1.2.0-rc42

   Manual trigger command:
   ENVIRONMENT=stage IMAGE_TAG=1.2.0-rc42 SKIP_BUILD=true
```

**Copy the IMAGE_TAG value** - that's the RC version you tested!

**Option 2: Check What's Running on Stage-EKS**

```bash
# See what image tag is currently deployed to stage-eks
kubectl get deployment frontend -n app -o jsonpath='{.spec.template.spec.containers[0].image}'

# Output: 408009925873.dkr.ecr.us-east-1.amazonaws.com/frontend:1.2.0-rc42
```

**Option 3: Query ECR for All Available RC Tags**

```bash
# List all RC tags for a service, sorted by most recent
aws ecr describe-images \
  --repository-name frontend \
  --region us-east-1 \
  --query 'sort_by(imageDetails, &imagePushedAt)[*].imageTags[]' \
  --output text | grep -oP '\d+\.\d+\.\d+-rc\d+' | sort -V | tail -5

# Output:
# 1.2.0-rc40
# 1.2.0-rc41
# 1.2.0-rc42  ← Most recent, this is the one on stage-eks
```

### Promoting a Tested RC to Production

Once you've tested an RC on stage-eks and want to promote it to production:

**Step 1: Get the tested RC tag**
- From Jenkins output: `IMAGE_TAG=1.2.0-rc42`

**Step 2: Extract the version from RC tag**
- RC tag: `1.2.0-rc42`
- Extract: `1.2.0`

**Step 3: Create git tag on main branch**

```bash
# Assuming release/1.2.0 has been merged to main
git checkout main
git pull origin main

# Create semantic version tag
git tag v1.2.0

# Push to trigger production deployment
git push origin v1.2.0
```

**Step 4: Production deployment**

Jenkins will automatically:
1. Detect main + git tag
2. Set ENVIRONMENT=prod, IMAGE_TAG=v1.2.0
3. Build images from main branch
4. Push to ECR with tag v1.2.0
5. Wait for production approval
6. Deploy to prod-eks

### Production Promotion Workflow

```
┌─ Release Branch Testing ──────────────────────────────┐
│                                                       │
│  release/1.2.0 push                                  │
│      ↓                                                │
│  Build: 1.2.0-rc42 (auto on webhook)                 │
│      ↓                                                │
│  Deploy to stage-eks (auto)                          │
│      ↓                                                │
│  QA Testing (manual)                                 │
│      ↓                                                │
│  ✅ APPROVED: Ready for production                   │
└──────────────────────────────────────────────────────┘
                      ↓
┌─ Production Release ──────────────────────────────────┐
│                                                       │
│  Merge release/1.2.0 to main (manual via PR)        │
│      ↓                                                │
│  git tag v1.2.0 (on main)                            │
│  git push origin v1.2.0                              │
│      ↓                                                │
│  Build: v1.2.0 (auto on webhook)                     │
│      ↓                                                │
│  Production Approval Prompt (manual approval)        │
│      ↓                                                │
│  Deploy to prod-eks                                  │
│      ↓                                                │
│  v1.2.0 is now live! 🎉                              │
└──────────────────────────────────────────────────────┘
```

### Template: Promote Specific RC to Production

If you want to promote **without rebuilding** (use exact tested RC):

```bash
# Manually trigger Jenkins with:
ENVIRONMENT = prod
IMAGE_TAG = 1.2.0-rc42 (the RC you tested on stage)
SKIP_BUILD = true (use existing tested images)
FORCE_ROLLOUT = true (restart pods if needed)

# Result: Deploys the exact 1.2.0-rc42 images from ECR to prod-eks
# (No rebuild = faster + guaranteed same tested images)
```

**Note**: This is a quick rollback/promotion. Normally you'd create a git tag (v1.2.0) on main for production releases.

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
