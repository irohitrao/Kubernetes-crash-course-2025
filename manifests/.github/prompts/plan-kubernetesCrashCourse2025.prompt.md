## Plan: Jenkins Ansible EKS Deployment Pipeline

Use Jenkins as the control plane for CI/CD and Ansible as the deployment orchestrator for EKS. Recommended approach: keep the current Kubernetes manifests as the deployment source, but introduce environment-specific variables in Ansible inventory/group vars so Jenkins can build images, push them to ECR, render/apply manifests in dependency order, create required Kubernetes secrets, and verify rollout state. Separate cluster prerequisite bootstrap from application rollout so the app pipeline remains repeatable and focused.

**Steps**
1. Phase 1: Define pipeline boundaries and prerequisites.
   Confirm that the pipeline is responsible for application build, image push, secret creation, manifest rendering, deployment, and validation.
   Keep EKS cluster creation, DNS registration, and operator installation as a prerequisite track, not part of the regular app rollout.
   Prerequisites to verify before the first successful deploy: EKS access for Jenkins/Ansible, AWS ECR access, cert-manager installed, CloudNativePG operator installed, Gateway API/kgateway installed, Prometheus operator installed if ServiceMonitor is required, AWS EBS CSI driver installed, and DNS for the application domain pointed at the ingress/gateway load balancer.
2. Phase 2: Organize repository and pipeline inputs.
   Create a deployment structure where Jenkins invokes Ansible with an environment selector such as dev, stage, or prod.
   Keep environment-specific values outside the manifests in Ansible vars: AWS account ID, AWS region, ECR repository/image names, Kubernetes namespace, domain, email for ClusterIssuer, database credentials secret values, and optional image tags.
   Because the current workspace is only the manifests directory, confirm that Jenkins checks out the full repository containing frontend/auth/game source before attempting image builds. If Jenkins only checks out manifests, split build and deploy into separate jobs and pass image tags into the deploy job.
3. Phase 3: Build and publish application images.
   Build frontend, auth, and game images from the application source.
   Tag images using an immutable identifier such as commit SHA and optionally a promoted environment tag.
   Authenticate Jenkins to AWS and push images to ECR.
   Export the produced image tags as pipeline variables/artifacts so Ansible can inject them into the deployment.
   This phase blocks deployment.
4. Phase 4: Prepare Ansible deployment logic.
   Create an Ansible playbook structure with roles or task files for: prerequisite checks, namespace/secret management, manifest rendering/apply, wait conditions, and post-deploy validation.
   Use Ansible inventory or group_vars for dev/stage/prod values.
   Render the existing manifests with variable substitution rather than maintaining separate copies per environment.
   Recommended deployment model (split for DB safety):
   - Bootstrap pipeline (rare/manual): apply StorageClass, namespace, ConfigMap, PostgreSQL Cluster, and first-time secret setup.
   - Application release pipeline (frequent/automated): deploy app Services/Deployments, Gateway/HTTPRoute, and optional ServiceMonitor; do not re-apply PostgreSQL Cluster by default.
   Safe apply order in the application release pipeline:
   - create or verify the application namespace
   - create/update postgres-app Kubernetes Secret from Jenkins credentials
   - apply Services and Deployments for auth, game, and frontend using the just-built image tags
   - apply Gateway and HTTPRoute resources
   - apply ServiceMonitor last, or make it conditional on Prometheus operator availability
   PostgreSQL Cluster changes should run only in a dedicated DB change job with manual approval, backup/snapshot checks, and maintenance-window controls.
5. Phase 5: Add explicit readiness gates and safe rollout checks.
   After PostgreSQL apply, wait for CloudNativePG cluster readiness and verify bootstrap completed before deploying dependent services.
   After workload apply, wait for Deployments to become ready and ensure Services have endpoints.
   After Gateway/HTTPRoute apply, wait for Gateway programmed state, external address allocation, and accepted route status.
   After cert-manager reconciliation, verify the TLS secret exists and the certificate is issued.
   Fail the pipeline early on missing CRDs, missing namespaces, missing AWS auth, or unresolved DNS prerequisites.
6. Phase 6: Support multiple environments cleanly.
   Use one Jenkins pipeline with an environment parameter and branch/tag rules for promotion.
   Recommended pattern:
   - development deploys from branch builds
   - stage deploys from selected branches or release candidates
   - production deploys from approved tags or manual promotion
   Keep environment config in Ansible vars, not duplicated manifest files.
   Add a manual approval gate in Jenkins before production deployment.
