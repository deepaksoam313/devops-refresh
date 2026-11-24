# Session Log — DevOps Refresh Project

This file is updated after every session. If you are a new Kiro session, read this file first before doing anything else. It tells you exactly what has been done, what is in progress, and what comes next.

---

## About this project

**User profile**: 6 years of DevOps experience on AWS and GCP.
**Goal**: Intensive one-week refresh of DevOps-relevant cloud services on both AWS and GCP.
**Start date**: Session began November 2024.
**Working directory**: `/home/ubuntu/devops-refresh/`

---

## Folder structure

```
/home/ubuntu/devops-refresh/
├── SESSION-LOG.md                   ← this file, read first on every new session
├── 00-INDEX.md                      ← project overview and file map
├── 01-aws-gcp-service-comparison.md ← full service mapping AWS vs GCP
├── 02-one-week-plan.md              ← 7-day intensive schedule
├── 03-key-concepts.md               ← running reference doc (grows each day)
├── 04-labs.md                       ← lab instructions per day
└── day1/
    ├── 00-day1-index.md             ← day 1 checklist and schedule
    ├── 01-iam-deep-dive.md          ← IAM deep dive (AWS + GCP)
    └── 02-networking-deep-dive.md   ← Networking deep dive + lab CLI commands
```

---

## Session history

### Session 1
**What we did**:
- User introduced themselves: 6 years DevOps, AWS + GCP experience, wants a refresh.
- Agreed on a 1-week intensive plan (compressed from an earlier 6-week suggestion).
- Created the full project folder structure under `/home/ubuntu/devops-refresh/`.
- Documented the AWS vs GCP service comparison (01-aws-gcp-service-comparison.md).
- Wrote the 7-day plan (02-one-week-plan.md).
- Wrote detailed key concepts reference (03-key-concepts.md) covering IAM, networking, compute, containers, CI/CD, observability, security, IaC.
- Wrote lab instructions (04-labs.md) with free-tier/cost guardrails.

**Files created this session**:
- `00-INDEX.md`
- `01-aws-gcp-service-comparison.md`
- `02-one-week-plan.md`
- `03-key-concepts.md`
- `04-labs.md`

---

### Session 2
**What we did**:
- User asked to start Day 1.
- Created `day1/` subfolder with full Day 1 content.
- Wrote `day1/01-iam-deep-dive.md` covering:
  - AWS: all 6 policy types, evaluation logic (explicit deny always wins), policy JSON structure, useful conditions, cross-account role assumption, common mistakes.
  - GCP: resource hierarchy (org→folder→project), role types (primitive/predefined/custom), service accounts (3 modes), Workload Identity for GKE, IAM conditions with CEL.
  - Side-by-side comparison table.
- Wrote `day1/02-networking-deep-dive.md` covering:
  - AWS: VPC (regional, subnets per AZ), security groups vs NACLs, NAT Gateway, VPC Endpoints (gateway vs interface), Transit Gateway, ALB/NLB/GWLB, Route 53 routing policies.
  - GCP: global VPC (biggest mental model shift), firewall rules by tag/SA, Cloud NAT + Cloud Router, Shared VPC, Private Google Access, single global LB (L4+L7), Cloud DNS.
  - Full CLI lab: build 3-tier VPC (public subnet + private subnet + NAT + bastion + private app VM) in BOTH clouds, verify, cleanup commands.
- Created `day1/00-day1-index.md` with checklist.
- User asked to document sessions so new Kiro sessions can resume easily → created this SESSION-LOG.md.

**Files created this session**:
- `day1/00-day1-index.md`
- `day1/01-iam-deep-dive.md`
- `day1/02-networking-deep-dive.md`
- `SESSION-LOG.md` (this file)

---

### Session 3
**What we did**:
- User asked to document session activity so new Kiro sessions can resume without re-explanation.
- Created `SESSION-LOG.md` with full project context, session history, folder structure, current status, next steps, conventions, and key preferences.
- Updated SESSION-LOG.md with this session entry (Session 3).

**Files created this session**:
- None (SESSION-LOG.md was created at end of Session 2, updated here)

