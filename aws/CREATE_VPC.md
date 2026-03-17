# AWS VPC Multi-AZ Setup Guide

## Architecture Overview

```
Region (ap-south-1)
├── VPC: 10.0.0.0/16
│   ├── Internet Gateway
│   ├── AZ ap-south-1a
│   │   ├── Public Subnet:  10.0.0.0/20
│   │   └── Private Subnet: 10.0.128.0/20 → Route to NAT
│   ├── AZ ap-south-1b
│   │   ├── Public Subnet:  10.0.16.0/20
│   │   └── Private Subnet: 10.0.144.0/20 → Route to NAT
│   ├── S3 Gateway Endpoint → attached to private route tables
│   └── Route Tables
│       ├── Public RT  (0.0.0.0/0 → IGW)
│       ├── Private RT-a (0.0.0.0/0 → NAT, S3 prefix → vpce)
│       └── Private RT-b (0.0.0.0/0 → NAT, S3 prefix → vpce)
```

---
---

# PART 1 — AWS CONSOLE ("VPC and more" Wizard)

This creates everything in ONE click.

---

## Step 1: Open the Wizard

1. Go to **VPC Dashboard** → click **Create VPC**.
2. Select **VPC and more** (not "VPC only").

---

## Step 2: Fill in the Settings

### Resources to create
- Select: **VPC and more**

### Name tag auto-generation
- Check: **Auto-generate**
- Enter: `livekit` (this will auto-name all resources like `livekit-vpc`, `livekit-igw`, `livekit-subnet-public1-ap-south-1a`, etc.)

### IPv4 CIDR block
- Enter: `10.0.0.0/16` (65,536 IPs)

### IPv6 CIDR block
- Select: **No IPv6 CIDR block**

### Tenancy
- Select: **Default**

---

## Step 3: Configure Availability Zones

### Number of Availability Zones (AZs)
- Select: **2**

### Customize AZs (expand to verify)
- First AZ: `ap-south-1a`
- Second AZ: `ap-south-1b`

---

## Step 4: Configure Subnets

### Number of public subnets
- Select: **2**

### Number of private subnets
- Select: **2**

### Customize subnets CIDR blocks (expand to verify/edit)

| Subnet | AZ | Default CIDR |
|---|---|---|
| public-subnet-public1-ap-south-1a | ap-south-1a | `10.0.0.0/20` |
| public-subnet-public2-ap-south-1b | ap-south-1b | `10.0.16.0/20` |
| private-subnet-private1-ap-south-1a | ap-south-1a | `10.0.128.0/20` |
| private-subnet-private2-ap-south-1b | ap-south-1b | `10.0.144.0/20` |

> You can keep the defaults or change them. Each /20 gives 4,096 IPs.

---

## Step 5: Configure NAT Gateway

### NAT gateways ($)
- Select: **Regional** (new option — single multi-AZ NAT gateway, cheaper)
    - OR select: **Zonal** (one NAT per AZ — more resilient but costs 2x)

**Regional vs Zonal:**

| Option | NAT Gateways Created | Cost | Resilience |
|---|---|---|---|
| **Regional** | 1 (works across AZs) | ~$32/month | Good (AWS-managed HA) |
| **Zonal** | 2 (one per AZ) | ~$64/month | Best (survives full AZ failure) |

> For most workloads, **Regional** is the recommended choice now.

---

## Step 6: Configure VPC Endpoint

### VPC endpoints
- Select: **S3 Gateway**

> This is free and routes S3 traffic privately, bypassing NAT (saves money).

---

## Step 7: DNS Options

- Check: **Enable DNS hostnames** ✅
- Check: **Enable DNS resolution** ✅

---

## Step 8: Verify Preview and Create

On the right side, you'll see a **Preview** panel showing:

```
VPC:             livekit-vpc
Subnets (4):
  ap-south-1a:   livekit-subnet-public1-ap-south-1a
                  livekit-subnet-private1-ap-south-1a
  ap-south-1b:   livekit-subnet-public2-ap-south-1b
                  livekit-subnet-private2-ap-south-1b
Route tables (3):
                  livekit-rtb-public
                  livekit-rtb-private1-ap-south-1a
                  livekit-rtb-private2-ap-south-1b
Network connections (2):
                  livekit-igw
                  livekit-vpce-s3
```

