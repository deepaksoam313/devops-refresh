# Day 1 — IAM Explained in Detail (Plain Language)

This file is a slow, detailed walkthrough of `01-iam-deep-dive.md`.
If the tables and compact notes felt too fast, read this instead.
Everything is explained with concrete stories and worked examples.

---

# Part 1 — How to think about IAM at all

Every permission question in any cloud comes down to three things:

```
WHO     is trying to do something      (the principal)
WHAT    action are they trying to do  (the verb: read, write, delete...)
WHICH   thing are they doing it to     (the resource)
```

- In **AWS**: WHO = IAM user or role. WHAT = an action like `s3:GetObject`.
  WHICH = a resource like `arn:aws:s3:::my-bucket/file.txt`.
- In **GCP**: WHO = a member (user, group, or service account).
  WHAT+WHICH = a role, like `roles/storage.objectViewer`, granted on a resource.

The entire job of IAM is: when the API receives a request, decide
ALLOW or DENY. Both clouds do this decision at request time, every
single time, by evaluating all the policies that apply.

---

# Part 2 — AWS IAM, explained piece by piece

## 2.1 Users, Groups, Roles

**User** — a permanent identity with long-lived credentials
(access key + secret key). Meant for humans. In a well-run
production account you barely use these: humans should federate
in (SSO / Google / Okta) and *assume roles*.

**Group** — just a container. You attach policies to the group,
every user in it inherits them. Users can be in many groups.
Groups exist only to make policy management easier.

**Role** — an identity that *has no permanent credentials*.
Something "becomes" the role for a limited time and gets
temporary credentials. Roles are how:
- an EC2 instance gets permissions,
- a Lambda function gets permissions,
- a CI/CD pipeline gets permissions,
- a human temporarily switches to admin,
- account A lets account B do something.

The mental picture for a role:

```
"Nobody logs in AS a role.
 You BECOME the role temporarily, you get a temporary
 access key that expires, and everything you do during
 that session is attributed to the role."
```

Why this is better than users: no permanent secret to leak, and
every action is logged under a session name you can trace.

## 2.2 The 6 policy types, one by one

The confusing part is that AWS has *multiple layers* of policies
that all get checked. Here is each one in plain language.

### 1. Identity-based policy

The "normal" policy. You write it and attach it to a user,
group, or role. It says "whoever holds me may do X".

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

Read it out loud: "Allow the action s3:GetObject on objects
inside my-bucket." Whoever this is attached to can read objects
in that bucket (and nothing else).

### 2. Resource-based policy

This one lives ON the thing being accessed, not on the caller.
The most common is an **S3 bucket policy**.

The key difference: an identity policy says "I am allowed to".
A resource policy says "you are allowed to, on this thing".
For cross-account access (another account's user reading your
bucket), the resource policy is the only way.

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::999999999999:root" },
  "Action": ["s3:GetObject"],
  "Resource": ["arn:aws:s3:::my-bucket/*"]
}
```

Read out loud: "Account 999999999999 is allowed to read objects
in this bucket." Note the `Principal` — that is the extra
element identity policies don't have. It names WHO is being
allowed.

### 3. Permission boundary

Think of it as a **ceiling**, not a grant.

Analogy: you give your kid a $50 weekly allowance (the identity
policy), but you also set a parental control that no single
purchase may exceed $20 (the boundary). The allowance and the
cap BOTH have to be satisfied. A $15 purchase works. A $30
purchase fails — even though the total allowance would cover it,
because the cap says no.

So: for an action to succeed, the identity policy must allow it
AND the boundary must also allow it. The boundary alone grants
nothing.

Real use case: a contractor gets `PowerUserAccess` (the identity
policy — broad) but you set a boundary that excludes IAM and
organizations actions, so they can never escalate themselves or
peek at the account structure.

### 4. SCP (Service Control Policy) — AWS Organizations

A guardrail applied at the Organization / OU / account level.
It does NOT grant anything. It defines the *maximum envelope*
of what is even possible inside the account.

Analogy: the identity policy is a driver's license; the SCP is
the set of roads that exist. Having a license (identity policy)
that lets you drive at 200 km/h means nothing if no road exists
(SCP blocks everything outside an allow list).

Two flavors:
- **Allow list**: "only EC2 and S3 are even possible in this
  account." Everything else — Lambda, DynamoDB, IAM — is dead
  even for the root admin of that account.
- **Deny list**: "everything is possible EXCEPT leaving these
  regions / except these dangerous actions."

Common use: an OU for sandbox accounts with a deny that blocks
`iam:CreateUser`, or an org-wide deny on regions outside
us-east-1 and eu-west-1.

### 5. Session policy

When you assume a role you can pass an extra inline policy into
the session. The session is then limited by BOTH the role's
normal policies and this session policy. Used for "let me
escalate to admin but only for this one task, at reduced
scope" — e.g. emergency break-glass with a session policy that
restricts the session to one specific bucket.

### 6. ACL

Legacy, from the early S3 era. Buckets/objects had ACLs. Modern
AWS guidance: do not use ACLs; use bucket policies. Listed only
so you recognize it and know to ignore it.

## 2.3 Policy evaluation — a worked example

The evaluation, step by step. AWS checks in this order:

```
Step 0: Is there an explicit DENY anywhere that matches
        this action+resource?
        YES → DENY. Stop. Nothing can override this.
        NO  → continue.

