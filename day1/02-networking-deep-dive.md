# Day 1 — Networking Deep Dive: AWS vs GCP

---

## AWS Networking

### VPC fundamentals

```
Region
└── VPC (CIDR: e.g. 10.0.0.0/16)
    ├── Availability Zone A
    │   ├── Public Subnet  (10.0.1.0/24)
    │   └── Private Subnet (10.0.2.0/24)
    └── Availability Zone B
        ├── Public Subnet  (10.0.3.0/24)
        └── Private Subnet (10.0.4.0/24)
```

Key components:
- **Internet Gateway (IGW)**: attached to VPC, enables internet for public subnets.
- **Route table**: each subnet associates one route table. Public subnet routes 0.0.0.0/0 → IGW.
- **NAT Gateway**: in public subnet, allows private subnet VMs to reach internet (not inbound). Regional, not AZ-resilient — deploy one per AZ for HA.
- **Elastic IP**: static public IP, attached to NAT GW or EC2.

### Security layers

```
Network ACL (NACL)                  Security Group
─────────────────────────           ─────────────────────────
Applied at subnet level             Applied at ENI (instance) level
Stateless — must allow in AND out   Stateful — allow in = reply auto-allowed
Rules evaluated in number order     All rules evaluated
Has explicit allow AND deny         Allow only (implicit deny)
```

Rule of thumb: Security Groups for instance-level control (you use these daily), NACLs for subnet-level guardrails (rarely touched, mostly for blocking a specific IP range).

### Connectivity options

```
Within the same VPC:          → route tables + security groups
VPC ↔ VPC (same account):    → VPC Peering (non-transitive) or Transit Gateway
VPC ↔ VPC (cross-account):   → VPC Peering or Transit Gateway + RAM (Resource Access Manager)
VPC ↔ On-premises:           → Direct Connect (private, dedicated 1/10/100 Gbps)
                                or Site-to-Site VPN (over internet, encrypted)
VPC ↔ AWS services:          → VPC Endpoints (private path, no internet)
                                Gateway type: S3, DynamoDB (free, route table based)
                                Interface type: everything else (PrivateLink, $0.01/hr + data)
```

### Transit Gateway — hub and spoke

```
           VPC-A
            |
VPC-B ── Transit GW ── VPC-C
            |
        On-premises (VPN/DX)
```

- Transitive routing — VPC-A can reach VPC-C through TGW.
- One TGW per region (can peer TGWs across regions).
- Attach VPCs, VPNs, Direct Connect gateways.
- Route tables on TGW control which attachments can talk to each other.

### VPC Endpoints (key for private architectures)

```bash
# Gateway endpoint — S3 (free, modify route table automatically)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-xyz789 \
  --vpc-endpoint-type Gateway

# Interface endpoint — SSM (private IP in your subnet)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-abc \
  --security-group-ids sg-xyz
```

Interface endpoints = ENI in your subnet with a private DNS name. App calls the same `ssm.us-east-1.amazonaws.com` endpoint but DNS resolves to private IP.

### Load balancers

| Type | Layer | Use case |
|---|---|---|
| ALB (Application LB) | L7 HTTP/HTTPS | Web apps, microservices, path/host-based routing, WAF |
| NLB (Network LB) | L4 TCP/UDP/TLS | High performance, static IP, non-HTTP protocols |
| GWLB (Gateway LB) | L3 | Insert 3rd party appliances (firewall, IDS) transparently |
| CLB (Classic LB) | L4/L7 | Legacy, don't use |

ALB features worth knowing:
- Path-based routing: `/api/*` → service A, `/web/*` → service B.
- Host-based routing: `api.example.com` → target group A.
- Weighted routing (canary deployments).
- Lambda as a target.
- Sticky sessions (cookie-based).
- Native integration with Cognito for auth.

### DNS — Route 53

```
Record types:
  A      → IPv4
  AAAA   → IPv6
  CNAME  → another hostname (not for zone apex)
  ALIAS  → AWS-specific, points to AWS resources, works at apex

Routing policies:
  Simple         → single record
  Weighted       → split traffic by %
  Latency        → route to lowest latency region
  Failover       → primary + secondary (health check required)
  Geolocation    → based on user's country/continent
  Geoproximity   → based on distance, with bias
  Multi-value    → up to 8 healthy IPs (not a replacement for LB)

Private hosted zone: DNS only within your VPC(s)
```