7. Phase 7: Handle rollback and repeatability.
   Make the deploy idempotent so re-running the same job converges the cluster without manual cleanup.
   Keep previous image tags and deployment metadata in Jenkins build history to allow rollback by re-running Ansible with the last known good image versions.
   For application rollback, prefer redeploying prior image tags rather than trying to roll back database schema automatically.
   Treat database changes as a separate controlled workflow if schema evolution becomes part of future releases.
8. Phase 8: Add verification and operational reporting.
   Publish Jenkins stage results for build, push, deploy, and validate separately.
   Archive rendered manifests and selected deployment metadata per run for traceability.
   Expose basic deployment outputs: image tags used, namespace, gateway address, certificate state, and rollout result.

**Relevant files**
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/storageclass-gp3.yaml — cluster storage prerequisite; decide whether this remains managed by app deploy or by cluster bootstrap.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/configmap.yaml — bootstrap SQL initialization for the PostgreSQL cluster.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/pgcluster.yaml — critical database dependency; pipeline must wait for readiness before app rollout.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/auth-deploy.yaml — app deployment using DB secret and ECR image reference; needs image tag and possibly env templating.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/game-deploy.yaml — same pattern as auth; depends on DB and secret.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/frontend.yaml — frontend image deployment; should consume the promoted image tag.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/auth-service.yaml — service exposure for auth workload.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/game-service.yaml — service exposure for game workload.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/frontend-service.yaml — service exposure for frontend workload.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/gateway.yaml — ingress/gateway entry point; environment-specific host/cert annotations must be parameterized.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/httproute.yaml — main routing rules tied to gateway and domain.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/httpredirect.yaml — ACME/HTTP redirect behavior; validate it does not interfere with certificate issuance.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/cluster-issuer.yaml — cert-manager dependency with environment-specific email/domain values.
- /home/rohity/repo/Kubernetes-crash-course-2025/manifests/servicemonitor.yaml — optional monitoring integration that should be conditional on operator availability.

**Verification**
1. Pre-flight checks in Jenkins or Ansible:
   verify kubectl/Ansible access to the target EKS cluster, AWS authentication for ECR and EKS, presence of required CRDs/operators, and presence or creatability of the target namespace.
2. Build validation:
   confirm all three images build successfully, are pushed to ECR, and the exact image tags are captured for deployment.
3. Database validation:
   verify PostgreSQL cluster reports ready status and database initialization objects/tables are present before app deployment continues.
4. Workload validation:
   verify auth, game, and frontend Deployments reach ready state and Services have populated endpoints.
5. Routing validation:
   verify Gateway is programmed, HTTPRoutes are accepted, the external address is assigned, and the domain resolves correctly.
6. TLS validation:
   verify the certificate is issued and the expected TLS secret exists before marking the deploy successful.
7. Monitoring validation:
   if Prometheus operator is enabled, verify ServiceMonitor is accepted and scrape targets appear.
8. Rollback drill:
   test one non-production redeploy of a previous image tag to prove rollback mechanics work.

**Decisions**
- Included scope: Jenkins-driven build and deploy, Ansible-based manifest orchestration, ECR image publication, Kubernetes secret creation from Jenkins credentials, multi-environment support, and rollout validation.
- Excluded scope: creating the EKS cluster itself, installing foundational operators/controllers on every regular app deploy, and automating database schema rollback.
- Recommended implementation split: one bootstrap path for cluster prerequisites and one regular application release pipeline.
- Recommended templating model: keep one logical manifest set and drive environment differences through Ansible variables rather than copying YAML per environment.

**Further Considerations**
1. The current opened workspace only exposes the manifests folder. If the application source is not available in the Jenkins checkout context, the build stage must either check out the repository root or consume images from upstream build jobs.
2. The postgres-app Kubernetes Secret is referenced by the app manifests but not defined in this folder. The deployment role should create it from Jenkins credentials before app rollout.
3. Several values appear environment-specific in the manifests today: domain, ClusterIssuer email, AWS account/region, and image references. Those should be parameterized first, or the pipeline will only fit one environment reliably.
4. Current storage class uses reclaimPolicy Delete in [storageclass-gp3.yaml](storageclass-gp3.yaml). To reduce blast radius, set production database storage to Retain (or use a dedicated DB StorageClass) and enforce snapshot backups before any DB change job.
