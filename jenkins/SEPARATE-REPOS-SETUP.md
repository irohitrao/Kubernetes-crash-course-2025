# Separate Repos Architecture - Setup Guide

## Overview

This guide explains how to split the monorepo into **two separate repositories** with **two separate Jenkins jobs** for cleaner CI/CD:

```
┌─────────────────────────────────────┐
│   Repository 1: app-source-code     │
│  (frontend, auth-service, (game-service)   │
│                                     │
│  Job 1: Build Images & Push ECR     │
│  ├─ Trigger: Code changes webhook   │
│  ├─ Build Docker images             │
│  ├─ Push to ECR                     │
│  └─ Invoke Job 2 with IMAGE_TAG ──┐│
└─────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────┐
│ Repository 2: kubernetes-infrastructure │
│ (manifests, ansible, helm, etc)    │
│                                     │
│ Job 2: Deploy to K8s Clusters       │
│ Trigger 1: Manifest changes webhook │
│ Trigger 2: Programmatic from Job 1 ◄──┘
│                                     │
│ ├─ Receive IMAGE_TAG from Job 1    │
│ ├─ Run Ansible playbooks           │
│ ├─ Update K8s deployments          │
│ └─ Verify deployment               │
└─────────────────────────────────────┘
```

## Key Benefits

✅ **No unnecessary rebuilds** - Manifest changes trigger only deployment, not image builds
✅ **Clean separation of concerns** - Build logic vs. deployment logic
✅ **Independent triggers** - Dev, stage, prod have separate webhook triggers
✅ **Atomic versioning** - IMAGE_TAG links builds to deployments
✅ **Production safety** - Explicit approval gate before prod deployment

---

## Step 1: Split Repositories

### Create Repository 1: `app-source-code`

```bash
# Clone or create new repo
mkdir app-source-code
cd app-source-code
git init

# Copy these directories from current monorepo
cp -r frontend/
cp -r auth-service/
cp -r game-service/

# Create Jenkinsfile
mkdir -p jenkins
cp jenkins/Job1-app-source-code-Jenkinsfile jenkins/Jenkinsfile

# Create minimal README
cat > README.md << 'EOF'
# Application Source Code

Contains frontend, auth-service, and game-service source code.

## CI Pipeline
Job 1: Build Docker images and push to ECR, then trigger deployment

### Webhook Trigger
- **URL**: `http://jenkins:8080/generic-webhook-trigger/invoke?token=app-ci-webhook`
- **Events**: Push to `develop`, `feature/*`, `main` branches
- **Payload**: JSON (GitHub default)
EOF

git add .
git commit -m "Initial commit: application source code"
git remote add origin https://github.com/YOUR_ORG/app-source-code.git
git push -u origin main
```

### Create Repository 2: `kubernetes-infrastructure`

```bash
# Clone or create new repo
mkdir kubernetes-infrastructure
cd kubernetes-infrastructure
git init

# Copy these directories from current monorepo
cp -r manifests/
cp -r ansible/
cp -r configmaps/
cp -r deployments/
cp -r services/
cp -r volumes/

# Create Jenkinsfile
mkdir -p jenkins
cp jenkins/Job2-kubernetes-infrastructure-Jenkinsfile jenkins/Jenkinsfile

# Create minimal README
cat > README.md << 'EOF'
# Kubernetes Infrastructure

Contains K8s manifests, Ansible playbooks, and deployment configurations.

## CD Pipeline
Job 2: Deploy applications to K8s clusters

### Webhook Triggers
- **URL**: `http://jenkins:8080/generic-webhook-trigger/invoke?token=k8s-infra-webhook`
- **Events**: Push to `release/*`, `main` branches
- **Payload**: JSON (GitHub default)

### Programmatic Trigger
Invoked by Job 1 after successful build with:
- IMAGE_TAG: Tag from Docker image build
- ENVIRONMENT: dev/stage/prod
EOF

git add .
git commit -m "Initial commit: kubernetes infrastructure"
git remote add origin https://github.com/YOUR_ORG/kubernetes-infrastructure.git
git push -u origin main
```

---

## Step 2: Configure Jenkins Jobs

### Create Job 1: `app-source-code-build`

1. **Jenkins Dashboard** → **+ New Item**
2. **Name**: `app-source-code-build`
3. **Type**: Pipeline
4. **Pipeline section**:
   ```groovy
   pipeline {
       agent any
       stages {
           stage('Checkout') {
               steps {
                   checkout([
                       $class: 'GitSCM',
                       branches: [[name: '*/*']],
                       userRemoteConfigs: [[url: 'https://github.com/YOUR_ORG/app-source-code.git']]
                   ])
               }
           }
       }
   }
   ```
5. **Build Triggers**: Check **"Generic Webhook Trigger"**
   - **Token**: `app-ci-webhook`
   - **Post content parameters**: 
     - Name: `ref`, Value: `$.ref`
   - **Regexp filter**: `^(refs/heads/develop|refs/heads/feature/.*|refs/heads/main)$`

6. **Pipeline** → **Definition**: **Pipeline script from SCM**
   - **SCM**: Git
   - **Repository URL**: `https://github.com/YOUR_ORG/app-source-code.git`
   - **Credentials**: (select your GitHub credential if private)
   - **Script Path**: `jenkins/Jenkinsfile`