---

## GCP Networking

### VPC fundamentals — the global difference

```
VPC (GLOBAL — not regional)
├── Subnet: us-central1 (10.0.0.0/20)
├── Subnet: europe-west1 (10.1.0.0/20)
└── Subnet: asia-east1 (10.2.0.0/20)

A VM in us-central1 and a VM in europe-west1
in the SAME VPC communicate over Google's internal
network — no public IP needed, no peering required.
```

This is the biggest mental model shift from AWS. In GCP:
- VPC spans all regions by default.
- Subnets are regional.
- Routes and firewall rules are global within the VPC.

### Firewall rules

GCP firewalls are **stateful** (like AWS SGs) but applied at the **VPC level**, filtered by:
- **Network tags**: attach a tag `web-server` to VMs, write rule targeting that tag.
- **Service account**: rule applies to VMs running as a specific SA.
- **IP range**: CIDR-based.

```bash
# Allow HTTP to VMs tagged "web-server" from anywhere
gcloud compute firewall-rules create allow-http \
  --network=my-vpc \
  --allow=tcp:80 \
  --target-tags=web-server \
  --source-ranges=0.0.0.0/0

# Allow internal traffic between app and db VMs
gcloud compute firewall-rules create allow-app-to-db \
  --network=my-vpc \
  --allow=tcp:5432 \
  --target-tags=db \
  --source-tags=app
```

Default firewall: GCP creates default-allow-internal and default-allow-ssh for the default VPC. In production, delete the default VPC and build your own.

### NAT — Cloud NAT

```bash
# Cloud NAT needs a Cloud Router (BGP route reflector, also used for VPN/Interconnect)
# Step 1: create router
gcloud compute routers create my-router \
  --region=us-central1 \
  --network=my-vpc

# Step 2: create NAT config on the router
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### Connectivity options

```
Within same VPC:              → routes (automatic) + firewall rules
VPC ↔ VPC (same org):        → Shared VPC (host project shares subnets to service projects)
                                or VPC Peering (non-transitive, like AWS)
VPC ↔ VPC (cross-org):       → VPC Peering
VPC ↔ On-premises:           → Cloud Interconnect (dedicated: 10/100 Gbps; partner: 50Mbps–10Gbps)
                                or Cloud VPN (HA VPN: 2 tunnels, 3 Gbps each)
VPC ↔ Google APIs:           → Private Google Access (enable on subnet, no extra resource)
                                or Private Service Connect (PrivateLink equivalent)

Multi-VPC hub-and-spoke:     → Network Connectivity Center (NCC) — newer than TGW, same concept
```

### Shared VPC — the enterprise pattern

```
Organisation
├── Host Project (owns the VPC and subnets)
└── Service Projects (compute resources live here)
    └── VMs, GKE clusters, Cloud Run — all use subnets from Host Project

Benefits:
- Centralized network control (firewall, subnets managed in one place)
- Service project teams can't modify network
- Billing stays in service projects
```

AWS analogue: Resource Access Manager (RAM) sharing a subnet, but GCP's model is cleaner.

### Private Google Access

```bash
# No extra resource needed — just enable on the subnet
# Allows VMs with no external IP to reach Google APIs (GCS, BigQuery, etc.)
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access

# Verify
gcloud compute networks subnets describe my-subnet \
  --region=us-central1 \
  --format="get(privateIpGoogleAccess)"
```

### Load balancers — GCP model

GCP has ONE external load balancer product that does everything:

```
External Application LB (Global, L7) → closest to AWS ALB
  - anycast IP, routes to nearest healthy backend
  - HTTP(S), HTTP/2, gRPC
  - Cloud CDN and Cloud Armor (WAF) integrate here

External Passthrough NLB (Regional, L4) → closest to AWS NLB
  - TCP/UDP
  - preserves client IP

Internal Application LB (L7) → for internal microservices
Internal Passthrough NLB (L4) → for internal TCP/UDP

