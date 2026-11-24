# Labs — One Per Day, Both Clouds

Each lab: ~4-6 hours. Do the same lab in AWS and GCP, same afternoon, to force the comparison.

---

## Lab 1 (Day 1) — 3-Tier VPC + IAM, CLI only

Build in both clouds:
- VPC with public + 2 private subnets across 2 AZs (AWS) / regions can't be split in GCP VPC — build subnets in 2 regions if you like, or 2 subnets in 1 region (fine).
- NAT (NAT Gateway / Cloud Router + NAT).
- One bastion VM in the public subnet, one app VM in private subnet, verify private VM can reach internet but is not reachable.
- IAM: create a least-privilege role/SA that can only read from the storage service; attach to the private VM and verify.

**Cleanup**: `terraform destroy` or delete VPC manually if CLI-built (CLI-only lab is intentional — muscle memory).

## Lab 2 (Day 2) — Static Site + Spot Batch

- Containerize a tiny app (single Go file or Python Flask "hello").
- Push image to ECR (AWS) and Artifact Registry (GCP).
- Deploy via ECS Fargate (ALB front) and Cloud Run (both public HTTPS endpoints).
- Note: deploy time, cold start behavior, rough per-request cost.

## Lab 3 (Day 3) — Same App, Three Ways

- Static site on S3 / GCS with bucket policy (public or OAC/OIDC), lifecycle rule to IA/Glacier or Nearline/Coldline, CloudFront/CDN in front.
- Spin a Spot (AWS) / Spot VM (GCP), have it process 5 files and exit. Observe termination notice mechanism in each.

## Lab 4 (Day 4) — Full CI/CD Pipeline + Terraform

- Terraform: one module per cloud defining the container infra from Lab 2.
- Pipeline AWS: GitHub push → CodePipeline → CodeBuild (build image, push to ECR) → deploy to ECS.
- Pipeline GCP: GitHub push → Cloud Build trigger → build & push to Artifact Registry → deploy to Cloud Run (or Cloud Deploy if time allows).

## Lab 5 (Day 5) — Observability

- Instrument the Lab 2/3 app: structured logs, custom metric (request count), trace on a slow endpoint.
- Dashboard + alarm in each cloud; alert to email/SMS.
- Inject failure: return 500s, scale down to zero, stop the container — confirm alerts fire.

## Lab 6 (Day 6) — Security + Cost

- KMS key + encrypt the storage bucket and DB in each cloud.
- Move any hardcoded secret into Secrets Manager / Secret Manager and mount at runtime.
- Pull the last day of CloudTrail / Cloud Audit Logs and find your own actions.
- Cost: build the running architecture in both pricing calculators, save estimates.

## Lab 7 (Day 7) — Capstone

Full stack, Terraform only:
- LB → 2+ container instances → managed DB (RDS vs Cloud SQL) → backups + monitoring alarm.
- README comparing: lines of Terraform, deploy time, estimated monthly cost, operational differences observed.
- `terraform destroy` both. Verify budget alerts are quiet.

---

## Free tier / cost guardrails

- GCP: e2-micro free (us-west1, us-central1, us-east1), 5GB GCS, Cloud Run 2M requests/mo, GKE gives $74.40 monthly credits toward control plane on one zonal/autopilot cluster (verify current terms).
- AWS: 750 hrs/mo t2.micro/t3.micro (first 12 months from account creation — check if yours has expired), 5GB S3, 1M Lambda requests.
- Budget alerts at $5 in both clouds on day 1. Check spend at the end of each day.
- EKS control plane costs $0.10/hr — if you must use EKS, create in the morning and destroy by evening (~$1), or stick to ECS Fargate.
