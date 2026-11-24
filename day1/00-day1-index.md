# Day 1 — Index

**Theme**: IAM + Networking (the foundation everything else sits on)

**Files**:
| File | Contents |
|---|---|
| 01-iam-deep-dive.md | AWS IAM policies, evaluation logic, roles, GCP roles/SA/Workload Identity |
| 01a-iam-explained-in-detail.md | Same IAM content in plain language with worked examples and answers |
| 01b-iam-simple-explained.md | Beginner version — building-security analogy, minimal, read this first |
| 02-networking-deep-dive.md | AWS VPC vs GCP VPC, security, NAT, peering, LBs, DNS + Lab CLI commands |

**Morning** (3–4 hrs): read both files, focus on the side-by-side tables and gotchas.

**Afternoon** (4–5 hrs): run Lab 1 from 02-networking-deep-dive.md — build the 3-tier VPC in both clouds CLI only, then destroy it.

**Day 1 Checklist**:
- [ ] Set $5 budget alert in AWS
- [ ] Set $5 budget alert in GCP
- [ ] Can explain all 6 AWS policy types from memory
- [ ] Can explain GCP resource hierarchy and policy inheritance
- [ ] Built 3-tier VPC in AWS via CLI
- [ ] Built 3-tier VPC in GCP via CLI
- [ ] Private VM in each cloud can reach internet via NAT but has no public IP
- [ ] Cleaned up all resources