Cross-Region LB → global failover for HTTP(S)
```

A key GCP advantage: the global external ALB uses Google's edge PoPs, so your HTTPS traffic enters Google's network at the nearest edge city and travels internally. Much lower latency than AWS for global users.

### Cloud DNS

```bash
# Create a private zone
gcloud dns managed-zones create my-private-zone \
  --description="internal zone" \
  --dns-name="internal.example.com." \
  --visibility=private \
  --networks=my-vpc

# Add an A record
gcloud dns record-sets create app.internal.example.com. \
  --zone=my-private-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=10.0.0.5
```

---

## AWS vs GCP Networking — Side-by-side

| Concept | AWS | GCP |
|---|---|---|
| VPC scope | Regional (subnets per AZ) | Global (subnets per region) |
| Cross-AZ/region internal routing | Need separate VPCs or TGW | Same VPC, automatic |
| Firewall | Security Groups (instance) + NACLs (subnet) | Firewall rules (VPC-level, tag or SA based, stateful) |
| NAT | NAT Gateway (managed, per AZ) | Cloud NAT (per router/region, managed) |
| Hub-and-spoke | Transit Gateway | Network Connectivity Center |
| Private API access | VPC Endpoints (gateway/interface) | Private Google Access / Private Service Connect |
| Multi-project networking | RAM, Peering | Shared VPC (cleaner model) |
| DNS | Route 53 | Cloud DNS |
| CDN | CloudFront | Cloud CDN (integrates with LB) |
| DDoS / WAF | AWS Shield + WAF | Cloud Armor |
| BGP route management | Direct Connect Gateway, TGW route tables | Cloud Router |

---

## Lab 1 — Build the 3-tier VPC (CLI only)

### AWS — commands

```bash
# Variables
VPC_CIDR="10.0.0.0/16"
REGION="us-east-1"
AZ1="us-east-1a"
AZ2="us-east-1b"

# 1. Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block $VPC_CIDR \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=devops-refresh-vpc}]' \
  --query 'Vpc.VpcId' --output text)

# 2. Create subnets
PUB_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone $AZ1 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PRIV_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 --availability-zone $AZ1 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]' \
  --query 'Subnet.SubnetId' --output text)

# 3. Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=devops-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# 4. Route table for public subnet
RTB_PUB=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_PUB \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RTB_PUB --subnet-id $PUB_SUB

# 5. NAT Gateway (needs EIP)
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc \
  --query 'AllocationId' --output text)
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUB --allocation-id $EIP_ALLOC \
  --query 'NatGateway.NatGatewayId' --output text)
# Wait for NAT GW to be available (~60s)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# 6. Route table for private subnet
RTB_PRIV=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $RTB_PRIV \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW
aws ec2 associate-route-table --route-table-id $RTB_PRIV --subnet-id $PRIV_SUB

# 7. Security group — bastion (allow SSH from your IP only)
SG_BASTION=$(aws ec2 create-security-group \
  --group-name bastion-sg --description "Bastion SSH" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
MY_IP=$(curl -s ifconfig.me)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_BASTION --protocol tcp --port 22 --cidr ${MY_IP}/32

# 8. Security group — app (allow SSH from bastion only)
SG_APP=$(aws ec2 create-security-group \
  --group-name app-sg --description "App tier" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP --protocol tcp --port 22 --source-group $SG_BASTION

# 9. Bastion EC2 instance (public subnet)
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \  # Amazon Linux 2 us-east-1 — replace if needed
  --instance-type t3.micro \
  --subnet-id $PUB_SUB \
  --security-group-ids $SG_BASTION \
  --associate-public-ip-address \
  --key-name YOUR_KEY_PAIR \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=bastion}]'

# 10. App EC2 instance (private subnet)
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \
  --instance-type t3.micro \
  --subnet-id $PRIV_SUB \
  --security-group-ids $SG_APP \
  --no-associate-public-ip-address \
  --key-name YOUR_KEY_PAIR \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app-private}]'
```

### GCP — commands

```bash
# Variables
PROJECT_ID="your-project-id"
REGION="us-central1"
gcloud config set project $PROJECT_ID