**Files updated this session**:
- `SESSION-LOG.md` — added Session 3 entry, updated current status and next steps

### Session 4
**What we did**:
- New session resumed from SESSION-LOG.md as designed.
- User asked for Day 1's IAM topic to be explained in detail (the compact deep-dive was too fast) and to document the discussion.
- Created `day1/01a-iam-explained-in-detail.md`: plain-language, story-driven walkthrough of the whole IAM topic — WHO/WHAT/WHICH model, users vs groups vs roles, all 6 AWS policy types with analogies (boundary = ceiling, SCP = envelope), a step-by-step policy evaluation walkthrough with a worked PutObject example, the S3 bucket vs /* resource gotcha, conditions with real JSON, cross-account "both sides must agree", GCP hierarchy (no deny, no subtracting), role types, the 3 service account modes, Workload Identity step by step, and fully explained answers to the 3 self-test questions.
- User reported IAM still too complex → created `day1/01b-iam-simple-explained.md`: minimal beginner version built on a building-security analogy (badge / door note / ceiling / blocked floor / banned list) plus the two GCP rules (grants flow down and can't be subtracted; service accounts attach or impersonate, never keys). Reading order advice: 01b first, 01a on second pass.
- User is now reading `01b-iam-simple-explained.md`.
- Next step when user returns: have them explain the building-security analogy back (badge / door note / ceiling / blocked floor / banned list) to confirm the mental model, then drill any stuck part, then move to the networking module (02).

**Files created this session**:
- `day1/01a-iam-explained-in-detail.md`
- `day1/01b-iam-simple-explained.md`

**Files updated this session**:
- `day1/00-day1-index.md` (file table)
- `SESSION-LOG.md` (this entry)

---

## Current status

**Day**: Day 1 in progress — IAM module.
**User is on**: Reading `day1/01b-iam-simple-explained.md` (simplified IAM). Found full detail too complex; simple version delivered and documented.
**Days remaining**: Day 2 through Day 7 content not yet created.
**Last action**: Updated session log while user reads 01b. Next: user explains the building analogy back, drill stuck parts, then move to networking (02).

---

## What to do next (for the next session)

1. **Read this file first** before responding to the user.
2. User was mid-read on `day1/01b-iam-simple-explained.md`. First ask: "Did the simple version click? Try explaining the building analogy back." (badge / door note / ceiling / blocked floor / banned list + the two GCP rules). Drill whatever is stuck.
3. When IAM clicks, move to networking: walk through `day1/02-networking-deep-dive.md` (offer the same simplified treatment if needed — e.g. a hotel/office-network analogy for VPC, subnets, NAT, firewall layers).
4. Then the Day 1 lab (3-tier VPC in both clouds). Budget alerts must be set before lab.
5. After Day 1 is done, build Day 2 content: `day2/` folder, Compute deep dive (EC2 vs GCE, spot/preemptible, ASGs vs MIGs) + Storage (S3 vs GCS, EBS vs PD, EFS vs Filestore). Lab: static site on S3/GCS behind CDN + spot/preemptible batch VM.
6. Keep updating this SESSION-LOG.md at the end of every session or when user says "update the log".

---

## Conventions used in this project

- Every topic is covered for BOTH AWS and GCP side by side.
- Labs always run in both clouds the same afternoon.
- All CLI commands are included in the docs — no console/GUI.
- Terraform is the IaC layer used on Day 4 and Day 7.
- Budget guardrail: $5 alert in both clouds, check spend at end of each day.
- EKS is avoided (costs $0.10/hr) — use ECS Fargate or GKE Autopilot instead.
- Each day gets its own subfolder (`day1/`, `day2/`, etc.) with an index + topic files.

---

## Key preferences / things Kiro should know

- User has strong existing knowledge — go deep, don't oversimplify.
- User wants everything documented in this folder as we go.
- User wants CLI-first approach (not console).
- Always update SESSION-LOG.md at the end of each session or when asked.
- If the user says "document this" or "add this", update the relevant file in `/home/ubuntu/devops-refresh/`.