If NAT is Regional → you'll also see 1 NAT Gateway.
If NAT is Zonal → you'll see 2 NAT Gateways.

Click **Create VPC**.

---

## Step 9: Wait and Verify

1. The wizard will show a progress page creating all resources.
2. Once all show ✅ green checkmarks, click **View VPC**.
3. Verify by checking:
    - **Route Tables** → `livekit-rtb-public` should have `0.0.0.0/0 → igw`
    - **Route Tables** → `livekit-rtb-private1-*` should have `0.0.0.0/0 → nat` and `pl-xxx → vpce`
    - **Endpoints** → S3 endpoint should show **Available**

---

**That's it — one form, one click, full Multi-AZ VPC with IGW + NAT + S3 Endpoint.**

---

## Step 10: Enable Auto-Assign Public IPv4 on Public Subnets

> The wizard does NOT enable this by default. You must do it manually.

1. Go to **VPC** → **Subnets**.
2. Select `livekit-subnet-public1-ap-south-1a`.
3. Click **Actions** → **Edit subnet settings**.
4. Check **Enable auto-assign public IPv4 address** ✅ → Save.
5. Repeat for `livekit-subnet-public2-ap-south-1b`.

---

## Step 11: Add Kubernetes EKS Tags to Subnets

These tags are required for the AWS Load Balancer Controller to auto-discover subnets.

### Tag Public Subnets

1. Select `livekit-subnet-public1-ap-south-1a` → **Tags** tab → **Manage tags**.
2. Click **Add new tag**:
    - **Key:** `kubernetes.io/role/elb`
    - **Value:** `1`
3. Click **Save**.
4. Repeat for `livekit-subnet-public2-ap-south-1b`.

### Tag Private Subnets

1. Select `livekit-subnet-private1-ap-south-1a` → **Tags** tab → **Manage tags**.
2. Click **Add new tag**:
    - **Key:** `kubernetes.io/role/internal-elb`
    - **Value:** `1`
3. Click **Save**.
4. Repeat for `livekit-subnet-private2-ap-south-1b`.

### Summary of EKS Tags

| Subnet Type | Tag Key | Value | Purpose |
|---|---|---|---|
| Public | `kubernetes.io/role/elb` | `1` | ALB Controller places **internet-facing** ALBs here |
| Private | `kubernetes.io/role/internal-elb` | `1` | ALB Controller places **internal** ALBs/NLBs here |

> **Optional:** If you plan to use a specific EKS cluster, also add:
> `kubernetes.io/cluster/<cluster-name>` = `shared` (or `owned`) to all 4 subnets.

---
---

# PART 2 — AWS CLI (Equivalent Setup)

---

> Set your region first: `export AWS_DEFAULT_REGION=ap-south-1`

## Step 1: Create the VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --query 'Vpc.VpcId' \
  --output text)

aws ec2 create-tags --resources $VPC_ID \
  --tags Key=Name,Value=livekit-vpc

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID \
  --enable-dns-support '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID \
  --enable-dns-hostnames '{"Value":true}'

echo "VPC: $VPC_ID"
```

---

## Step 2: Create 4 Subnets

```bash
# Public Subnet 1 — ap-south-1a
PUB_SUB_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.0.0/20 \
  --availability-zone ap-south-1a \
  --query 'Subnet.SubnetId' --output text)
aws ec2 create-tags --resources $PUB_SUB_1A \
  --tags Key=Name,Value=livekit-subnet-public1-ap-south-1a

# Public Subnet 2 — ap-south-1b
PUB_SUB_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.16.0/20 \
  --availability-zone ap-south-1b \
  --query 'Subnet.SubnetId' --output text)
aws ec2 create-tags --resources $PUB_SUB_1B \
  --tags Key=Name,Value=livekit-subnet-public2-ap-south-1b

