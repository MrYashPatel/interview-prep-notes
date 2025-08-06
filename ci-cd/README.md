Great — we’ll now build your **CI/CD mastery** with:

### 🔹 PHASE 1: CI using GitHub Actions (Beginner → Expert)

* Workflow structure
* Linting, unit testing, security scanning (Trivy, TFLint, tfsec, Checkov)
* Build → Docker → push (GHCR/ECR)
* Reusable workflows, matrix builds, composite actions
* GitHub secrets, OIDC with AWS, fine-grained permissions
* Scenario: CI pipeline failing due to secret rotation, image not building, test flakiness


Bhai, tu serious CI/CD warrior banne nikla hai — let's crush **PHASE 1: CI using GitHub Actions** step-by-step from **Beginner → Expert**, covering exactly what you need with examples, real-world issues, and solutions 🔥

---

## ✅ Step 1: Basic Workflow Structure

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js / Python / Go (based on your stack)
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Install Dependencies
        run: npm install
      - name: Run Tests
        run: npm test
```

➡️ This is your **CI base**. Now we start layering expert features.

---

## ✅ Step 2: Add Linting, Unit Testing, Security Scanning

### 🧹 Linting

```yaml
- name: Run ESLint
  run: npm run lint
```

Or for Terraform:

```yaml
- name: Run TFLint
  uses: terraform-linters/setup-tflint@v4
```

---

### ✅ Unit Testing with Coverage

```yaml
- name: Run Tests with Coverage
  run: npm test -- --coverage
```

---

### 🔐 Security Scanning

#### 1. **Trivy** (Container + Code scanning)

```yaml
- name: Trivy scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ghcr.io/your-org/app:latest
```

#### 2. **tfsec** (Terraform)

```yaml
- name: tfsec scan
  uses: aquasecurity/tfsec-action@v1.0.0
```

#### 3. **Checkov**

```yaml
- name: Checkov
  uses: bridgecrewio/checkov-action@v12
```

---

## ✅ Step 3: Docker Build & Push (GHCR/ECR)

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build & Push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository_owner }}/myapp:latest
```

> For ECR: use `aws-actions/amazon-ecr-login@v2` and `aws-actions/configure-aws-credentials`.

---

## ✅ Step 4: Reusable Workflows & Composite Actions

### 🧩 Reusable Workflow

📁 `.github/workflows/docker-build.yml`

```yaml
on: workflow_call
jobs:
  docker:
    ...
```

📁 In another workflow:

```yaml
jobs:
  call-docker:
    uses: ./.github/workflows/docker-build.yml
```

---

### ⚙️ Composite Action

📁 `.github/actions/setup-env/action.yml`

```yaml
runs:
  using: "composite"
  steps:
    - name: Setup Go
      uses: actions/setup-go@v5
      with:
        go-version: 1.22
```

Use in workflow:

```yaml
- uses: ./.github/actions/setup-env
```

---

## ✅ Step 5: GitHub Secrets, OIDC, Fine-Grained Access

### 🔐 GitHub Secrets

* Store secrets in repo → `Settings > Secrets and Variables > Actions`
* Use: `${{ secrets.AWS_ACCESS_KEY_ID }}`

---

### 🔓 OIDC with AWS

Use OIDC to assume IAM role securely (no secrets):

```yaml
- name: Configure AWS OIDC
  uses: aws-actions/configure-aws-credentials@v3
  with:
    role-to-assume: arn:aws:iam::<account>:role/github-oidc-role
    aws-region: us-east-1
```

🔐 Requires:

* OIDC trust policy in AWS
* IAM role with actions (ECR push, S3 write, etc.)

---

## 💥 SCENARIOS (Real-World CI Failures & Fixes)

---

### 🚨 CI Failing: Secret Rotated

**Problem:** Your CI pipeline fails to deploy because the AWS/ECR secret is rotated.

**Fix:**

* Use **OIDC** instead of long-lived secrets.
* If secrets are still used, configure **GitHub Secret Scanning Alerts**
* Set up **secret expiration alerts** via AWS/HashiCorp Vault

---

### 🐳 Docker Build Fails in CI

**Problem:** Image not building in CI but works locally.

**Fix:**

* Add `--no-cache` in Dockerfile
* Check `.dockerignore` – you might be missing key files
* Use `buildx` to debug:

  ```yaml
  uses: docker/setup-buildx-action@v3
  ```

---

### 🧪 Test Flakiness

**Problem:** Random test failures.

**Fix:**

* Add `--runInBand` for Jest to avoid parallel flaky tests
* Retry strategy:

  ```yaml
  - run: npm test
    retry: 3
  ```
* Add test retry plugin or mark tests `@flaky` and isolate

---

## 📦 Sample Folder Structure (Best Practice)

```
.github/
├── workflows/
│   ├── ci.yml
│   ├── docker-build.yml
│   └── reusable-test.yml
├── actions/
│   └── setup-env/
│       └── action.yml
```

---

## 🧠 Interview Questions You’ll Face

---

### Q: How do you secure secret access in GitHub Actions?

