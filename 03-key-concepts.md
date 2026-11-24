# Key Concepts — AWS vs GCP Side-by-Side

This file grows as we discuss topics. Each section = one conceptual area.

---

## IAM

### AWS
- **Policy types**: identity-based (attached to user/role/group), resource-based (S3 bucket policy, trust policy), permission boundary, SCP (org level).
- **Explicit deny always wins** over any allow, regardless of source.
- **Roles** are assumed; users get credentials via AssumeRole (STS).
- **Instance profiles** attach a role to EC2 so the app calls IMDS without static keys.
- **Conditions**: `aws:RequestedRegion`, `aws:SourceVpc`, `aws:MultiFactorAuthPresent` etc.

### GCP
- **Resource hierarchy**: Organisation → Folder → Project → Resource. Policies bind at any level and are inherited down.
- **Role types**: Primitive (Owner/Editor/Viewer — avoid these), Predefined (e.g. `roles/compute.instanceAdmin`), Custom.
- **No explicit deny** (Deny policies exist in preview but are not the default model).
- **Service Accounts** are both identities (they have roles) and resources (you can grant humans the ability to impersonate them via `roles/iam.serviceAccountTokenCreator`).
- **Workload Identity**: preferred way for GKE pods to call GCP APIs — maps K8s service account → GCP service account without key files.

---

## Networking

### AWS
- VPC is **regional**; subnets are per AZ.
- Route tables attached per subnet.
- Internet Gateway (public), NAT Gateway (private outbound), VPC Endpoints (private AWS API access — Gateway type for S3/DynamoDB, Interface type for everything else).
- Peering is non-transitive; for hub-and-spoke use **Transit Gateway**.
- Security Groups are stateful; NACLs are stateless.
- **On-prem**: Direct Connect (dedicated) or Site-to-Site VPN.

### GCP
- VPC is **global**; subnets are per region (no AZ-level construct).
- One VPC, subnets in multiple regions — a VM in us-central1 can talk privately to a VM in europe-west1 in the same VPC.
- Cloud Router + BGP handles dynamic routing.
- **Private Google Access** lets VMs without external IPs reach Google APIs.
- Peering is non-transitive; for hub-and-spoke use **Network Connectivity Center** (or Shared VPC for same-org workloads).
- Firewall rules are global within the VPC, applied by network tag or service account.
- **On-prem**: Cloud Interconnect (dedicated/partner) or Cloud VPN.

---

## Compute

### AWS EC2
- Instance families: general (m/t), compute (c), memory (r/x), storage (i/d), GPU (p/g/inf).
- **Spot Instances**: up to 90% off; can be interrupted with 2-min notice. Use Spot Fleet or ASG with mixed policies.
- **Launch Templates** replace Launch Configurations.
- **Auto Scaling Group (ASG)**: target tracking, step scaling, scheduled.
- **AMI**: regional; must copy to other regions.

### GCP Compute Engine
- Machine families: general (E2/N2/N4), compute (C2/C3), memory (M2/M3), GPU (A2/G2).
- **Preemptible VMs** (older) or **Spot VMs** (new name): up to 91% off; 30-second shutdown notice, max 24h runtime.
- **Instance Templates** are global.
- **Managed Instance Group (MIG)**: autoscaling, autohealing, rolling updates, regional spread built in.
- **Custom machine types**: pick exact vCPU + RAM without being locked to predefined sizes.
- **Live migration**: GCE migrates VMs during host maintenance without rebooting (a major operational advantage over AWS).

---

## Containers

### AWS
- **ECS**: proprietary orchestrator. Task Definition (pod-like) + Service + Cluster. Launch type: EC2 or Fargate.
- **EKS**: managed control plane; you manage node groups. Expensive ($0.10/hr for control plane).
- **Fargate**: serverless container runtime; works with both ECS and EKS. No node management.
- **ECR**: managed container registry.
- **App Runner**: simplest path (source or image → running HTTPS service) — closest to Cloud Run.

### GCP
- **GKE Standard**: you manage node pools; most control.
- **GKE Autopilot**: fully managed nodes; Google manages scaling and security hardening; billed per pod resource not per node.
- **Cloud Run**: serverless containers, scale to zero, HTTP/gRPC, billed per request + CPU/memory ms.
- **Cloud Run Jobs**: for batch/non-HTTP workloads.
- **Artifact Registry**: stores container images, Helm charts, Maven/npm artifacts.

