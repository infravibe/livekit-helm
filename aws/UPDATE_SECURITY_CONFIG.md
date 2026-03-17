# UPDATE SECURITY CONFIG

Configure security groups to allow **UDP 50000–60000** for LiveKit WebRTC media traffic on the EKS cluster.

---

## Prerequisites

- **VPC:** `livekit-vpc` — see [CREATE_VPC.md](./CREATE_VPC.md)
- **EKS Cluster:** `livekit-cluster` must be running — see [CREATE_EKS_CLUSTER.md](./CREATE_EKS_CLUSTER.md)

---

## Why These Ports?

LiveKit uses UDP ports 50000–60000 for **WebRTC media transport** (audio, video, data channels). Without these ports open, participants cannot send or receive media streams.

```
Client (Browser/App)
    │
    │  UDP 50000–60000 (media)
    ▼
[ Internet ]
    │
    ▼
[ ALB / NLB ]  ← or NodePort on worker nodes
    │
    ▼
[ EKS Worker Nodes ]
    │
    ▼
[ LiveKit Server Pod ]
```

---

## Which Security Group to Update?

There are typically 2 security groups associated with your EKS cluster:

| Security Group | Name Pattern | Purpose |
|---|---|---|
| **Cluster SG** | `eks-cluster-sg-livekit-cluster-*` | Auto-created by EKS, shared by control plane and nodes |
| **Node SG** | `eksctl-livekit-cluster-nodegroup-*` or `livekit-nodes-*` | Attached to worker node ENIs |

You need to add the UDP rule to the **Node security group** (the one attached to your EC2 worker nodes) so traffic can reach the LiveKit pods.

> If you're using `hostNetwork: true` for LiveKit (common for WebRTC), the node SG is the one that matters.

---
---

# STEP 1: Identify the Node Security Group

---

### AWS Console

1. Go to **EC2** → **Instances**.
2. Find one of the worker nodes (named like `livekit-nodes-*` or tagged with `eks:nodegroup-name = livekit-nodes`).
3. Click the instance → **Security** tab.
4. Note the **Security group ID(s)** — typically you'll see:
    - `eks-cluster-sg-livekit-cluster-*` (cluster SG)
    - Possibly an additional node-specific SG
5. You'll add the rule to the **cluster security group** (the one starting with `eks-cluster-sg-`), as this is shared by all nodes.

Alternatively:

1. Go to **EKS** → `livekit-cluster` → **Networking** tab.
2. Under **Cluster security group**, note the **Security group ID**.

### AWS CLI

```bash
# Method 1: Get the cluster security group directly from EKS
CLUSTER_SG=$(aws eks describe-cluster \
  --name livekit-cluster \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' \
  --output text)

echo "Cluster SG: $CLUSTER_SG"

# Method 2: Find SGs attached to worker nodes
NODE_INSTANCE=$(aws ec2 describe-instances \
  --filters "Name=tag:eks:nodegroup-name,Values=livekit-nodes" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)

aws ec2 describe-instances \
  --instance-ids $NODE_INSTANCE \
  --query 'Reservations[0].Instances[0].SecurityGroups[*].{ID:GroupId,Name:GroupName}' \
  --output table
```

---
---

# STEP 2: Add UDP 50000–60000 Inbound Rule

---

### AWS Console

1. Go to **EC2** → **Security Groups**.
2. Search for and select the cluster security group (`eks-cluster-sg-livekit-cluster-*`).
3. Click the **Inbound rules** tab → **Edit inbound rules**.
4. Click **Add rule** and fill in:

   | Field | Value |
      |---|---|
   | **Type** | Custom UDP |
   | **Port range** | `50000 - 60000` |
   | **Source** | `0.0.0.0/0` (Anywhere IPv4) |
   | **Description** | `LiveKit WebRTC media UDP` |

5. *(Optional)* If you need IPv6 support, click **Add rule** again:

   | Field | Value |
      |---|---|
   | **Type** | Custom UDP |
   | **Port range** | `50000 - 60000` |
   | **Source** | `::/0` (Anywhere IPv6) |
   | **Description** | `LiveKit WebRTC media UDP IPv6` |

6. Click **Save rules**.

### AWS CLI

```bash
# Get the cluster security group
CLUSTER_SG=$(aws eks describe-cluster \
  --name livekit-cluster \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' \
  --output text)

# Add UDP 50000-60000 from anywhere (IPv4)
aws ec2 authorize-security-group-ingress \
  --group-id $CLUSTER_SG \
  --ip-permissions \
    IpProtocol=udp,\
FromPort=50000,\
ToPort=60000,\
IpRanges='[{CidrIp=0.0.0.0/0,Description="LiveKit WebRTC media UDP"}]'

echo "Added UDP 50000-60000 (IPv4) to $CLUSTER_SG"

# (Optional) Add IPv6
aws ec2 authorize-security-group-ingress \
  --group-id $CLUSTER_SG \
  --ip-permissions \
    IpProtocol=udp,\
FromPort=50000,\
ToPort=60000,\
Ipv6Ranges='[{CidrIpv6=::/0,Description="LiveKit WebRTC media UDP IPv6"}]'

echo "Added UDP 50000-60000 (IPv6) to $CLUSTER_SG"
```

