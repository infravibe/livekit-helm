# CREATE EKS CLUSTER

Complete guide to create an AWS EKS cluster with Node Group, OIDC Provider, and AWS Load Balancer Controller.

---

## Prerequisites

- **VPC:** Must be created first using the [VPC Setup Guide (CREATE_VPC.md)](./CREATE_VPC.md)
    - VPC Name: `livekit`
    - 2 AZs (ap-south-1a, ap-south-1b)
    - 2 Public Subnets (tagged `kubernetes.io/role/elb = 1`)
    - 2 Private Subnets (tagged `kubernetes.io/role/internal-elb = 1`)
    - NAT Gateway, Internet Gateway, S3 Endpoint
- **Tools Required (for CLI method):**
    - AWS CLI v2 (`aws --version`)
    - `kubectl` ([install guide](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html))
    - `eksctl` ([install guide](https://eksctl.io/installation/))
    - `helm` ([install guide](https://helm.sh/docs/intro/install/))

---

## Architecture

```
livekit-vpc (10.0.0.0/16)
│
├── EKS Cluster: livekit-cluster
│   ├── OIDC Provider (for IAM Roles for Service Accounts)
│   ├── Cluster Security Group (auto-created)
│   └── Cluster IAM Role: livekit-eks-cluster-role
│
├── Managed Node Group: livekit-nodes
│   ├── Instance Type: t3.medium
│   ├── Desired: 2 | Min: 1 | Max: 4
│   ├── Deployed in: Private Subnets
│   └── Node IAM Role: livekit-eks-node-role
│
├── AWS Load Balancer Controller
│   ├── Service Account: aws-load-balancer-controller
│   ├── IAM Role: livekit-alb-controller-role (via OIDC)
│   └── Installed via Helm in kube-system namespace
│
├── Public Subnets  → Internet-facing ALBs (Ingress)
└── Private Subnets → Worker Nodes + Internal LBs
```

---
---

# STEP 1: Create IAM Roles

---

## 1a. Create EKS Cluster IAM Role

This role allows the EKS service to manage AWS resources on your behalf.

### AWS Console

1. Go to **IAM** → **Roles** → **Create role**.
2. **Trusted entity type:** Select **AWS service**.
3. **Use case:** Under the dropdown, search and select **EKS** → choose **EKS - Cluster** → click **Next**.
    - This automatically selects the `AmazonEKSClusterPolicy`.
4. **Permissions:** Verify `AmazonEKSClusterPolicy` is listed → click **Next**.
5. **Role name:** `livekit-eks-cluster-role`
6. Click **Create role**.

### AWS CLI

```bash
# Create trust policy
cat > eks-cluster-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name livekit-eks-cluster-role \
  --assume-role-policy-document file://eks-cluster-trust-policy.json

# Attach required policy
aws iam attach-role-policy \
  --role-name livekit-eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

---

## 1b. Create EKS Node Group IAM Role

This role is assumed by the EC2 worker nodes.

### AWS Console

1. Go to **IAM** → **Roles** → **Create role**.
2. **Trusted entity type:** Select **AWS service**.
3. **Use case:** Select **EC2** → click **Next**.
4. **Permissions:** Search and check all 3 policies below:
    - `AmazonEKSWorkerNodePolicy`
    - `AmazonEKS_CNI_Policy`
    - `AmazonEC2ContainerRegistryReadOnly`
5. Click **Next**.
6. **Role name:** `livekit-eks-node-role`
7. Click **Create role**.

### AWS CLI

```bash
# Create trust policy
cat > eks-node-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name livekit-eks-node-role \
  --assume-role-policy-document file://eks-node-trust-policy.json

# Attach all 3 required policies
aws iam attach-role-policy \
  --role-name livekit-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name livekit-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy \
  --role-name livekit-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

---
---

# STEP 2: Gather VPC Information

---

### AWS Console

1. Go to **VPC** → **Your VPCs** → find `livekit-vpc` → note the **VPC ID**.
2. Go to **Subnets** → filter by `livekit-vpc`:
    - Note the 2 **private** subnet IDs (names contain `private`)
    - Note the 2 **public** subnet IDs (names contain `public`)
3. Go to top-right corner → click your username → note your **Account ID**.

### AWS CLI

```bash
# Get VPC ID by name
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=livekit-vpc" \
  --query 'Vpcs[0].VpcId' --output text)

# Get Private Subnet IDs
PRIV_SUBS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
            "Name=tag:Name,Values=*private*" \
  --query 'Subnets[*].SubnetId' --output text)
PRIV_SUB_1=$(echo $PRIV_SUBS | awk '{print $1}')
PRIV_SUB_2=$(echo $PRIV_SUBS | awk '{print $2}')

# Get Public Subnet IDs
PUB_SUBS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
            "Name=tag:Name,Values=*public*" \
  --query 'Subnets[*].SubnetId' --output text)
PUB_SUB_1=$(echo $PUB_SUBS | awk '{print $1}')
PUB_SUB_2=$(echo $PUB_SUBS | awk '{print $2}')

# Get Account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)

echo "VPC:         $VPC_ID"
echo "Private:     $PRIV_SUB_1, $PRIV_SUB_2"
echo "Public:      $PUB_SUB_1, $PUB_SUB_2"
echo "Account ID:  $ACCOUNT_ID"
```

---
---

# STEP 3: Create EKS Cluster

---

### AWS Console

1. Go to **EKS** → **Clusters** → **Create cluster**.
2. **Step 1 — Configure cluster:**
    - **Name:** `livekit-cluster`
    - **Kubernetes version:** `1.33`
    - **Cluster IAM role:** Select `livekit-eks-cluster-role`
    - Click **Next**.
3. **Step 2 — Specify networking:**
    - **VPC:** Select `livekit-vpc`
    - **Subnets:** Select all 4 subnets (2 public + 2 private)
    - **Security groups:** Leave default (EKS auto-creates one)
    - **Cluster endpoint access:** Select **Public and private**
    - Click **Next**.
4. **Step 3 — Configure observability:**
    - Leave defaults (or optionally enable CloudWatch logging for API server, audit, etc.)
    - Click **Next**.
5. **Step 4 — Select add-ons:**
    - Keep the defaults: CoreDNS, kube-proxy, Amazon VPC CNI
    - Click **Next**.
6. **Step 5 — Configure selected add-ons settings:**
    - Keep defaults → click **Next**.
7. **Step 6 — Review and create:**
    - Review all settings → click **Create**.
8. Wait **10–15 minutes** until status changes to **Active**.

### AWS CLI

```bash
aws eks create-cluster \
  --name livekit-cluster \
  --region ap-south-1 \
  --kubernetes-version 1.31 \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/livekit-eks-cluster-role \
  --resources-vpc-config \
    subnetIds=${PRIV_SUB_1},${PRIV_SUB_2},${PUB_SUB_1},${PUB_SUB_2},\
endpointPublicAccess=true,\
endpointPrivateAccess=true

echo "Cluster creation started. Takes 10-15 minutes..."

# Wait for cluster to become ACTIVE
aws eks wait cluster-active --name livekit-cluster
echo "Cluster is ACTIVE!"
```

---

## Update kubeconfig (Required for Both Methods)

After the cluster is active, run this to connect `kubectl`:

```bash
aws eks update-kubeconfig \
  --name livekit-cluster \
  --region ap-south-1

# Verify connection
kubectl get svc
# Should show: kubernetes   ClusterIP   10.100.0.1   443
```

---
---

# STEP 4: Associate OIDC Provider

---

The OIDC provider enables **IAM Roles for Service Accounts (IRSA)** — this is how the ALB Controller (and other pods) get AWS permissions securely.

### AWS Console

1. Go to **EKS** → click `livekit-cluster` → **Overview** tab.
2. In the **Details** section, find **OpenID Connect provider URL** → click **Copy**.
    - It looks like: `https://oidc.eks.ap-south-1.amazonaws.com/id/ABCDEF1234567890`
3. Go to **IAM** → **Identity providers** (left sidebar) → **Add provider**.
4. **Provider type:** Select **OpenID Connect**.
5. **Provider URL:** Paste the URL you copied → click **Get thumbprint**.
    - Wait for the thumbprint to appear and show a green checkmark.
6. **Audience:** Enter `sts.amazonaws.com`
7. Click **Add provider**.
8. You should see the new provider listed under Identity providers.

### AWS CLI (using eksctl — recommended)

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster livekit-cluster \
  --region ap-south-1 \
  --approve
```

### AWS CLI (manual)

```bash
# Get the OIDC issuer URL
OIDC_URL=$(aws eks describe-cluster \
  --name livekit-cluster \
  --query 'cluster.identity.oidc.issuer' \
  --output text)

OIDC_ID=$(echo $OIDC_URL | awk -F'/' '{print $NF}')

# Check if already exists
aws iam list-open-id-connect-providers | grep $OIDC_ID

# If not found, get thumbprint and create
THUMBPRINT=$(echo | openssl s_client -servername oidc.eks.ap-south-1.amazonaws.com \
  -connect oidc.eks.ap-south-1.amazonaws.com:443 2>/dev/null | \
  openssl x509 -fingerprint -noout | sed 's/://g' | awk -F= '{print tolower($2)}')

aws iam create-open-id-connect-provider \
  --url $OIDC_URL \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list $THUMBPRINT

echo "OIDC Provider associated!"
```

---
---

# STEP 5: Create Managed Node Group

---

### AWS Console

1. Go to **EKS** → click `livekit-cluster` → **Compute** tab.
2. Under **Node groups**, click **Add node group**.
3. **Step 1 — Configure node group:**
    - **Name:** `livekit-nodes`
    - **Node IAM role:** Select `livekit-eks-node-role`
    - Click **Next**.
4. **Step 2 — Set compute and scaling configuration:**
    - **AMI type:** Amazon Linux 2 (AL2_x86_64)
    - **Capacity type:** On-Demand
    - **Instance types:** `t3.medium`
    - **Disk size:** `20` GB
    - **Scaling configuration:**
        - Minimum size: `1`
        - Maximum size: `4`
        - Desired size: `2`
    - Click **Next**.
5. **Step 3 — Specify networking:**
    - **Subnets:** Select only the **2 public subnets** (deselect private ones)
        - `livekit-subnet-public1-ap-south-1a`
        - `livekit-subnet-public2-ap-south-1b`
    - **Configure SSH access to nodes:** (Optional)
        - If needed, select an EC2 key pair and allowed security group
    - Click **Next**.
6. **Step 4 — Review** → click **Create**.
7. Wait **3–5 minutes** until status shows **Active**.

### AWS CLI

```bash
aws eks create-nodegroup \
  --cluster-name livekit-cluster \
  --nodegroup-name livekit-nodes \
  --node-role arn:aws:iam::${ACCOUNT_ID}:role/livekit-eks-node-role \
  --subnets $PUB_SUB_1 $PUB_SUB_2 \
  --instance-types t3.medium \
  --scaling-config minSize=1,maxSize=4,desiredSize=2 \
  --capacity-type ON_DEMAND \
  --disk-size 20

echo "Node group creation started. Takes 3-5 minutes..."

aws eks wait nodegroup-active \
  --cluster-name livekit-cluster \
  --nodegroup-name livekit-nodes

echo "Node group ACTIVE!"
```

---

## Verify Nodes (Both Methods)

```bash
kubectl get nodes
# Should show 2 nodes in Ready state

kubectl get nodes -o wide
# Verify INTERNAL-IP is from private subnet CIDR (10.0.128.x / 10.0.144.x)
```

---
---

# STEP 6: Install AWS Load Balancer Controller

---

## 6a. Create IAM Policy for ALB Controller

### AWS Console

1. Go to **IAM** → **Policies** → **Create policy**.
2. Click the **JSON** tab.
3. Open this URL in a new tab and copy the entire JSON:
   `https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json`
4. Paste the JSON into the policy editor (replace the default content).
5. Click **Next**.
6. **Policy name:** `AWSLoadBalancerControllerIAMPolicy`
7. Click **Create policy**.

### AWS CLI

```bash
# Download the official IAM policy
curl -o alb-iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

# Create the policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-iam-policy.json

echo "Policy ARN: arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy"
```

---

## 6b. Create IAM Role for ALB Controller (Using IRSA)

This uses the OIDC provider from Step 4 to let the controller pod assume an IAM role.

### AWS Console

1. Go to **IAM** → **Roles** → **Create role**.
2. **Trusted entity type:** Select **Web identity**.
3. **Identity provider:** Select your EKS OIDC provider from the dropdown
    - It looks like: `oidc.eks.ap-south-1.amazonaws.com/id/ABCDEF1234567890`
4. **Audience:** Select `sts.amazonaws.com`
5. Click **Next**.
6. **Permissions:** Search and select `AWSLoadBalancerControllerIAMPolicy` → click **Next**.
7. **Role name:** `livekit-alb-controller-role`
8. Click **Create role**.
9. **IMPORTANT — Edit the trust policy to restrict to the service account:**
    - Go to the newly created role → **Trust relationships** tab → **Edit trust policy**.
    - Find the `Condition` block and change it to:
   ```json
   "Condition": {
     "StringEquals": {
       "oidc.eks.ap-south-1.amazonaws.com/id/<YOUR_OIDC_ID>:aud": "sts.amazonaws.com",
       "oidc.eks.ap-south-1.amazonaws.com/id/<YOUR_OIDC_ID>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
     }
   }
   ```
    - Replace `<YOUR_OIDC_ID>` with your actual OIDC ID (the last part of the OIDC URL).
    - Click **Update policy**.
10. Now create the Kubernetes service account. Copy the **Role ARN** from the role summary page, then run:
    ```bash
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
      annotations:
        eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/livekit-alb-controller-role
    EOF
    ```

### AWS CLI (using eksctl — recommended)

```bash
eksctl create iamserviceaccount \
  --cluster livekit-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region ap-south-1
```

### AWS CLI (manual)

```bash
# Get OIDC ID
OIDC_ID=$(aws eks describe-cluster \
  --name livekit-cluster \
  --query 'cluster.identity.oidc.issuer' \
  --output text | awk -F'/' '{print $NF}')

# Create trust policy
cat > alb-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/${OIDC_ID}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/${OIDC_ID}:aud": "sts.amazonaws.com",
          "oidc.eks.ap-south-1.amazonaws.com/id/${OIDC_ID}:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name livekit-alb-controller-role \
  --assume-role-policy-document file://alb-trust-policy.json

# Attach the policy
aws iam attach-role-policy \
  --role-name livekit-alb-controller-role \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy

# Create the Kubernetes service account
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::${ACCOUNT_ID}:role/livekit-alb-controller-role
EOF
```

---

## 6c. Install ALB Controller via Helm

> This step requires Helm (CLI only). There is no Console equivalent — the ALB Controller is a Kubernetes add-on installed via Helm.

```bash
# Add the EKS Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=livekit-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=$VPC_ID
```

### Alternative — Install via EKS Console Add-on (if available)

1. Go to **EKS** → `livekit-cluster` → **Add-ons** tab.
2. Click **Get more add-ons**.
3. Search for **AWS Load Balancer Controller**.
4. If listed, select it → choose the latest version → select the `livekit-alb-controller-role` IAM role → **Create**.


---

## 6d. Verify ALB Controller (Both Methods)

```bash
# Check pods are running
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# Expected output:
# NAME                                            READY   STATUS    RESTARTS   AGE
# aws-load-balancer-controller-xxxxxxxxx-xxxxx    1/1     Running   0          1m
# aws-load-balancer-controller-xxxxxxxxx-xxxxx    1/1     Running   0          1m

# Check logs for errors
kubectl logs -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller --tail=20

# Verify the webhook
kubectl get mutatingwebhookconfigurations
# Should show: aws-load-balancer-webhook
```

### Verify in Console

1. Go to **EC2** → **Load Balancers** — no ALBs yet (expected, we haven't created an Ingress).
2. Go to **EC2** → **Target Groups** — empty (expected).
3. Go to **EKS** → `livekit-cluster` → **Resources** tab → **Workloads** → filter namespace `kube-system` — you should see the `aws-load-balancer-controller` deployment.

---
---

# STEP 7: Test with a Sample Ingress (Optional)

---

Deploy a sample app to verify the ALB Controller creates an ALB:

```bash
# Deploy a sample nginx app + service + ingress
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx-test
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-test-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-test
            port:
              number: 80
EOF

# Wait and check ALB creation
echo "Waiting for ALB to provision (~1-2 min)..."
sleep 60

kubectl get ingress nginx-test-ingress
# ADDRESS column should show an ALB DNS name like:
# k8s-default-nginxtes-xxxxxxxxxx-xxxxxxxxx.ap-south-1.elb.amazonaws.com
```

### Verify in Console

1. Go to **EC2** → **Load Balancers** → you should see a new ALB named `k8s-default-nginxtes-*`.
2. Click it → **Description** tab → copy the **DNS name**.
3. Open the DNS name in a browser → should show the **nginx welcome page**.
4. Go to **Target Groups** → verify healthy targets registered.

### Test via CLI

```bash
ALB_DNS=$(kubectl get ingress nginx-test-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$ALB_DNS
# Should return the nginx welcome page HTML
```

### Cleanup Test Resources

```bash
kubectl delete ingress nginx-test-ingress
kubectl delete svc nginx-test
kubectl delete deployment nginx-test
```

---
---

# Verification Checklist

```bash
echo "========== CLUSTER =========="
aws eks describe-cluster --name livekit-cluster \
  --query 'cluster.{Status:status,Version:version,Endpoint:endpoint}' \
  --output table

echo "========== OIDC =========="
aws eks describe-cluster --name livekit-cluster \
  --query 'cluster.identity.oidc.issuer' --output text

echo "========== NODE GROUP =========="
aws eks describe-nodegroup \
  --cluster-name livekit-cluster \
  --nodegroup-name livekit-nodes \
  --query 'nodegroup.{Status:status,DesiredSize:scalingConfig.desiredSize,InstanceTypes:instanceTypes}' \
  --output table

echo "========== NODES =========="
kubectl get nodes -o wide

echo "========== ALB CONTROLLER =========="
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

echo "========== SERVICE ACCOUNTS =========="
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml | grep role-arn
```

### Console Verification

| Check | Where to Verify |
|---|---|
| Cluster is Active | **EKS** → Clusters → `livekit-cluster` → Status: **Active** |
| OIDC Provider exists | **IAM** → Identity providers → shows `oidc.eks.ap-south-1...` |
| Node Group is Active | **EKS** → `livekit-cluster` → Compute tab → `livekit-nodes`: **Active** |
| Nodes are running | **EC2** → Instances → 2 instances tagged `livekit-nodes` in **Running** state |
| ALB Controller running | **EKS** → `livekit-cluster` → Resources → Workloads → `aws-load-balancer-controller` |
| IAM Roles exist | **IAM** → Roles → search `livekit` → should see 3 roles |

### Expected Results

| Check | Expected |
|---|---|
| Cluster status | `ACTIVE` |
| OIDC URL | `https://oidc.eks.ap-south-1.amazonaws.com/id/...` |
| Node group status | `ACTIVE` |
| Nodes | 2 nodes, `Ready` status |
| ALB Controller pods | 2 pods, `Running` |
| Service account | Shows `role-arn` annotation |

---
---

# Cleanup (Tear Down Everything)

---

Run in this order to avoid dependency errors.

## Console Cleanup

1. **Delete Node Group:**
    - **EKS** → `livekit-cluster` → **Compute** tab → select `livekit-nodes` → **Delete**.
    - Wait until fully deleted.

2. **Delete EKS Cluster:**
    - **EKS** → Clusters → select `livekit-cluster` → **Delete**.
    - Type the cluster name to confirm → wait until deleted.

3. **Delete OIDC Provider:**
    - **IAM** → **Identity providers** → select the EKS OIDC provider → **Delete**.

4. **Delete IAM Roles:**
    - **IAM** → **Roles** → for each role below:
        - Click the role → **Permissions** tab → remove all attached policies → **Delete role**.
        - `livekit-eks-cluster-role`
        - `livekit-eks-node-role`
        - `livekit-alb-controller-role`

5. **Delete IAM Policy:**
    - **IAM** → **Policies** → search `AWSLoadBalancerControllerIAMPolicy` → **Delete**.

6. **Delete CloudFormation stack (if you used eksctl):**
    - **CloudFormation** → Stacks → delete any stack named `eksctl-livekit-cluster-*`

7. **Delete VPC:**
    - Follow the cleanup steps in [CREATE_VPC.md](./CREATE_VPC.md).

## CLI Cleanup

```bash
# 1. Delete ALB Controller
helm uninstall aws-load-balancer-controller -n kube-system

# 2. Delete service account
kubectl delete sa aws-load-balancer-controller -n kube-system

# 3. Delete Node Group
aws eks delete-nodegroup \
  --cluster-name livekit-cluster \
  --nodegroup-name livekit-nodes
echo "Waiting for node group deletion..."
aws eks wait nodegroup-deleted \
  --cluster-name livekit-cluster \
  --nodegroup-name livekit-nodes

# 4. Delete EKS Cluster
aws eks delete-cluster --name livekit-cluster
echo "Waiting for cluster deletion..."
aws eks wait cluster-deleted --name livekit-cluster

# 5. Delete OIDC Provider
OIDC_ARN=$(aws iam list-open-id-connect-providers \
  --query "OpenIDConnectProviderList[?contains(Arn,'oidc.eks.ap-south-1')].Arn" \
  --output text)
aws iam delete-open-id-connect-provider \
  --open-id-connect-provider-arn $OIDC_ARN

# 6. Delete IAM Roles and Policies
# Cluster role
aws iam detach-role-policy \
  --role-name livekit-eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
aws iam delete-role --role-name livekit-eks-cluster-role

# Node role
for P in AmazonEKSWorkerNodePolicy AmazonEKS_CNI_Policy AmazonEC2ContainerRegistryReadOnly; do
  aws iam detach-role-policy \
    --role-name livekit-eks-node-role \
    --policy-arn arn:aws:iam::aws:policy/$P
done
aws iam delete-role --role-name livekit-eks-node-role

# ALB Controller role
aws iam detach-role-policy \
  --role-name livekit-alb-controller-role \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy
aws iam delete-role --role-name livekit-alb-controller-role

# ALB Controller policy
aws iam delete-policy \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy

# 7. Clean up temp files
rm -f eks-cluster-trust-policy.json eks-node-trust-policy.json \
      alb-iam-policy.json alb-trust-policy.json

echo "EKS cleanup complete!"
echo "Now delete the VPC using the cleanup steps in CREATE_VPC.md"
```

---
---

# Quick Reference

| Component | Name | Purpose |
|---|---|---|
| EKS Cluster | `livekit-cluster` | Managed Kubernetes control plane |
| Node Group | `livekit-nodes` | EC2 workers (t3.medium, private subnets) |
| Cluster Role | `livekit-eks-cluster-role` | Allows EKS to manage AWS resources |
| Node Role | `livekit-eks-node-role` | Allows nodes to pull images, join cluster |
| OIDC Provider | auto-created | Enables IAM Roles for Service Accounts (IRSA) |
| ALB Controller | `aws-load-balancer-controller` | Watches Ingress → creates ALBs automatically |
| ALB Role | `livekit-alb-controller-role` | IAM permissions for ALB controller (via IRSA) |
| ALB Policy | `AWSLoadBalancerControllerIAMPolicy` | Permissions to create/manage ALBs, Target Groups |