### Create Job 2: `kubernetes-infrastructure-deploy`

1. **Jenkins Dashboard** → **+ New Item**
2. **Name**: `kubernetes-infrastructure-deploy`
3. **Type**: Pipeline
4. **Parameters** (add these in Parameters section):
   - **String parameter**:
     - Name: `IMAGE_TAG`
     - Default: (empty)
     - Description: `Image tag to deploy`
   - **Choice parameter**:
     - Name: `ENVIRONMENT`
     - Choices: `dev`, `stage`, `prod`
     - Default: `dev`

5. **Build Triggers**: Check **"Generic Webhook Trigger"**
   - **Token**: `k8s-infra-webhook`
   - **Regexp filter**: `^(refs/heads/release/.*|refs/heads/main)$`
   
   (Also receives parameters from Job 1 programmatically)

6. **Pipeline** → **Definition**: **Pipeline script from SCM**
   - **SCM**: Git
   - **Repository URL**: `https://github.com/YOUR_ORG/kubernetes-infrastructure.git`
   - **Script Path**: `jenkins/Jenkinsfile`

---

## Step 3: Configure GitHub Webhooks

### For Repository 1 (app-source-code)

1. Go to **Settings** → **Webhooks** → **Add webhook**
2. **Payload URL**: `http://JENKINS_HOST:8080/generic-webhook-trigger/invoke?token=app-ci-webhook`
3. **Content type**: `application/json`
4. **Events**: 
   - ✅ Push events
   - ✅ Pull requests (optional)
5. **Active**: ✅ Yes
6. Click **Add webhook**

### For Repository 2 (kubernetes-infrastructure)

1. Go to **Settings** → **Webhooks** → **Add webhook**
2. **Payload URL**: `http://JENKINS_HOST:8080/generic-webhook-trigger/invoke?token=k8s-infra-webhook`
3. **Content type**: `application/json`
4. **Events**: 
   - ✅ Push events
5. **Active**: ✅ Yes
6. Click **Add webhook**

---

## Step 4: Trigger Flow

### Scenario 1: Code Change (No Manifest Change)

```
Developer pushes to feature/add-feature branch
  ↓
GitHub webhook → Job 1 (app-ci-webhook)
  ↓
Job 1 runs:
  • Detects branch = feature/* → ENVIRONMENT=dev
  • Builds Docker images with SHA tag: abc1234d
  • Pushes to ECR
  • Invokes Job 2 with:
    - IMAGE_TAG=abc1234d
    - ENVIRONMENT=dev
  ↓
Job 2 runs:
  • Deploys frontend:abc1234d, auth:abc1234d, game:abc1234d
  • Runs Ansible playbooks
  • Verifies deployment on dev-eks
```

### Scenario 2: Manifest Change (No Code Change)

```
DevOps updates manifests/ or ansible/
  ↓
GitHub webhook → Job 2 (k8s-infra-webhook)
  ↓
Job 2 runs:
  • Detects branch = release/1.2.0 → ENVIRONMENT=stage
  • IMAGE_TAG not provided (empty) → queries ECR for latest
  • Uses currently deployed image version
  • Runs updated Ansible playbooks
  • Verifies deployment on stage-eks
  ↓
✅ NO image rebuild! Manual deployment only
```

### Scenario 3: Tag & Release (For Production)

```
Developer creates release on main
  ↓
Steps:
1. Push tag: git tag v1.2.0 && git push origin --tags
2. Creates release/1.2.0 branch from main
3. Commits to release/1.2.0
  ↓
GitHub webhook → Job 1 (app-ci-webhook)
  ↓
Job 1 runs:
  • Detects branch = release/* → ENVIRONMENT=stage (or main → ENVIRONMENT=prod)
  • Builds images with tag: v1.2.0
  • Pushes to ECR
  • Invokes Job 2 with IMAGE_TAG=v1.2.0, ENVIRONMENT=prod
  ↓
Job 2 runs:
  • Prompts: "Deploy to PRODUCTION? Approve?"
  • After approval, deploys v1.2.0 to prod-eks
```

---

## Step 5: Testing

### Test Job 1 Webhook

```bash
# In app-source-code repo
git checkout -b feature/test-webhook
echo "test" >> README.md
git add .
git commit -m "Test webhook"
git push origin feature/test-webhook

# Check Jenkins: Job 1 should trigger automatically
# Visit Jenkins dashboard → app-source-code-build → Build History
```

### Test Job 2 Manifest Trigger