---
---

# STEP 3: Verify the Rules

---

### AWS Console

1. Go to **EC2** → **Security Groups** → select the cluster SG.
2. Click **Inbound rules** tab.
3. Confirm you see:

   | Type | Protocol | Port range | Source | Description |
      |---|---|---|---|---|
   | Custom UDP | UDP | 50000 - 60000 | 0.0.0.0/0 | LiveKit WebRTC media UDP |
   | Custom UDP | UDP | 50000 - 60000 | ::/0 | LiveKit WebRTC media UDP IPv6 |

### AWS CLI

```bash
aws ec2 describe-security-group-rules \
  --filters "Name=group-id,Values=$CLUSTER_SG" \
  --query 'SecurityGroupRules[?FromPort==`50000`].{Direction:IsEgress,Protocol:IpProtocol,From:FromPort,To:ToPort,Source:CidrIpv4||CidrIpv6,Desc:Description}' \
  --output table
```

Expected output:

```
-----------------------------------------------------------------
| Direction | Protocol | From  | To    | Source    | Desc       |
| False     | udp      | 50000 | 60000 | 0.0.0.0/0| LiveKit... |
-----------------------------------------------------------------
```

---
---

# STEP 4: Test UDP Connectivity (Optional)

---

From a machine outside the VPC, test that UDP traffic reaches your nodes:

```bash
# Get a worker node's public/external IP (if using NodePort)
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')

# If nodes are in private subnets (no external IP), use the NLB DNS or
# the service's external endpoint instead
# NODE_IP=<your-nlb-dns-or-public-endpoint>

# Test with netcat (from your local machine)
nc -zuv $NODE_IP 50000
# Expected: Connection to <IP> 50000 port [udp/*] succeeded!
```

Or from inside the cluster:

```bash
# Run a test pod
kubectl run udp-test --image=busybox --rm -it --restart=Never -- sh

# Inside the pod, test connectivity
nc -zuv <livekit-pod-ip> 50000
exit
```

---
---

# Security Considerations

**`0.0.0.0/0` opens these ports to the entire internet.** This is expected for WebRTC since clients connect from arbitrary IPs. However, keep in mind:

- LiveKit's built-in authentication (API key + secret) protects against unauthorized usage at the application layer.
- Only UDP media traffic flows through these ports — no management or control plane access.
- If you know your clients will always come from specific CIDR ranges (e.g., a corporate VPN), restrict the source accordingly:
  ```bash
  # Example: restrict to specific CIDR
  aws ec2 authorize-security-group-ingress \
    --group-id $CLUSTER_SG \
    --ip-permissions \
      IpProtocol=udp,\
  FromPort=50000,\
  ToPort=60000,\
  IpRanges='[{CidrIp=203.0.113.0/24,Description="LiveKit WebRTC - Office only"}]'
  ```

---
---

# Cleanup (Remove the Rule)

---

### AWS Console

1. Go to **EC2** → **Security Groups** → select the cluster SG.
2. **Inbound rules** tab → **Edit inbound rules**.
3. Click **Delete** (X) next to the UDP 50000–60000 rule(s).
4. Click **Save rules**.

### AWS CLI

```bash
# Remove IPv4 rule
aws ec2 revoke-security-group-ingress \
  --group-id $CLUSTER_SG \
  --ip-permissions \
    IpProtocol=udp,\
FromPort=50000,\
ToPort=60000,\
IpRanges='[{CidrIp=0.0.0.0/0}]'

# Remove IPv6 rule
aws ec2 revoke-security-group-ingress \
  --group-id $CLUSTER_SG \
  --ip-permissions \
    IpProtocol=udp,\
FromPort=50000,\
ToPort=60000,\
Ipv6Ranges='[{CidrIpv6=::/0}]'

echo "UDP rules removed from $CLUSTER_SG"
```

---
---

# Quick Reference

| Setting | Value |
|---|---|
| Protocol | UDP |
| Port Range | 50000 – 60000 |
| Source | 0.0.0.0/0 (and optionally ::/0) |
| Security Group | EKS Cluster SG (`eks-cluster-sg-livekit-cluster-*`) |
| Purpose | LiveKit WebRTC media transport |
| Direction | Inbound |