---

## CI/CD

### AWS
- **CodeBuild**: managed build server, runs buildspec.yml.
- **CodePipeline**: orchestrates stages (source → build → deploy); integrates with GitHub/Bitbucket.
- **CodeDeploy**: deploys to EC2, Lambda, ECS (blue/green). AppSpec file defines hooks.
- **CodeArtifact**: managed npm/Maven/PyPI repository.

### GCP
- **Cloud Build**: single YAML file (cloudbuild.yaml), steps are Docker containers. Fast, integrates with GitHub triggers natively.
- **Cloud Deploy**: managed delivery pipeline with promotion (dev → staging → prod) and approvals.
- **Artifact Registry**: images + packages.

### Cross-cloud reality
Most teams at this level use **GitHub Actions** or **GitLab CI** as the pipeline engine and use the cloud-native tools only for deployment targets. Worth knowing both.

---

## Observability

### AWS CloudWatch
- **Metrics**: default every 5 min (1 min with detailed monitoring, $). Custom metrics via PutMetricData.
- **Logs**: Log Groups → Log Streams. Log Insights for SQL-like queries.
- **Alarms**: up to 1-min resolution. Actions: SNS, Auto Scaling, EC2 stop/start.
- **X-Ray**: distributed tracing. Instrument via SDK; view service map and traces.
- **Container Insights**: enhanced metrics for ECS/EKS.

### GCP Cloud Operations (formerly Stackdriver)
- **Cloud Monitoring**: metrics from GCP resources + custom metrics via API or agent. Dashboards, alerting policies with notification channels (PagerDuty, Slack, etc.).
- **Cloud Logging**: structured logs (JSON), Log Sinks (export to GCS/BigQuery/Pub/Sub), Log-based metrics.
- **Cloud Trace**: distributed tracing, auto-collected from Cloud Run/GKE with OpenTelemetry.
- **Cloud Profiler**: continuous production profiling (CPU, memory) — no AWS equivalent without 3rd party.

---

## Security

### AWS
- **KMS**: symmetric/asymmetric keys, automatic annual rotation, used by most services for at-rest encryption.
- **Secrets Manager**: stores secrets with automatic rotation (Lambda backed), fine-grained access control.
- **Parameter Store**: lighter-weight (SSM), free tier for standard params.
- **GuardDuty**: threat detection (ML on CloudTrail/VPC Flow Logs/DNS). Enable per region.
- **CloudTrail**: API audit log across all services. Enable in all regions; set up S3 + CloudWatch Logs integration.
- **Security Hub**: aggregates findings from GuardDuty, Inspector, Macie etc.

### GCP
- **Cloud KMS**: symmetric/asymmetric/HMAC keys. CMEK (customer-managed encryption keys) for most services.
- **Secret Manager**: versioned secrets, IAM-controlled access, automatic replication.
- **Security Command Center (SCC)**: aggregated security findings, vulnerability scanning, misconfiguration detection.
- **Cloud Audit Logs**: Admin Activity (always on), Data Access (opt-in), System Events.
- **BeyondCorp / Identity-Aware Proxy (IAP)**: zero-trust access to apps/VMs without a VPN — no direct AWS equivalent.

---

## IaC

### Terraform (use this as your primary tool)
- Providers: `hashicorp/aws` and `hashicorp/google`.
- State backend: S3 + DynamoDB table (locking) for AWS; GCS bucket (native locking) for GCP.
- Workspaces or separate state files for dev/staging/prod.
- Modules: use the official registry modules as a starting point, then customise.

### AWS native
- **CloudFormation**: JSON/YAML templates, stacks, stack sets (multi-account/region). Slow. Declarative.
- **CDK**: generate CF via code (TypeScript, Python, Java etc.). Better for complex apps.

### GCP native
- **Deployment Manager**: YAML + Jinja/Python. Legacy; not recommended for new projects.
- **Infrastructure Manager** (Config Connector / Terraform-based): newer managed Terraform runner.

---
*Last updated: session 2*