# Private Subnet 1 — ap-south-1a
PRIV_SUB_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.128.0/20 \
  --availability-zone ap-south-1a \
  --query 'Subnet.SubnetId' --output text)
aws ec2 create-tags --resources $PRIV_SUB_1A \
  --tags Key=Name,Value=livekit-subnet-private1-ap-south-1a

# Private Subnet 2 — ap-south-1b
PRIV_SUB_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.144.0/20 \
  --availability-zone ap-south-1b \
  --query 'Subnet.SubnetId' --output text)
aws ec2 create-tags --resources $PRIV_SUB_1B \
  --tags Key=Name,Value=livekit-subnet-private2-ap-south-1b

# Enable auto-assign public IP on public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUB_1A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUB_1B --map-public-ip-on-launch

echo "Public 1a: $PUB_SUB_1A | Public 1b: $PUB_SUB_1B"
echo "Private 1a: $PRIV_SUB_1A | Private 1b: $PRIV_SUB_1B"
```

### Enable Auto-Assign Public IPv4 on Public Subnets

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUB_1A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUB_1B --map-public-ip-on-launch
```

### Add Kubernetes EKS Tags

```bash
# Tag PUBLIC subnets for internet-facing load balancers
aws ec2 create-tags --resources $PUB_SUB_1A $PUB_SUB_1B \
  --tags Key=kubernetes.io/role/elb,Value=1

# Tag PRIVATE subnets for internal load balancers
aws ec2 create-tags --resources $PRIV_SUB_1A $PRIV_SUB_1B \
  --tags Key=kubernetes.io/role/internal-elb,Value=1

# (Optional) Tag ALL subnets for a specific EKS cluster
# Replace <cluster-name> with your actual cluster name
# aws ec2 create-tags \
#   --resources $PUB_SUB_1A $PUB_SUB_1B $PRIV_SUB_1A $PRIV_SUB_1B \
#   --tags Key=kubernetes.io/cluster/<cluster-name>,Value=shared
```

---

## Step 3: Create and Attach Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 create-tags --resources $IGW_ID \
  --tags Key=Name,Value=livekit-igw

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

echo "IGW: $IGW_ID"
```

---

## Step 4: Allocate Elastic IP + Create NAT Gateway

```bash
# Allocate 1 EIP for Regional NAT
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' --output text)
aws ec2 create-tags --resources $EIP_ALLOC \
  --tags Key=Name,Value=livekit-nat-eip

# Create NAT Gateway in public subnet 1a
NAT_GW=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUB_1A \
  --allocation-id $EIP_ALLOC \
  --query 'NatGateway.NatGatewayId' --output text)
aws ec2 create-tags --resources $NAT_GW \
  --tags Key=Name,Value=livekit-nat

# Wait until available
echo "Waiting for NAT Gateway..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW
echo "NAT Gateway ready: $NAT_GW"
```

> For **Zonal** (2 NAT Gateways), repeat with a second EIP and create a
> second NAT in `PUB_SUB_1B`, then point each private RT to its own NAT.

---

## Step 5: Create Route Tables and Add Routes

```bash
# === Public Route Table ===
PUB_RT=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-tags --resources $PUB_RT \
  --tags Key=Name,Value=livekit-rtb-public

aws ec2 create-route \
  --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

aws ec2 associate-route-table \
  --route-table-id $PUB_RT --subnet-id $PUB_SUB_1A
aws ec2 associate-route-table \
  --route-table-id $PUB_RT --subnet-id $PUB_SUB_1B

# === Private Route Table 1 (ap-south-1a) ===
PRIV_RT_A=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-tags --resources $PRIV_RT_A \
  --tags Key=Name,Value=livekit-rtb-private1-ap-south-1a

aws ec2 create-route \
  --route-table-id $PRIV_RT_A \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW

aws ec2 associate-route-table \
  --route-table-id $PRIV_RT_A --subnet-id $PRIV_SUB_1A

