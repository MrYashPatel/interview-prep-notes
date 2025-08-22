# AWS Service Connection Best Practices

---

| Service            | Connects To             | What We Need (Integration Requirements)         | Auth Mechanism (Best Practice)            |
| ------------------ | ----------------------- | ----------------------------------------------- | ----------------------------------------- |
| **EC2**            | S3                      | IAM Role with S3 access, VPC endpoint (private) | **Instance IAM Role** (Direct IAM)        |
|                    | ECR                     | IAM Role with `ecr:*`, Docker login             | **Instance IAM Role**                     |
|                    | RDS                     | SG rules (EC2 → RDS), DB creds                  | **Secrets Manager** (not hardcoded)       |
|                    | EKS                     | `kubectl` config + IAM auth                     | **IAM Role** mapped via aws-auth          |
|                    | ALB                     | Target group registration                       | **SG + IAM**                              |
|                    | ASG                     | Launch template with IAM instance profile       | **Instance IAM Role**                     |
| **S3**             | Lambda                  | Event notification (S3 → Lambda)                | **Execution Role (IAM)**                  |
|                    | CloudFront              | S3 as origin + OAC (Origin Access Control)      | **OAC/OAI**, not public bucket            |
|                    | ECS/EKS                 | Task Role/IRSA with S3 access                   | **IRSA (EKS)** / **Task Role (ECS)**      |
| **Lambda**         | S3                      | Trigger + IAM Role                              | **Execution Role**                        |
|                    | RDS                     | VPC + SG rules + DB creds                       | **Secrets Manager**                       |
|                    | ECR                     | Container image pull                            | **Execution Role with ECR access**        |
|                    | ALB                     | Lambda target integration                       | **IAM Role for Lambda Execution**         |
| **ECS**            | ECR                     | Task Execution Role with pull perms             | **IAM Task Execution Role**               |
|                    | RDS                     | SG rules + DB creds                             | **Secrets Manager**                       |
|                    | S3                      | Task Role with S3 perms                         | **IAM Task Role**                         |
|                    | ALB                     | Service registered to Target Group              | **SG + IAM**                              |
|                    | ASG                     | ECS Capacity Provider scaling                   | **IAM Role for ECS Service Auto Scaling** |
| **ECR**            | ECS/EKS                 | Pull images                                     | **Task Role (ECS)** / **IRSA (EKS)**      |
|                    | Lambda                  | Container-based Lambda                          | **Execution Role**                        |
|                    | GitHub Actions / Docker | Push images                                     | **OIDC (best)** or IAM User creds         |
| **EKS**            | ECR                     | Nodes pull from ECR                             | **IRSA** / **Node IAM Role**              |
|                    | S3                      | Pods read/write S3                              | **IRSA**                                  |
|                    | RDS                     | Same VPC + SG + DB creds                        | **IRSA + Secrets Manager**                |
|                    | ALB                     | Ingress via LB Controller                       | **OIDC Provider (IRSA)**                  |
|                    | ASG                     | Nodegroup auto scaling                          | **Node IAM Role**                         |
|                    | GitHub Actions          | Deploy workloads                                | **OIDC (GitHub → AWS)**                   |
| **RDS**            | EC2                     | SG inbound + creds                              | **Secrets Manager**                       |
|                    | ECS/EKS                 | SG inbound + creds                              | **Secrets Manager + IAM Auth (Aurora)**   |
|                    | Lambda                  | VPC + creds                                     | **Secrets Manager**                       |
| **GitHub**         | GitHub Actions          | Built-in CI/CD                                  | Native                                    |
|                    | AWS                     | Deploy infra (EKS, ECS, Lambda, etc.)           | **OIDC Provider (best)**                  |
| **Docker**         | ECR                     | Login & push/pull images                        | **OIDC → ECR Login**                      |
|                    | ECS/EKS                 | Deploy containers                               | Uses **ECR creds via Role/IRSA**          |
|                    | GitHub Actions          | Build & push images                             | **OIDC (preferred)**                      |
| **GitHub Actions** | AWS (all services)      | Deploy apps/infrastructure                      | **OIDC Federation** (no static keys)      |
|                    | Docker/ECR              | Build & push images                             | **OIDC + AWS Role**                       |
| **CloudFront**     | S3                      | Origin with OAC/OAI                             | **OAC (modern best practice)**            |
|                    | ALB/ECS/EKS             | ALB as origin                                   | **IAM/Cert Manager for HTTPS**            |
| **ALB**            | EC2/ECS/EKS             | Register targets                                | **IAM + SG**                              |
|                    | Lambda                  | Lambda target integration                       | **Execution Role**                        |
|                    | CloudFront              | ALB as origin                                   | **TLS Certs**                             |
| **ASG**            | EC2                     | Scale EC2 nodes                                 | **Launch Template IAM Role**              |
|                    | ECS                     | ECS capacity provider                           | **IAM Role for ECS Auto Scaling**         |
|                    | EKS                     | Worker node scaling                             | **Node IAM Role**                         |
|                    | ALB                     | Auto register targets                           | **Target Group Binding**                  |

---

### 🔑 Highlights (Auth & Security Best Practices):

* **IRSA** → EKS pods ko S3, RDS, ECR access dene ka best practice.
* **OIDC** → GitHub Actions → AWS integration ke liye (no static IAM creds).
* **Direct IAM Role** → EC2, ECS Tasks, Lambda execution ke liye.
* **Secrets Manager** → DB credentials, API keys, sensitive values ke liye.
* **OAC (CloudFront)** → S3 ko private rakhne ke liye.
* **SG + VPC Endpoints** → Secure networking ke liye.

---