Step 1: Do the applicable SCPs allow this action
        (i.e. is the action inside the SCP envelope)?
        NO  → DENY.
        YES → continue.

Step 2: Does a resource-based policy on the target
        allow this principal? (relevant mainly
        cross-account, or same-account resource grants)
        YES → ALLOW. Done.
        NO  → continue to step 3.

Step 3: Does an identity-based policy allow it?
        YES → tentatively ALLOW, but keep going.

Step 4: Is there a permission boundary, and does it
        also allow it?
        NO  → DENY.
        YES → continue.

Step 5: Is there a session policy, and does it also
        allow it?
        NO  → DENY.
        YES → ALLOW.

Fell all the way through with no allow → implicit DENY
(default: everything is denied unless allowed).
```

The two logics to burn into memory:

- Steps 2 and 3 are **OR**: either a resource policy or an
  identity policy allowing it is enough (same account).
- Steps 4 and 5 are **AND**: boundary and session policy are
  extra conditions stacked on top. They can only *take away*.

### Worked example

Setup:
- Role `DeployRole` has an identity policy allowing `s3:*` on
  bucket `app-data` (full S3 powers on that bucket).
- The same role has a permission boundary allowing only
  `s3:GetObject` on `arn:aws:s3:::app-data/*`.
- No session policy. No deny anywhere. No SCP.

Request: `s3:PutObject` on `app-data/report.txt`.

Walk it:
```
Step 0: no explicit deny → continue
Step 1: no SCP → continue
Step 3: identity policy allows s3:* → tentatively allow
Step 4: boundary — does it allow s3:PutObject?
        It only allows s3:GetObject. → NO.
Result: DENIED.
```

The identity policy said yes, the boundary said no, and the
boundary always wins because it is an AND condition.
`Get report.txt` would work (both allow it).

## 2.4 The S3 two-resource gotcha

S3 actions split into two families that need DIFFERENT resource
ARNs:

| Action | Level | Required Resource ARN |
|---|---|---|
| `s3:ListBucket` | bucket | `arn:aws:s3:::my-bucket` |
| `s3:GetObject` | object | `arn:aws:s3:::my-bucket/*` |
| `s3:PutObject` | object | `arn:aws:s3:::my-bucket/*` |
| `s3:DeleteObject` | object | `arn:aws:s3:::my-bucket/*` |

Story: you write a policy with `s3:GetObject` and Resource
`arn:aws:s3:::my-bucket` (no `/*`). The app calls
`aws s3 cp s3://my-bucket/file.txt .` → AccessDenied. You
stare at the action, which is correct. The problem is the
resource: GetObject is an OBJECT action, and objects live
under `my-bucket/*`. A read-only S3 policy needs both lines:

```json
"Resource": [
  "arn:aws:s3:::my-bucket",
  "arn:aws:s3:::my-bucket/*"
]
```

## 2.5 Conditions — with real examples

A `Condition` block adds "only if" logic to a statement.

**Require MFA for dangerous stuff:**
```json
"Condition": { "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" } }
```
Read: "if MFA was NOT present, this (Deny) applies" —
i.e. without MFA, denied.

**Lock a role to one region:**
```json
"Condition": { "StringEquals": { "aws:RequestedRegion": "us-east-1" } }
```

**Only from our VPC / VPC endpoint (bucket policy):**
```json
"Condition": { "StringEquals": { "aws:SourceVpce": "vpce-12345" } }
```

**Tag-based control:**
```json
"Condition": { "StringEquals": { "ec2:ResourceTag/Env": "prod" } }
```
Read: this statement only matches resources tagged Env=prod.
Pair it with a deny for destructive actions on Env=prod and
only the prod-change role survives.

## 2.6 Roles in practice — the commands you actually use

```bash
# Who am I right now? (first command to run when perms seem wrong)
aws sts get-caller-identity
# Returns: the ARN of the identity making the calls.
# If this surprises you, you found your bug.

# Assume a role
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name debugging-session
# Returns temporary keys (AccessKeyId, SecretAccessKey,
# SessionToken) valid 1h by default, 12h max.
```

For an EC2 instance you never call assume-role in your code.
You attach an **instance profile** at launch, and the AWS
tools on the box automatically fetch short-lived credentials
from the metadata service at 169.254.169.254. Your application
code contains zero credentials.

## 2.7 Cross-account access — both sides must agree

For account B's user to read account A's bucket:

```
Side A (the bucket owner) must ALLOW it:
  → bucket policy with Principal = account B
  (or a role in A with a trust policy naming B)

Side B (the caller) must be ALLOWED to make the call:
  → identity policy in B granting s3:GetObject on A's bucket
  (or sts:AssumeRole on A's role)
```

If only side A allows it: B's request is denied because B's own
identity policy doesn't permit the action.
If only side B allows it: A never said yes, so denied.
Both must say yes. People constantly forget one side.

---

# Part 3 — GCP IAM, explained piece by piece

## 3.1 The resource hierarchy

```
Organisation           (represents your company; created once)
└── Folder             (optional; e.g. "engineering", "prod")
    └── Project        (the unit of: billing, APIs, quotas, deletion)
        └── Resources  (VMs, buckets, clusters, Cloud Run services)
```

Projects are the hard boundary. Delete a project and everything
in it goes (after a 30-day recovery window). APIs are enabled
per project. Quotas are per project.

**Inheritance:** an IAM binding granted at any level applies to
everything BELOW it. Grant `roles/viewer` at the org → every
project, every resource. Grant at a folder → everything in that
folder's projects.

**The rule with no exception:** you cannot subtract at a lower
level. If someone is Editor at the folder, you cannot say "but
not on this project." GCP IAM (GA) has NO deny. Protection is
by structure:

- Want isolation? Put it in its own project and control grants
  there.
- Grant at the LOWEST level that works. Org-level grants are
  for org admins only, folder-level for folder admins only.

(A Deny feature exists in preview/Admin Org policy, but design
as if deny does not exist.)

## 3.2 Roles

A GCP role is a *named bag of permissions*. A binding is
`member + role + resource` (+ optional condition).

- **Primitive roles** — `roles/viewer`, `roles/editor`,
  `roles/owner`. They cover EVERY product in the project.
  `editor` can edit almost anything. Fine for a throwaway
  sandbox, never for prod.
- **Predefined roles** — Google ships ~hundreds, product by
  product: `roles/storage.objectViewer` (read objects only),
  `roles/compute.instanceAdmin.v1`, `roles/secretmanager.viewer`.
  This is your default choice.
- **Custom roles** — you pick individual permissions when even
  predefined is too broad. More maintenance (Google updates
  predefined roles; yours stay frozen), so use sparingly.

```bash
# Grant
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:jane@example.com" \
  --role="roles/storage.objectViewer"

# Read the full policy
gcloud projects get-iam-policy PROJECT_ID --format=json

# Find roles available for a product
gcloud iam roles list --filter="name:storage"
```

## 3.3 Service accounts — the three modes, in depth

A service account is an identity for non-humans. Three ways it
gets used:

### Mode 1 — the workload runs AS the SA (the right default)

```
VM:   created with --service-account=sa@proj.iam.gserviceaccount.com
Cloud Run / GKE / Cloud Functions: same idea
```

The workload's Google API calls are automatically authenticated
as that SA; tokens come from the metadata server on the box.
No key file exists at all. Grant the SA the role, not the
human.

Example: a VM needs to read GCS:
1. Create `vm-reader@proj.iam.gserviceaccount.com`
2. Grant it `roles/storage.objectViewer` on the bucket (or
   project)
3. Launch the VM with `--service-account=vm-reader@...`
4. On the VM, `gcloud storage ls` just works.

A human with zero storage roles can still deploy such a VM
*if* they have `roles/iam.serviceAccountUser` on the SA.
Separation: the human can build the runner but not read the
data.

### Mode 2 — impersonation

Instead of a permanent key, you let a human or service BECOME
the SA for an hour:

```bash
# Grant once:
gcloud iam service-accounts add-iam-policy-binding \
  sa@proj.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountTokenCreator \
  --member="user:jane@example.com"

# Jane then runs things AS the SA (short-lived token, 1h):
gcloud auth activate-service-account --impersonate-service-account \
  sa@proj.iam.gserviceaccount.com
```

Use case: emergency admin, CI jobs needing elevated rights for
one run, local dev against a real SA. Everything is auditable
and nothing permanent leaks.

### Mode 3 — the JSON key file (avoid)

`gcloud iam service-keys create` downloads a JSON credential
that NEVER EXPIRES. Equivalent to a password with no rotation
forced. If you must: store it in Secret Manager, rotate
regularly, and understand that anyone with the file IS the SA.
Most "GCP key leaked in a GitHub repo" incidents come from this
mode. Modern GCP lets you disable key creation entirely — do
that org-wide.

## 3.4 Workload Identity (GKE) — step by step

Problem: a pod needs GCP credentials. The 2019 answer was
mounting a key file into the pod (Mode 3, bad). The current
answer: a mapping between a Kubernetes SA and a GCP SA.

```
Pod (uses KSA "frontend")
 └── KSA annotated → maps to GCP SA "frontend-gcs@proj"
      └── Pod's metadata server transparently returns
          tokens for the GCP SA
```

Full setup:

```bash
# 1. Cluster must have Workload Identity enabled
gcloud container clusters create my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog

# 2. Create the GCP SA
gcloud iam service-accounts create frontend-gcs

# 3. Let the K8s SA impersonate the GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  frontend-gcs@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[default/frontend]"

# 4. Tell the K8s SA which GCP SA it maps to
kubectl annotate serviceaccount frontend \
  iam.gke.io/gcp-service-account=frontend-gcs@PROJECT_ID.iam.gserviceaccount.com
```

AWS twin: IRSA. Same idea — pod gets scoped, temporary
credentials, no static keys.

## 3.5 Conditions in GCP

Bindings can carry a CEL expression:

```json
"condition": {
  "title": "Business hours only",
  "expression": "request.time.getHours('America/New_York') >= 9 && request.time.getHours('America/New_York') <= 17"
}
```

Useful for time-boxed elevated grants (on-call access, PIM-style
just-in-time).

## 3.6 Audit

Cloud Audit Logs split into:
- **Admin Activity** — every control-plane change (who created
  what). ALWAYS on, free.
- **Data Access** — who read/wrote data. Opt-in, has cost.

IAM-related rule of thumb: any binding change shows up in Admin
Activity — no excuse not to check who granted what.

---

# Part 4 — Answers to the three self-test questions

**Q1. Role with identity policy `s3:*` and a boundary allowing
only `s3:GetObject`. It tries `s3:PutObject`. What happens?**

DENIED. The identity policy allows it, but the boundary is an
AND condition and it only covers GetObject. A permission
boundary can only shrink, never expand. GetObject would
succeed; every other S3 action on that bucket fails.

**Q2. SCP at the OU allows only `ec2:*` and `s3:*`. An admin in
that account tries to create a Lambda function. What happens?**

DENIED, and it doesn't matter how powerful the admin's identity
policy is — even the account root. The SCP defines the envelope
of what is possible in the account; Lambda is outside the
envelope, so the request fails at step 1 of evaluation. To make
it work, the SCP itself must change.

**Q3. GCP: user has `roles/editor` at the folder. You want to
block them from ONE project under it. What do you do?**

You cannot block them — GCP has no deny and inheritance can't
be subtracted. Your options:
- Get the grant moved from the folder down to only the projects
  where the user actually needs it (the correct fix), or
- Move the sensitive project OUT from under that folder, or
- Accept it and change the role to something scoped properly.
The lesson: in GCP you protect things with structure
(projects/folders) and careful grants, never with denies.

---

# Part 5 — The single-page mental summary

```
AWS
- 6 policy types; identity + resource are the two you write daily
- Explicit deny beats everything, always
- SCP = envelope (grants nothing), boundary = ceiling, both AND
- S3: bucket ARN for ListBucket, /* for object actions
- Roles + temporary credentials > users + permanent keys
- Cross-account needs BOTH sides to allow

GCP
- Org → Folder → Project; everything lives in a project
- Inheritance flows down and cannot be subtracted (no deny)
- Prefer predefined roles; never primitive roles in prod
- Service accounts: attach to workload (good), impersonate (good),
  key files (avoid)
- Workload Identity for GKE = no keys in pods
- Admin Activity logs are always on — check who granted what
```