**Answer:**
Use **OIDC with short-lived tokens** to assume IAM roles. Avoid hardcoded secrets. Use fine-grained **repo-level or org-level secrets** with minimum scope.

---

### Q: Why did your CI pipeline randomly fail last week?

**Answer:**
It was a flaky test case due to parallel runs in Jest. I fixed it using `--runInBand` and implemented retry logic.

---

### Q: How to reuse common CI logic in multiple workflows?

**Answer:**
Via **reusable workflows** (`workflow_call`) or **composite actions** stored in `.github/actions`.

---

## 📌 What’s Next?

Let me know if you want any of the following:

1. 🔨 Full working CI pipeline repo with all these features
2. 🚀 Setup to deploy to EKS after this
3. 🧪 Advanced testing (integration, coverage, report uploads)
4. 📊 Integrate with Slack for alerts or CodeCov

---
---
---




### 🔹 PHASE 2: CD using Argo CD (Beginner → Expert)

* GitOps model
* Application CRs, sync methods
* App-of-apps, multi-cluster deployment
* RBAC, OIDC, multi-tenancy, drift detection
* Helm/Kustomize integration
* Progressive delivery (canary, blue/green)
* Scenario: App stuck in pending, drift issues, sync order problem, multi-cluster strategy

---

Great! You're now set to master **Phase 2: CD using Argo CD** in depth. Here's how we'll break it down:

---

### 🔷 **Argo CD – Beginner to Expert Roadmap**

#### ✅ **1. GitOps Model**

* Declarative desired state in Git
* Argo CD as the **pull-based** CD tool
* Git is the single source of truth
* Reconciliation loops

---

#### ✅ **2. Application CRs (Custom Resources)**

* `Application` object: maps Git repo to K8s namespace
* Key fields: `spec.source`, `spec.destination`, `spec.syncPolicy`, `spec.project`
* Health and sync status tracked by Argo CD

---

#### ✅ **3. Sync Methods**

* **Manual Sync**: `argocd app sync`
* **Auto Sync**: `automated: { prune: true, selfHeal: true }`
* **Hooks**: PreSync, Sync, PostSync (used for migrations, DB changes, etc.)

---

#### ✅ **4. App-of-Apps Pattern**

* One "Parent" app declares multiple child `Application` CRs
* Used for managing microservices, teams, or environments
* Best with `kustomize` directory structure or Helm charts

---

#### ✅ **5. Multi-Cluster Deployment**

* Argo CD supports multiple clusters via `argocd cluster add`
* Each `Application` CR can target a different `cluster + namespace`
* Use labels and projects for access control and grouping

---

#### ✅ **6. RBAC, OIDC, Multi-Tenancy**

* RBAC: based on Argo CD roles + JWT
* OIDC: integrates with GitHub, Okta, Google, etc.
* Projects: isolate apps by team or environment
* Prevent namespace/cluster overlap via project restrictions

---

#### ✅ **7. Drift Detection**

* Auto detects drift between Git and live cluster
* Can auto-revert or alert via Slack
* Use `resource.exclusions` to avoid noise

---

#### ✅ **8. Helm / Kustomize Integration**

* Supports:

  * Helm 3 charts
  * Helm value overrides
  * Kustomize with overlays
* Declarative setup in Git → Helm/Kustomize → Argo CD Application

---

#### ✅ **9. Progressive Delivery**

* **Canary Deployment**: Manual promotion or with tools like Argo Rollouts
* **Blue/Green**: Switch services between live and preview
* Integrate with Ingress controllers or Service Mesh (Istio/Linkerd)

---

### 🔶 **Real-World Scenarios & Troubleshooting**

#### 🚨 Scenario 1: App stuck in `Pending`

* Check if cluster is added and accessible
* Validate `spec.source.path` and repo credentials
* Inspect sync policy errors

#### 🚨 Scenario 2: Drift Detected but Not Syncing

* Auto-sync might be disabled
* Hook or manifest might block sync
* Check for pruning-related conflicts

#### 🚨 Scenario 3: Sync Order Problem

* Use `dependsOn` annotation in App-of-Apps
* Control order using Helm `hooks` or resource `wait`

#### 🚨 Scenario 4: Multi-cluster Strategy

* Add multiple clusters: `argocd cluster add`
* Isolate per Application or Project
* Use Argo CD Projects to manage multi-team/multi-tenant access

---
---
---


### 🔹 PHASE 3: CI/CD Integration

* GitHub Actions → Argo CD trigger (Webhooks, Argo CD auto-sync, commit conventions)
* Promote builds from dev → staging → prod using Git tags or branches
* Safe rollout using Argo CD + notifications + Slack integration

---

### 🔹 PHASE 4: Expert + Real-World Challenges

* Handle GitOps drift due to manual kubectl apply
* Secure CD pipelines using OIDC + IRSA
* Argo CD for 200+ microservices (App-of-apps + sync waves)
* Handling failed deployments, hooks, rollback strategies
* CI tests slow? Use parallel matrix jobs, caching, fail-fast

---

