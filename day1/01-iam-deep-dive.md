# Day 1 — IAM Deep Dive: AWS vs GCP

---

## AWS IAM

### Core concepts

```
Account
└── IAM Identities
    ├── Users        (long-term credentials — avoid in production)
    ├── Groups       (collection of users, policies attached here)
    └── Roles        (assumed by services, apps, humans, cross-account)
```

### Policy types (know all 6)

| Type | Attached to | Use case |
|---|---|---|
| Identity-based | User / Group / Role | Grant permissions to a principal |
| Resource-based | S3 bucket, SQS queue, KMS key, etc. | Grant cross-account or cross-service access |
| Permission boundary | User / Role | Cap the max permissions an identity can ever have |
| SCP (Service Control Policy) | OU / Account in AWS Org | Org-wide guardrails — overrides everything within the account |
| Session policy | Temporary session (AssumeRole) | Further restrict what a role can do in this specific session |
| ACL (Access Control List) | S3, VPC | Legacy; prefer bucket policies |

### Policy evaluation logic (CRITICAL — exam + real life)

```
Evaluation order:
1. Explicit DENY in any policy   → DENY (always wins, no exceptions)
2. SCP allows?                   → if no, DENY
3. Resource-based policy allows? → ALLOW (if same account)
4. Identity-based policy allows? → ALLOW
5. Permission boundary allows?   → must also say ALLOW
6. Session policy allows?        → must also say ALLOW
7. Default                       → implicit DENY
```

Key takeaway: **explicit deny beats everything**. This is the most common IAM trap.

### Policy structure (JSON)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnly",
      "Effect": "Allow",
      "Principal": "*",            // only in resource-based policies
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

### Useful conditions

| Condition key | What it checks |
|---|---|
| `aws:RequestedRegion` | Region the API call targets |
| `aws:SourceVpc` / `aws:SourceVpce` | Call came from a specific VPC / endpoint |
| `aws:MultiFactorAuthPresent` | MFA was used to authenticate |
| `aws:PrincipalOrgID` | Caller is in your AWS Organization |
| `aws:CalledVia` | Service calling on behalf of principal (e.g. CloudFormation) |
| `s3:prefix` | Restrict access to specific S3 prefixes |
| `ec2:ResourceTag/Env` | Tag-based resource control |

### Roles in practice

```
# Assume a role (STS)
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session

# Check who you currently are
aws sts get-caller-identity

# Instance profile — attach role to EC2 at launch
# App calls IMDS, no static credentials needed
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/MyRole
```

### Cross-account access pattern

```
Account A (trusting): Role has trust policy allowing Account B
Account B (trusted):  User/Role calls sts:AssumeRole on Account A's role

Trust policy on Account A's role:
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::ACCOUNT-B-ID:role/DevRole" },
  "Action": "sts:AssumeRole"
}
```

### Common mistakes

- Using root account for anything after initial setup.
- Giving `*` actions or `*` resources in a policy (use least privilege).
- Forgetting both `s3:::bucket` AND `s3:::bucket/*` in S3 policies (one is for bucket-level ops, one for object-level).
- Using inline policies instead of managed policies — harder to audit and reuse.
- Forgetting that permission boundaries only constrain, they don't grant.

---

## GCP IAM

### Resource hierarchy (this is the big mental model shift)

```
Organisation
└── Folder (optional, can be nested)
    └── Project          ← everything lives in a project
        └── Resources (GCE, GCS, GKE, etc.)
```

- Policies are set at any level.
- **Inheritance flows DOWN** — a policy at the org level applies to all projects.
- **You cannot restrict inherited permissions** at a lower level (no deny by default).
- Best practice: grant at the lowest level that makes sense.

### Role types

| Type | Example | When to use |
|---|---|---|
| Primitive | `roles/owner`, `roles/editor`, `roles/viewer` | Never in production — too broad |
| Predefined | `roles/compute.instanceAdmin.v1`, `roles/storage.objectViewer` | Day-to-day assignments |
| Custom | Your own combination of permissions | When predefined is too broad |

### IAM binding structure

```
# Policy binding = member + role + (optional) condition
# "Who can do what on which resource"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:my-sa@PROJECT.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# View current policy
gcloud projects get-iam-policy PROJECT_ID --format=json
```

### Service accounts — 3 uses

```
1. As an identity for a workload (VM, GKE pod, Cloud Run service)
   → attach to resource so it authenticates without key files

2. As a resource to be impersonated
   → grant a human "roles/iam.serviceAccountTokenCreator"
   → they can generate short-lived tokens for that SA

3. As a key holder (service account key — JSON file)
   → AVOID this if at all possible
   → if you must: rotate keys, never commit to git, use Secrets Manager
```

### Workload Identity (GKE best practice)

```
# Replaces the old pattern of mounting SA key files into pods

# Concept: map a Kubernetes Service Account → GCP Service Account
# Pod gets GCP credentials automatically via metadata server

# Step 1: Enable Workload Identity on the cluster
gcloud container clusters create my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog

# Step 2: Create GCP SA
gcloud iam service-accounts create my-app-sa

# Step 3: Bind K8s SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"

# Step 4: Annotate the K8s SA
kubectl annotate serviceaccount KSA_NAME \
  iam.gke.io/gcp-service-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

### Conditions in GCP IAM

```json
{
  "condition": {
    "title": "Only during business hours",
    "expression": "request.time.getHours('America/New_York') >= 9 &&
                   request.time.getHours('America/New_York') <= 17"
  }
}
```

Common condition attributes: `resource.name`, `resource.type`, `request.time`, `resource.service`.

### Common mistakes

- Using primitive roles (`Editor`) in production.
- Downloading and committing service account JSON keys.
- Granting roles at Org/Folder level when Project level is enough.
- Forgetting that GCP IAM has no explicit deny — to restrict, you must not grant, or use a different project structure.
- Confusing service account impersonation with key-based auth (impersonation is preferred).

---

## AWS vs GCP IAM — Side-by-side summary

| Concept | AWS | GCP |
|---|---|---|
| Explicit deny | Yes — most powerful override | No (Deny policies are in preview only) |
| Multi-account structure | AWS Organizations + SCPs | Org → Folder → Project |
| Workload identity | Instance Profile (EC2), IRSA (EKS), Lambda execution role | Service Account attached to VM/Cloud Run; Workload Identity for GKE |
| Cross-service trust | Resource-based policies + IAM roles | SA impersonation, domain-wide delegation |
| Temporary credentials | STS AssumeRole | Short-lived tokens via SA impersonation / metadata server |
| Policy format | JSON (verbose) | Bindings (member + role + optional CEL condition) |
| Audit trail | CloudTrail | Cloud Audit Logs (Admin Activity always on; Data Access opt-in) |