# 1. Create VPC (custom mode — don't auto-create subnets in every region)
gcloud compute networks create devops-refresh-vpc \
  --subnet-mode=custom

# 2. Create subnets
gcloud compute networks subnets create public-subnet \
  --network=devops-refresh-vpc \
  --region=$REGION \
  --range=10.0.1.0/24

gcloud compute networks subnets create private-subnet \
  --network=devops-refresh-vpc \
  --region=$REGION \
  --range=10.0.2.0/24

# 3. Enable Private Google Access on private subnet
gcloud compute networks subnets update private-subnet \
  --region=$REGION \
  --enable-private-ip-google-access

# 4. Cloud Router + NAT
gcloud compute routers create devops-router \
  --network=devops-refresh-vpc \
  --region=$REGION

gcloud compute routers nats create devops-nat \
  --router=devops-router \
  --region=$REGION \
  --auto-allocate-nat-external-ips \
  --nat-custom-subnet-ip-ranges=private-subnet

# 5. Firewall rules
# Allow SSH to bastion from your IP
MY_IP=$(curl -s ifconfig.me)
gcloud compute firewall-rules create allow-ssh-bastion \
  --network=devops-refresh-vpc \
  --allow=tcp:22 \
  --source-ranges=${MY_IP}/32 \
  --target-tags=bastion

# Allow SSH from bastion to app VMs (tag-based)
gcloud compute firewall-rules create allow-ssh-from-bastion \
  --network=devops-refresh-vpc \
  --allow=tcp:22 \
  --source-tags=bastion \
  --target-tags=app-tier

# Allow egress for private subnet (NAT handles this — allow-egress is default allow in GCP)

# 6. Bastion VM (public subnet with external IP)
gcloud compute instances create bastion \
  --zone=${REGION}-a \
  --machine-type=e2-micro \
  --subnet=public-subnet \
  --tags=bastion \
  --image-family=debian-12 \
  --image-project=debian-cloud

# 7. App VM (private subnet, no external IP)
gcloud compute instances create app-private \
  --zone=${REGION}-a \
  --machine-type=e2-micro \
  --subnet=private-subnet \
  --no-address \
  --tags=app-tier \
  --image-family=debian-12 \
  --image-project=debian-cloud

# 8. SSH to bastion (via IAP — no need to open firewall to 0.0.0.0/0)
gcloud compute ssh bastion --zone=${REGION}-a --tunnel-through-iap

# From bastion, verify private VM can reach internet
# (get the private IP of app-private first)
gcloud compute instances describe app-private \
  --zone=${REGION}-a \
  --format='get(networkInterfaces[0].networkIP)'
```

### Verify the lab works

AWS:
```bash
# From bastion, SSH to private instance using private IP
ssh -i ~/.ssh/key.pem ec2-user@<PRIVATE_IP>
# From private instance, verify outbound internet via NAT
curl -s ifconfig.me   # should return NAT GW's EIP
```

GCP:
```bash
# From bastion
ssh <PRIVATE_IP>   # or use gcloud compute ssh via IAP
# From private VM
curl -s ifconfig.me   # should return one of the Cloud NAT external IPs
```

### Cleanup (important — avoid costs)

```bash
# AWS
aws ec2 terminate-instances --instance-ids <BASTION_ID> <APP_ID>
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW
aws ec2 release-address --allocation-id $EIP_ALLOC
# Wait ~60s for NAT to delete, then:
aws ec2 delete-subnet --subnet-id $PUB_SUB
aws ec2 delete-subnet --subnet-id $PRIV_SUB
aws ec2 delete-route-table --route-table-id $RTB_PUB
aws ec2 delete-route-table --route-table-id $RTB_PRIV
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-vpc --vpc-id $VPC_ID

# GCP
gcloud compute instances delete bastion app-private --zone=${REGION}-a
gcloud compute routers nats delete devops-nat --router=devops-router --region=$REGION
gcloud compute routers delete devops-router --region=$REGION
gcloud compute firewall-rules delete allow-ssh-bastion allow-ssh-from-bastion
gcloud compute networks subnets delete public-subnet private-subnet --region=$REGION
gcloud compute networks delete devops-refresh-vpc
```