```bash
# In kubernetes-infrastructure repo
git checkout main
cd manifests
# Make small change to a manifest
echo "# Test" >> frontend.yaml
git add .
git commit -m "Update frontend manifest"
git push origin main

# Check Jenkins: Job 2 should trigger automatically
# Visit Jenkins dashboard → kubernetes-infrastructure-deploy → Build History
```

### Manual Trigger (Rollback)

```bash
# If you want to manually trigger Job 2 with an existing image:
curl -X POST \
  http://JENKINS_HOST:8080/job/kubernetes-infrastructure-deploy/buildWithParameters \
  -u jenkins-user:API_TOKEN \
  -F 'IMAGE_TAG=1.2.0-rc42' \
  -F 'ENVIRONMENT=stage'
```

---

## Step 6: Version Coordination Between Repos

### How IMAGE_TAG flows:

1. **Job 1** (app-source-code) builds image with tag `abc1234d`
2. **Job 1** pushes to ECR: `123456789.dkr.ecr.us-east-1.amazonaws.com/frontend:abc1234d`
3. **Job 1** stores `IMAGE_TAG=abc1234d` as Jenkins environment variable
4. **Job 1** invokes **Job 2** with parameter: `IMAGE_TAG=abc1234d`
5. **Job 2** receives `IMAGE_TAG=abc1234d` and uses it in Ansible playbooks
6. Ansible renders manifests with: `image: 123456789.dkr.ecr.us-east-1.amazonaws.com/frontend:abc1234d`

### Git Tag Strategy (for prod):

```bash
# In app-source-code repo, before creating release:
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin --tags

# Job 1 detects this tag and uses it:
# IMAGE_TAG=v1.2.0 (instead of commit SHA)
# Sent to Job 2 for production deployment
```

---

## Environment Auto-Detection

| Branch | Job 1 Sets | Job 2 Deploys To |
|--------|-----------|-----------------|
| `feature/*` | ENVIRONMENT=dev | dev-eks |
| `develop` | ENVIRONMENT=dev | dev-eks |
| `release/*` | ENVIRONMENT=stage | stage-eks |
| `main` | ENVIRONMENT=prod | prod-eks |

---

## Credentials Needed

Both jobs need these Jenkins credentials configured:

```
aws-jenkins-credentials
  └─ Type: AWS Credentials
  └─ Access Key: (your AWS key)
  └─ Secret Access Key: (your AWS secret)

kubeconfig-dev
  └─ Type: Secret file
  └─ File: (kubeconfig for dev-eks)

kubeconfig-stage
  └─ Type: Secret file
  └─ File: (kubeconfig for stage-eks)

kubeconfig-prod
  └─ Type: Secret file
  └─ File: (kubeconfig for prod-eks)
```

**To add in Jenkins**:
1. **Manage Jenkins** → **Manage Credentials** → **System** → **Global credentials**
2. **+ Add Credentials**
3. Configure each credential

---

## Rollback Scenarios

### Rollback Last Deployment (Same IMAGE_TAG)

```bash
# Just re-run Job 2 with same IMAGE_TAG
curl -X POST \
  http://JENKINS_HOST:8080/job/kubernetes-infrastructure-deploy/buildWithParameters \
  -u jenkins-user:API_TOKEN \
  -F 'IMAGE_TAG=v1.2.0' \
  -F 'ENVIRONMENT=prod'
```

### Rollback to Previous Build

```bash
# Check Job 2 build history, find previous successful build
# Note the IMAGE_TAG from that build
# Trigger with that IMAGE_TAG

# Example: Previous was rc42
curl -X POST \
  http://JENKINS_HOST:8080/job/kubernetes-infrastructure-deploy/buildWithParameters \
  -u jenkins-user:API_TOKEN \
  -F 'IMAGE_TAG=1.2.0-rc42' \
  -F 'ENVIRONMENT=prod'
```

---

## Troubleshooting

### Job 1 builds but Job 2 doesn't trigger

- Check: Is `build job: 'kubernetes-infrastructure-deploy'` name correct?
- Check: Does Job 2 exist in Jenkins?
- Check: Jenkins logs: `/var/log/jenkins/jenkins.log`

### Webhook not firing

- Check: Is webhook active in GitHub? (Settings → Webhooks → green checkmark)
- Check: Is token correct? (`app-ci-webhook` vs `k8s-infra-webhook`)
- Check: Jenkins → Manage → System Log → watch for webhook hits
- Test webhook in GitHub: Settings → Webhooks → Recent Deliveries → click → Redeliver

### IMAGE_TAG is empty in Job 2

- If triggered via manifest change webhook, IMAGE_TAG will be empty (expected)
- Job 2 queries ECR for latest image automatically
- Check: `aws ecr describe-images` command output

---

## Next Steps

1. ✅ Create both repositories
2. ✅ Copy files to each repo
3. ✅ Configure Jenkins Job 1 and Job 2
4. ✅ Add GitHub webhooks
5. ✅ Test with feature branch push
6. ✅ Test with manifest change push
7. ✅ Configure credentials
8. ✅ Test rollback scenario
