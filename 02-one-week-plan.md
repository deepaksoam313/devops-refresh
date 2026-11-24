# One-Week Intensive — AWS + GCP DevOps Refresher

Audience: 6 years DevOps experience. Goal: refresh both clouds hands-on in 7 days.
Strategy: mornings = reading/videos, afternoons = labs, everything via CLI/Terraform, both clouds each day.

## Ground rules

- Set a budget alert at $5 in BOTH clouds on day 1, before any lab.
- Prefer free tier: GCP e2-micro + Cloud Run + GKE free tier; AWS t2/t3.micro + Lambda + free EKS? (EKS is NOT free — use ECS Fargate for container labs instead, or a short-lived EKS spot cluster you delete same-day).
- Use one repo all week: `devops-refresh-labs`. Every lab done once per cloud, committed with screenshots/outputs.

## Day 1 — Accounts, IAM, networking

AM: IAM deep dive — AWS policies (explicit deny, conditions, resource vs identity policies), roles, instance profiles. GCP: roles, service accounts, Workload Identity, org/folder/project inheritance.
PM lab: build a 3-tier VPC per cloud with private subnets, NAT, and one bastion — CLI only (aws ec2 / gcloud compute). Peer the two VPCs is optional; instead set up private Google Access / VPC endpoints (AWS: S3 gateway endpoint; GCP: Private Google Access).

## Day 2 — Compute + storage

AM: EC2 vs GCE (spot/preemptible, launch templates vs instance templates, ASG vs MIG). S3 vs GCS (classes, lifecycle, encryption, signed URLs).
PM lab: static site on S3/GCS behind CloudFront/Cloud CDN; then a spot/preemptible batch VM that processes files and exits.

## Day 3 — Containers

AM: EKS + GKE architecture, node pools, autoscaling (Karpenter vs GKE Autopilot), Fargate vs Cloud Run.
PM lab: deploy the SAME containerized app (a tiny Go/Python HTTP service) three ways: ECS Fargate, GKE (Autopilot), Cloud Run. Compare deploy time, cold start, and cost per million requests.

## Day 4 — CI/CD + IaC

AM: Terraform as the common IaC layer (modules, state backends: S3+DynamoDB vs GCS). Native pipelines: CodePipeline vs Cloud Build.
PM lab: one GitHub repo, two pipelines (CodePipeline + Cloud Build) that build the container, push to ECR/Artifact Registry, and deploy to ECS/Cloud Run on merge. Write the whole infra in Terraform.

## Day 5 — Observability

AM: CloudWatch (metrics, logs insights, alarms, X-Ray) vs Cloud Monitoring/Logging/Trace. Structured logging, custom metrics, dashboards.
PM lab: instrument the Day-3 service in both clouds (logs, custom metric, trace), create a dashboard, set an alarm, then inject a failure (kill the app / return 500s) and watch the alert fire.

## Day 6 — Security + cost

AM: KMS/Secrets Manager vs Cloud KMS/Secret Manager; GuardDuty vs Security Command Center; audit trails (CloudTrail vs Cloud Audit Logs). Cost: pricing calculators, budgets, rightsizing (Compute Optimizer vs Recommender).
PM lab: add KMS encryption + secrets to the running services; pull a week's (or day's) audit log in both; produce a cost estimate of everything you built.

## Day 7 — Capstone + teardown

AM/PM: deploy the full stack (LB → containers → managed DB) entirely with Terraform, once per cloud. Document differences in a short README (see 04-labs.md). Then `terraform destroy` everything and confirm budgets show ~$0 remaining spend.

## Time budget

~8–10 hours/day. If you can only do 5–6 hours, drop Day 2 PM, Day 5 PM, and shrink Day 7 to a single-cloud capstone (do the other next week).

## After the week

- AWS DOP-C02 and/or GCP Professional Cloud DevOps Engineer as certification forcing functions.