# === Private Route Table 2 (ap-south-1b) ===
PRIV_RT_B=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-tags --resources $PRIV_RT_B \
  --tags Key=Name,Value=livekit-rtb-private2-ap-south-1b

aws ec2 create-route \
  --route-table-id $PRIV_RT_B \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW

aws ec2 associate-route-table \
  --route-table-id $PRIV_RT_B --subnet-id $PRIV_SUB_1B

echo "Public RT: $PUB_RT"
echo "Private RT-A: $PRIV_RT_A | Private RT-B: $PRIV_RT_B"
```

---

## Step 6: Create S3 Gateway VPC Endpoint

```bash
S3_VPCE=$(aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.s3 \
  --route-table-ids $PRIV_RT_A $PRIV_RT_B \
  --query 'VpcEndpoint.VpcEndpointId' --output text)

aws ec2 create-tags --resources $S3_VPCE \
  --tags Key=Name,Value=livekit-vpce-s3

echo "S3 Endpoint: $S3_VPCE"
```

---

## Step 7: Verify

```bash
echo "=== Public Route Table ==="
aws ec2 describe-route-tables --route-table-ids $PUB_RT \
  --query 'RouteTables[0].Routes[*].{Dest:DestinationCidrBlock,Target:GatewayId}' \
  --output table

echo "=== Private Route Table A ==="
aws ec2 describe-route-tables --route-table-ids $PRIV_RT_A \
  --query 'RouteTables[0].Routes[*].{CIDR:DestinationCidrBlock,PrefixList:DestinationPrefixListId,GW:GatewayId,NAT:NatGatewayId}' \
  --output table

echo "=== Private Route Table B ==="
aws ec2 describe-route-tables --route-table-ids $PRIV_RT_B \
  --query 'RouteTables[0].Routes[*].{CIDR:DestinationCidrBlock,PrefixList:DestinationPrefixListId,GW:GatewayId,NAT:NatGatewayId}' \
  --output table

echo "=== S3 Endpoint ==="
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids $S3_VPCE \
  --query 'VpcEndpoints[0].{Id:VpcEndpointId,Service:ServiceName,State:State}' \
  --output table
```

**Expected output for each private RT:**

| CIDR | PrefixList | GW | NAT |
|---|---|---|---|
| 10.0.0.0/16 | None | local | None |
| 0.0.0.0/0 | None | None | nat-xxxxx |
| None | pl-xxxxx | vpce-xxxxx | None |

---

## Cleanup Script

```bash
# 1. Delete S3 Endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $S3_VPCE

# 2. Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW
echo "Waiting for NAT deletion..." && sleep 60

# 3. Release Elastic IP
aws ec2 release-address --allocation-id $EIP_ALLOC

# 4. Delete route tables (disassociate first)
for RT in $PUB_RT $PRIV_RT_A $PRIV_RT_B; do
  ASSOCS=$(aws ec2 describe-route-tables --route-table-ids $RT \
    --query 'RouteTables[0].Associations[?!Main].RouteTableAssociationId' \
    --output text)
  for A in $ASSOCS; do
    aws ec2 disassociate-route-table --association-id $A
  done
  aws ec2 delete-route-table --route-table-id $RT
done

# 5. Detach and delete IGW
aws ec2 detach-internet-gateway \
  --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 6. Delete subnets
for S in $PUB_SUB_1A $PUB_SUB_1B $PRIV_SUB_1A $PRIV_SUB_1B; do
  aws ec2 delete-subnet --subnet-id $S
done

# 7. Delete VPC
aws ec2 delete-vpc --vpc-id $VPC_ID
echo "Cleanup complete."
```

---

## Console vs CLI — Quick Comparison

| | Console ("VPC and more") | CLI |
|---|---|---|
| **Steps** | 1 form → Create | ~15 commands |
| **Time** | ~2 minutes | ~5 minutes |
| **Automation** | Manual, not repeatable | Scriptable, version-controllable |
| **Best for** | Quick setup, learning | CI/CD, IaC, repeatable environments |
| **What it creates** | VPC + Subnets + IGW + NAT + RTs + S3 Endpoint | Same, but you control every detail |