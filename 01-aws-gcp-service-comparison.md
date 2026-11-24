# AWS vs GCP — DevOps Service Comparison

## Core service mapping

| Category | AWS | GCP | Notes |
|---|---|---|---|
| Compute (VMs) | EC2 | Compute Engine | Near 1:1. AWS has more instance variety; GCE has live migration and per-second billing. |
| Containers (managed) | ECS, EKS, Fargate | GKE, Cloud Run | GKE is generally the most polished managed Kubernetes. Cloud Run competes with Fargate + App Runner. |
| Serverless | Lambda | Cloud Run functions (formerly Cloud Functions) | GCP folded Functions into Cloud Run in 2024. |
| IaC (native) | CloudFormation, CDK | Deployment Manager (legacy), Infrastructure Manager | In practice most teams use Terraform/OpenTofu or Pulumi across both clouds. |
| CI/CD | CodePipeline, CodeBuild, CodeDeploy | Cloud Build, Cloud Deploy, Artifact Registry | CodePipeline is clunkier; Cloud Build is a single-step YAML pipeline. Many teams use GitHub Actions or GitLab anyway. |
| Config management | SSM (State Manager, Patch Manager) | OS Config, VM Manager | |
| Container registry | ECR | Artifact Registry | Artifact Registry replaced Container Registry (GCR is shut down). |
| Networking | VPC, Route 53, CloudFront, ELB | VPC, Cloud DNS, Cloud CDN, Cloud Load Balancing | GCP VPCs are global by default; AWS VPCs are regional. Big mental-model difference. |
| Storage | S3, EBS, EFS | Cloud Storage, Persistent Disk, Filestore | GCS has stronger built-in lifecycle + single global namespace. |
| Databases | RDS, Aurora, DynamoDB, ElastiCache | Cloud SQL, AlloyDB, Firestore, Memorystore | Cloud SQL ≈ RDS; Aurora ≈ AlloyDB; DynamoDB vs Firestore; Bigtable for high throughput. |
| Monitoring/logging | CloudWatch, X-Ray | Cloud Monitoring, Cloud Logging, Cloud Trace | GCP ops suite is more coherent; CloudWatch now covers logs+metrics+traces in one product. |
| Security/identity | IAM, KMS, Secrets Manager, GuardDuty | IAM, Cloud KMS, Secret Manager, Security Command Center | AWS IAM allows explicit Deny; GCP has roles + resource hierarchy (org → folder → project). |
| Cost | AWS Budgets, Cost Explorer | Budgets, Pricing Calculator | |
| Git hosting | CodeCommit (deprecated for new customers since 2024) | — (never had one) | Use GitHub/GitLab externally. |

## Architectural differences to internalize

1. **Resource hierarchy**: GCP has org → folder → project; policy inherits downward. AWS analogue: accounts + Organizations + SCPs.
2. **VPC scope**: AWS VPC is regional with subnets per AZ; GCP VPC is global with subnets per region.
3. **IAM model**: GCP uses roles (primitive, predefined, custom) with no explicit deny; AWS allows explicit Deny — powerful but a common trap.
4. **Load balancers**: GCP's external LB is one proxy doing L4 and L7; AWS splits ALB / NLB / GWLB.

## Certification targets (optional)

- **AWS DevOps Engineer Professional (DOP-C02)** — deep, matches 6 yrs experience.
- **GCP Professional Cloud DevOps Engineer** — lighter, covers GCP side.
- Both are achievable within 3–4 months of focused study after this refresher.
