# AWS EKS Lab: PuTTY Bastion, EKS, and Nginx

This lab uses a bastion EC2 instance created manually in the AWS Console. You connect to the bastion using **PuTTY** from Windows. After connecting, run AWS CLI, `kubectl`, and `eksctl` commands **on the bastion**, not from your Windows/WSL terminal.

The lab uses the existing default VPC and its default networking resources where suitable, creates the EKS control plane without a node group, validates it, adds two worker nodes, deploys Nginx without a Load Balancer, and cleans up the resources.

## Lab configuration

- **AWS region:** `us-east-1` (N. Virginia)
- **Bastion:** Amazon Linux 2023, created manually in the AWS Console
- **Connection:** PuTTY from Windows
- **VPC/networking:** Existing default VPC, verified before use
- **EKS cluster:** `DB13ekscluster`
- **Managed node group:** `DB13workers`, 2 × `t3.micro`
- **Application:** Nginx, 2 replicas
- **Service:** `ClusterIP` — no AWS Load Balancer for the Nginx Service

> **Cost warning:** EKS control planes and EC2 instances incur charges while running. EBS volumes, public IPv4 addresses, NAT Gateways, Load Balancers, snapshots, and other resources may also incur charges. Clean up when finished.

## Architecture

```text
Windows desktop
  └── PuTTY
       |
       | SSH using existing private key
       v
Bastion EC2 (Amazon Linux 2023)
  ├── AWS CLI
  ├── eksctl
  └── kubectl
       |
       v
Amazon EKS: DB13ekscluster
  └── Managed node group: DB13workers (2 × t3.micro)
       └── Nginx Deployment (2 replicas)
            └── ClusterIP Service (no AWS Load Balancer)
```

## Security notes

- Do not put AWS credentials or private keys in this README, Git, or screenshots.
- The AWS access key previously pasted into chat is exposed. Deactivate it and create a replacement before using CLI credentials.
- Prefer an IAM instance profile attached to the bastion over long-lived IAM user access keys. The role must have the required permissions; do not grant broad administrator access without a deliberate decision.
- Restrict SSH (TCP/22) to your public IP `/32`. Do not open SSH to `0.0.0.0/0`.
- Default VPC, subnet, and Security Group settings must still be inspected. “Default” does not automatically mean that the subnet is public or the SSH rules are safe.
- `t3.micro` worker nodes have limited CPU and memory. Check quotas and capacity; they may be too small for some Kubernetes workloads.

## Step 1: Create the bastion manually in AWS Console

1. Open **EC2 → Instances → Launch instances**.
2. Name: `DB13-bastion`.
3. AMI: **Amazon Linux 2023**.
4. Instance type: `t3.micro` (or an instance type permitted by your account).
5. **Key pair:** choose your existing EC2 key pair.
6. **Network:** choose the existing default VPC and a subnet in that VPC.
7. Select the default Security Group only if its inbound rules are appropriate. If SSH is not restricted to your public IP, edit the rule or create a dedicated Security Group.
8. Allow inbound TCP port `22` only from **My IP** or your exact public IP with `/32`.
9. Assign a public IPv4 address if you need to connect directly using PuTTY over the internet. The subnet must have a route to an Internet Gateway.
10. Launch the instance and wait for **Running** and successful status checks.

Record the bastion's public IP, instance ID, VPC ID, subnet ID, and Security Group ID.

## Step 2: Configure PuTTY on Windows

PuTTY generally uses a `.ppk` private key. If your existing key is in `.ppk` format, use it. If you have a `.pem` file, use **PuTTYgen** to load the PEM key and save a `.ppk` copy, following your organization's key-handling policy.

1. Open PuTTY.
2. In **Session**, set **Host Name** to `ec2-user@BASTION_PUBLIC_IP` and port `22`.
3. Under **Connection → SSH → Auth → Credentials** (menu wording may vary by PuTTY version), browse to the `.ppk` private key.
4. Return to **Session**, enter a saved-session name such as `DB13-bastion`, and select **Save**.
5. Select **Open** and accept the host key only after checking that you are connecting to the expected host.
6. Log in as `ec2-user` if prompted.

If the connection fails, check the instance public IP, route to an Internet Gateway, SSH Security Group source IP, username, and private key.

## Step 3: Install AWS CLI on the bastion

All remaining setup commands run in the PuTTY SSH terminal unless stated otherwise.

Update/install prerequisites on Amazon Linux 2023:

```bash
sudo dnf update -y
sudo dnf install -y unzip wget curl tar gzip
```

Install AWS CLI v2:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

## Step 4: Configure AWS access on the bastion

**Recommended approach: IAM instance profile**

For a safer setup, create or use an IAM role for EC2, attach it to the bastion as an instance profile, and grant only the permissions required for the lab. EKS cluster creation also needs permission to create/use the necessary EKS, EC2, CloudFormation, and IAM resources. Worker-node IAM roles are separate from the bastion's role. Configure EKS access for the identity that creates the cluster.

If your training environment specifically requires access keys, create a new key for a suitable IAM user, then run this command in PuTTY:

```bash
aws configure
```

Enter the new credentials interactively:

```text
AWS Access Key ID: <your-new-access-key-id>
AWS Secret Access Key: <your-new-secret-access-key>
Default region name: us-east-1
Default output format: json
```

Do not use a key previously exposed in chat. Do not paste credentials into commands or save them in this README.

Verify identity and configuration:

```bash
aws sts get-caller-identity
aws configure list
aws configure get region
```

Linux credentials normally live in `~/.aws/credentials` and configuration in `~/.aws/config`. Keep these files private. Do not share terminal screenshots containing account-sensitive information.

## Step 5: Install kubectl on the bastion

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

## Step 6: Install eksctl on the bastion

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
tar -xzf "eksctl_${PLATFORM}.tar.gz" -C /tmp
sudo install -m 0755 /tmp/eksctl /usr/local/bin/eksctl
eksctl version
```

Verify the AWS identity again:

```bash
aws sts get-caller-identity
```

## Step 7: Verify default VPC, subnets, route tables, and Security Groups

Run these checks in PuTTY on the bastion. These are inspection commands; they do not create networking resources.

Set the region:

```bash
export AWS_REGION=us-east-1
```

List EKS clusters first so you do not create a duplicate:

```bash
aws eks list-clusters --region "$AWS_REGION"
```

Find the default VPC:

```bash
aws ec2 describe-vpcs --region "$AWS_REGION" --filters Name=isDefault,Values=true --query 'Vpcs[].{VpcId:VpcId,CIDR:CidrBlock,State:State}' --output table
```

Set `VPC_ID` to the verified VPC ID returned above:

```bash
export VPC_ID=vpc-REPLACE_WITH_VERIFIED_DEFAULT_VPC_ID
```

Verify the VPC:

```bash
aws ec2 describe-vpcs --region "$AWS_REGION" --vpc-ids "$VPC_ID" --query 'Vpcs[].{VpcId:VpcId,IsDefault:IsDefault,CIDR:CidrBlock,State:State}' --output table
```

List its subnets:

```bash
aws ec2 describe-subnets --region "$AWS_REGION" --filters Name=vpc-id,Values="$VPC_ID" --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,PublicIP:MapPublicIpOnLaunch,State:State}' --output table
```

Inspect route tables:

```bash
aws ec2 describe-route-tables --region "$AWS_REGION" --filters Name=vpc-id,Values="$VPC_ID" --query 'RouteTables[].{RouteTableId:RouteTableId,Routes:Routes[*].{Destination:DestinationCidrBlock,Gateway:GatewayId,State:State},Subnets:Associations[*].SubnetId}' --output json
```

A public subnet should have an effective `0.0.0.0/0` route to an Internet Gateway (`igw-...`). The route can come from the VPC's main route table.

List Security Groups:

```bash
aws ec2 describe-security-groups --region "$AWS_REGION" --filters Name=vpc-id,Values="$VPC_ID" --query 'SecurityGroups[].{GroupId:GroupId,Name:GroupName,Description:Description}' --output table
```

Inspect the default Security Group:

```bash
aws ec2 describe-security-groups --region "$AWS_REGION" --filters Name=vpc-id,Values="$VPC_ID" Name=group-name,Values=default --query 'SecurityGroups[].{GroupId:GroupId,Ingress:IpPermissions,Egress:IpPermissionsEgress}' --output json
```

Check the bastion's networking:

```bash
aws ec2 describe-instances --region "$AWS_REGION" --filters Name=tag:Name,Values=DB13-bastion --query 'Reservations[].Instances[].{InstanceId:InstanceId,State:State.Name,VpcId:VpcId,SubnetId:SubnetId,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress,SecurityGroups:SecurityGroups[*].GroupId}' --output table
```

### EKS subnet checks

- Use subnets in at least two Availability Zones.
- Ensure the chosen subnets are in the verified VPC.
- Check available IP address capacity.
- Verify route tables and whether public or private subnet behavior is appropriate.
- Do not change or delete shared default networking resources just to make the lab work.

Select two verified subnets in different AZs and set their IDs below. The IDs from a previous environment may not match your account, so do not copy them without verification:

```bash
export SUBNET_A=subnet-REPLACE_WITH_VERIFIED_SUBNET_IN_AZ_A
export SUBNET_B=subnet-REPLACE_WITH_VERIFIED_SUBNET_IN_AZ_B
```

Validate both:

```bash
aws ec2 describe-subnets --region "$AWS_REGION" --subnet-ids "$SUBNET_A" "$SUBNET_B" --query 'Subnets[].{SubnetId:SubnetId,VpcId:VpcId,AZ:AvailabilityZone,CIDR:CidrBlock}' --output table
```

Only continue when both subnets exist, belong to the intended VPC, and are in different AZs.

## Step 8: Verify the EKS version and create the control plane only

List available EKS versions:

```bash
aws eks describe-cluster-versions --region "$AWS_REGION" --query 'clusters[].clusterVersion' --output table
```

Choose a version supported by EKS in your region. The command below uses `1.34` as an example; replace it if that version is not available.

Check existing clusters:

```bash
aws eks list-clusters --region "$AWS_REGION"
```

Create the EKS control plane without a node group. This is a single-line command. Replace the version and subnet variables as needed:

```bash
eksctl create cluster --name DB13ekscluster --region us-east-1 --version 1.34 --vpc-public-subnets "$SUBNET_A","$SUBNET_B" --without-nodegroup
```

Wait for the command to finish successfully. This is billable.

### Verify the control plane

```bash
aws eks describe-cluster --region "$AWS_REGION" --name DB13ekscluster --query 'cluster.{Name:name,Status:status,Version:version,VpcId:resourcesVpcConfig.vpcId,Subnets:resourcesVpcConfig.subnetIds}' --output json
```

Expected status: `ACTIVE`.

Configure `kubectl` on the bastion:

```bash
aws eks update-kubeconfig --region "$AWS_REGION" --name DB13ekscluster
kubectl get namespaces
kubectl get nodes
```

No worker nodes are expected yet because the cluster was created with `--without-nodegroup`.

## Step 9: Add two micro worker nodes

After the control plane is `ACTIVE`, create the managed node group:

```bash
eksctl create nodegroup --cluster DB13ekscluster --region us-east-1 --name DB13workers --node-type t3.micro --nodes 2 --nodes-min 2 --nodes-max 2 --managed
```

`t3.micro` has limited memory and CPU and may not be sufficient for all Kubernetes system pods and workloads. If nodes do not join or become `Ready`, inspect node-group status and events; consider a larger instance type if needed.

Validate:

```bash
eksctl get nodegroup --cluster DB13ekscluster --region "$AWS_REGION"
kubectl get nodes -o wide
kubectl get pods -A
```

Expected: node group `ACTIVE`, two worker nodes `Ready`, and system pods running.

## Step 10: Deploy Nginx without a Load Balancer

Create the manifest on the bastion:

```bash
mkdir -p ~/DB13eks-lab
cd ~/DB13eks-lab
nano nginx-app.yaml
```

Paste:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Apply and wait:

```bash
kubectl apply -f nginx-app.yaml
kubectl rollout status deployment/nginx
```

Validate:

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl get service nginx-service -o jsonpath='{.spec.type}{"\n"}'
```

Expected Service type: `ClusterIP`. This does not create an AWS Load Balancer.

### Test Nginx from the bastion

Run and leave this command active in one PuTTY session:

```bash
kubectl port-forward service/nginx-service 8080:80 --address 127.0.0.1
```

Because the port-forward binds only to the bastion's loopback interface, open a **second PuTTY session** to the same bastion and run:

```bash
curl http://127.0.0.1:8080
```

The response should contain the default Nginx HTML. Press `Ctrl+C` in the first session when finished. This does not expose Nginx publicly.

## Step 11: Final validation

Run from the bastion:

```bash
aws eks describe-cluster --region "$AWS_REGION" --name DB13ekscluster --query 'cluster.status' --output text
eksctl get nodegroup --cluster DB13ekscluster --region "$AWS_REGION"
kubectl get nodes
kubectl get pods -A
kubectl get deployment nginx
kubectl get service nginx-service
```

Expected:

| Component | Expected result |
|---|---|
| Bastion | Running |
| EKS control plane | `ACTIVE` |
| Node group | `ACTIVE` |
| Worker nodes | 2 `Ready` (subject to `t3.micro` capacity) |
| Nginx Deployment | 2 ready replicas |
| Nginx Service | `ClusterIP` |
| Nginx AWS Load Balancer | None |

## Step 12: Cleanup

Run cleanup from the bastion while it still has AWS credentials/role access and `kubectl` access. Delete the EKS cluster before terminating the bastion.

### 12.1 Delete Nginx

```bash
kubectl delete -f ~/DB13eks-lab/nginx-app.yaml
kubectl get deployment,service,pods
```

The built-in `kubernetes` Service remains while the cluster exists.

### 12.2 Delete the EKS cluster and managed node group

```bash
eksctl delete cluster --name DB13ekscluster --region us-east-1
```

Wait for completion and verify:

```bash
aws eks list-clusters --region us-east-1
```

`DB13ekscluster` should no longer appear. Inspect any deletion failure before proceeding.

### 12.3 Terminate the bastion manually

After cluster cleanup is complete:

1. Open **AWS Console → EC2 → Instances**.
2. Select `DB13-bastion`.
3. Choose **Instance state → Terminate instance**.
4. Confirm termination.

If you need to verify from AWS CLI before closing PuTTY, run:

```bash
aws ec2 describe-instances --region us-east-1 --filters Name=tag:Name,Values=DB13-bastion --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,PublicIP:PublicIpAddress}' --output table
```

### 12.4 Inspect remaining billable resources

Load Balancers:

```bash
aws elbv2 describe-load-balancers --region us-east-1 --query 'LoadBalancers[].{Name:LoadBalancerName,State:State.Code}' --output table
```

NAT Gateways:

```bash
aws ec2 describe-nat-gateways --region us-east-1 --filter Name=state,Values=pending,available --query 'NatGateways[].{ID:NatGatewayId,VPC:VpcId,State:State}' --output table
```

EBS volumes:

```bash
aws ec2 describe-volumes --region us-east-1 --query 'Volumes[].{ID:VolumeId,State:State,SizeGiB:Size,AttachedTo:Attachments[].InstanceId}' --output table
```

Elastic IP addresses:

```bash
aws ec2 describe-addresses --region us-east-1 --query 'Addresses[].{IP:PublicIp,AllocationId:AllocationId,InstanceId:InstanceId}' --output table
```

CloudFormation stacks:

```bash
aws cloudformation list-stacks --region us-east-1 --query "StackSummaries[?contains(StackName, 'eksctl-DB13ekscluster')].{Name:StackName,Status:StackStatus}" --output table
```

Delete only resources confirmed to belong to this lab and no longer needed. Do not delete the default VPC, shared subnets, default Security Group, or unrelated resources. `DELETE_COMPLETE` CloudFormation stacks are historical records; investigate `DELETE_FAILED` stacks before taking action.

### 12.5 Final verification

Before closing the bastion session, run:

```bash
aws eks list-clusters --region us-east-1
aws ec2 describe-instances --region us-east-1 --filters Name=tag:Name,Values=DB13-bastion --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}' --output table
```

The EKS cluster should be absent. After manually terminating the bastion, it should show as terminated or no longer appear in the filtered output.

> EKS and EC2 deletion removes the main lab compute resources, but EBS volumes, snapshots, NAT Gateways, Elastic IPs, Load Balancers, or resources in other regions may still incur charges. Review AWS Billing after cleanup.

## Troubleshooting

| Problem | First checks |
|---|---|
| PuTTY SSH fails | Check key format/path, `ec2-user`, public IP, route to Internet Gateway, and SSH source IP rule. |
| `AccessDenied` | Run `aws sts get-caller-identity`; verify IAM permissions or instance profile. |
| Cluster creation fails | Verify subnet IDs, AZs, VPC, supported Kubernetes version, IAM permissions, quotas, and capacity. |
| Worker nodes do not join | Check node-group status, EC2 capacity, networking, IAM node role, and system pods. |
| `kubectl` authentication fails | Run `aws eks update-kubeconfig --region us-east-1 --name DB13ekscluster` and verify the active identity. |
| Nginx does not respond | Check pods, events, logs, Service selector, and whether port-forward is still running. |
| Cleanup fails | Inspect the AWS error and dependencies; do not delete shared default networking resources blindly. |

## Security reminders

- Never commit AWS credentials or private key files to GitHub.
- Deactivate and replace credentials that have been exposed.
- Restrict SSH to trusted source IPs.
- Prefer an IAM instance profile over long-lived keys on the bastion.
- Keep Nginx as `ClusterIP` for this lab.
- Delete billable resources when finished.


---

# Part 2: CI/CD with GitHub, AWS CodePipeline, Amazon ECR, and Amazon EKS

This extension uses the **same `DB13ekscluster` EKS cluster and Nginx application** from Part 1. It does not create a second EKS cluster. GitHub stores the source; CodePipeline orchestrates the stages; CodeBuild builds and deploys; ECR stores the versioned image.

## CI/CD architecture

```text
Developer pushes to GitHub
          |
          v
AWS CodePipeline (GitHub via CodeStar Connection)
          |
          +--> Source stage
          |
          +--> Build stage: CodeBuild (privileged Docker build)
          |       |-- Build Nginx image from Dockerfile
          |       |-- Tag with Git commit SHA
          |       `-- Push image to Amazon ECR
          |
          `--> Deploy stage: CodeBuild
                  |-- Authenticate to the existing EKS cluster
                  |-- Apply Nginx Deployment + ClusterIP Service
                  |-- Set the ECR image and wait for rollout
                  `-- Verify Deployment, Pods, and Service
```

### Resources this part creates

- One Amazon ECR repository: `db13-nginx`
- One CodePipeline pipeline: `DB13-nginx-pipeline`
- Two CodeBuild projects: `DB13-nginx-build` and `DB13-nginx-deploy`
- IAM service roles/policies and an EKS access entry for the deploy role
- A GitHub repository and a CodeStar Connection

**Cost note:** CodePipeline, CodeBuild build minutes, ECR storage/data transfer, and the existing EKS/EC2 resources may incur charges. Avoid repeated builds, and complete the cleanup section when finished. A CodeStar Connection may require a one-time GitHub authorization.

## Prerequisites

Complete Part 1 first and verify all of the following from the PuTTY session on the bastion:

```bash
aws sts get-caller-identity
aws eks describe-cluster --region us-east-1 --name DB13ekscluster --query 'cluster.status' --output text
kubectl get nodes
kubectl get deployment nginx
kubectl get service nginx-service
```

The cluster should be `ACTIVE`, worker nodes should be `Ready`, and the current Nginx app should be available. If you previously deleted the cluster during cleanup, recreate and validate it using Part 1 before proceeding.

For this guide, use these names consistently:

```text
AWS region:       us-east-1
EKS cluster:      DB13ekscluster
ECR repository:   db13-nginx
GitHub repo:      db13-nginx-cicd (replace OWNER with your GitHub username/org)
CodePipeline:     DB13-nginx-pipeline
Build project:    DB13-nginx-build
Deploy project:   DB13-nginx-deploy
Kubernetes app:   nginx
Kubernetes svc:   nginx-service (ClusterIP)
```

## Step 1: Create the ECR repository

In the AWS Console, select **US East (N. Virginia) / `us-east-1`**.

1. Open **Amazon ECR → Private registry → Repositories → Create repository**.
2. Repository name: `db13-nginx`.
3. Visibility: **Private**.
4. Leave image tag mutability at its default or select **Mutable** for this teaching lab.
5. Enable scan-on-push if available and desired.
6. Create the repository and copy its **repository URI**, which looks like:

```text
ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/db13-nginx
```

Do not replace `ACCOUNT_ID` with a guessed value. Retrieve it from the account you are using:

```bash
aws sts get-caller-identity --query Account --output text
```

Set these environment variables in the PuTTY terminal for later commands (replace only if you intentionally use a different account/region):

```bash
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ECR_REPOSITORY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/db13-nginx"
export EKS_CLUSTER=DB13ekscluster
printf 'ECR image repository: %s\n' "$ECR_REPOSITORY"
```

## Step 2: Create the Git repository

Create an **empty private GitHub repository** named `db13-nginx-cicd` using the GitHub website. Do not initialize it with a README, license, or `.gitignore` because the next steps create the initial files from the bastion. Copy the repository HTTPS URL.

Install Git on the bastion if needed:

```bash
sudo dnf install -y git
 git --version
```

Create a project directory and the application files:

```bash
mkdir -p ~/db13-nginx-cicd/k8s
cd ~/db13-nginx-cicd
```

Create `Dockerfile`:

```dockerfile
FROM nginx:stable
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Create `index.html`:

```html
<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>DB13 Nginx CI/CD Lab</title></head>
<body>
  <h1>DB13 Nginx CI/CD deployment succeeded</h1>
  <p>Delivered through GitHub, AWS CodePipeline, CodeBuild, ECR, and EKS.</p>
  <p>Build version: replace-at-deploy-time</p>
</body>
</html>
```

Create `k8s/nginx-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: IMAGE_PLACEHOLDER
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
```

The `IMAGE_PLACEHOLDER` is replaced by the deploy buildspec with the exact ECR image built from the Git commit. The service remains **ClusterIP**; this lab does not create a public Load Balancer.

## Step 3: Add the CodeBuild build specification

Create `buildspec-build.yml` in the repository root:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - set -eu
      - ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - REGION="${AWS_DEFAULT_REGION}"
      - REPOSITORY_URI="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPOSITORY_NAME}"
      - IMAGE_TAG=$(printf '%s' "$CODEBUILD_RESOLVED_SOURCE_VERSION" | cut -c 1-12)
      - echo "Logging in to ECR in ${REGION}"
      - aws ecr get-login-password --region "$REGION" | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.${REGION}.amazonaws.com"
  build:
    commands:
      - echo "Building ${REPOSITORY_URI}:${IMAGE_TAG}"
      - docker build --pull -t "${REPOSITORY_URI}:${IMAGE_TAG}" .
      - docker tag "${REPOSITORY_URI}:${IMAGE_TAG}" "${REPOSITORY_URI}:latest"
  post_build:
    commands:
      - docker push "${REPOSITORY_URI}:${IMAGE_TAG}"
      - docker push "${REPOSITORY_URI}:latest"
      - printf '{"image":"%s"}\n' "${REPOSITORY_URI}:${IMAGE_TAG}" > imageDetail.json
      - echo "Published ${REPOSITORY_URI}:${IMAGE_TAG}"

artifacts:
  files:
    - imageDetail.json
```

The CodeBuild environment variable `ECR_REPOSITORY_NAME` must be set to `db13-nginx`. The tag is based on the Git commit, making deployments traceable. The `latest` tag is convenient for demonstrations, but deployments use the immutable commit-specific tag.

## Step 4: Add the CodeBuild deploy specification

Create `buildspec-deploy.yml` in the repository root:

```yaml
version: 0.2

phases:
  install:
    commands:
      - set -eu
      - curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
      - chmod +x kubectl
      - install -m 0755 kubectl /usr/local/bin/kubectl
  pre_build:
    commands:
      - ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - REGION="${AWS_DEFAULT_REGION}"
      - REPOSITORY_URI="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPOSITORY_NAME}"
      - IMAGE_TAG=$(printf '%s' "$CODEBUILD_RESOLVED_SOURCE_VERSION" | cut -c 1-12)
      - aws eks update-kubeconfig --region "$REGION" --name "$EKS_CLUSTER_NAME"
      - kubectl get nodes
  build:
    commands:
      - sed "s|IMAGE_PLACEHOLDER|${REPOSITORY_URI}:${IMAGE_TAG}|g" k8s/nginx-app.yaml > /tmp/nginx-app.yaml
      - kubectl apply -f /tmp/nginx-app.yaml
      - kubectl rollout status deployment/nginx --timeout=180s
      - kubectl get deployment nginx
      - kubectl get pods -l app=nginx -o wide
      - kubectl get service nginx-service
      - echo "Successfully deployed ${REPOSITORY_URI}:${IMAGE_TAG}"
```

**Important:** The `kubectl` download above follows the latest stable release. For a production pipeline, pin a reviewed `kubectl` version compatible with the EKS cluster and verify its checksum. The CodeBuild role needs outbound internet access to download it, or you should bake `kubectl` into a controlled custom build image.

## Step 5: Commit and push the source

From the project directory on the bastion:

```bash
cd ~/db13-nginx-cicd
git init
git branch -M main
git add Dockerfile index.html k8s/nginx-app.yaml buildspec-build.yml buildspec-deploy.yml
git status
git commit -m "Add Nginx EKS CI/CD lab"
git remote add origin https://github.com/OWNER/db13-nginx-cicd.git
git push -u origin main
```

Replace `OWNER` with your GitHub username or organization. If Git prompts for credentials, use the authentication method approved for your GitHub account (for HTTPS, a GitHub personal access token is used in place of a password). **Never paste a token into this README or commit it to Git.** For a long-term setup, prefer GitHub App/CodeStar Connection and short-lived authentication rather than storing a token on the bastion.

If `git remote add origin` says the remote already exists, inspect it with `git remote -v` and correct it with `git remote set-url origin https://github.com/OWNER/db13-nginx-cicd.git`.

## Step 6: Configure GitHub access for CodePipeline

1. Open **AWS Console → Developer Tools → Settings → Connections** (the exact menu label can vary).
2. Create a connection with provider **GitHub**.
3. Name it `DB13-GitHub-Connection` and choose **Connect to GitHub**.
4. Complete the GitHub authorization/installation flow and grant access to the `db13-nginx-cicd` repository.
5. Finish creating the connection and wait until its status is **Available**.
6. Record the connection ARN. You will select it in the CodePipeline source action.

Do not continue until the connection status is `Available` and the repository branch `main` exists.

## Step 7: Create IAM roles and EKS access for the deploy project

CodeBuild needs separate permissions for image publishing and cluster deployment. Prefer separate service roles so the build stage cannot administer the cluster unnecessarily.

### 7A. Build project role

When creating `DB13-nginx-build`, let CodeBuild create a service role, or create a dedicated role trusted by `codebuild.amazonaws.com`. Grant it:

- CloudWatch Logs permissions for its own build log group
- ECR authorization-token permission (`ecr:GetAuthorizationToken` on `*`)
- Push/pull permissions scoped to the `db13-nginx` repository: `ecr:BatchCheckLayerAvailability`, `ecr:CompleteLayerUpload`, `ecr:InitiateLayerUpload`, `ecr:PutImage`, `ecr:UploadLayerPart`, and any read actions required for build/pull

For a short teaching lab, `AmazonEC2ContainerRegistryPowerUser` is simpler but broader than a repository-scoped policy. Use least privilege outside a disposable lab.

### 7B. Deploy project role

Create or let CodeBuild create a separate service role for `DB13-nginx-deploy`. Grant it:

- CloudWatch Logs permissions for its own build log group
- `eks:DescribeCluster` for `DB13ekscluster` in `us-east-1`
- Permission to use the required EKS access-entry APIs if the role itself will manage them (otherwise configure access separately using an administrator role)

The role also needs **Kubernetes authorization** through an EKS access entry or a deliberately managed `aws-auth` mapping. IAM permission `eks:DescribeCluster` alone does not grant permission to create Deployments or Services inside Kubernetes.

### 7C. Register the deploy role with EKS

After the deploy CodeBuild project exists, open its details and copy its **service role ARN**. Use an administrator identity with permission to manage EKS access entries, or configure this in the AWS Console.

Check the cluster authentication mode:

```bash
aws eks describe-cluster --region us-east-1 --name DB13ekscluster --query 'cluster.accessConfig.authenticationMode' --output text
```

If access entries are supported but the mode is only `CONFIG_MAP`, change it to `API_AND_CONFIG_MAP` in **EKS → Cluster → Access**. Do not change this blindly in a shared/production cluster; confirm the cluster's existing access method first.

For a **disposable teaching lab only**, an administrator can register the deploy role and grant cluster-admin access as follows. Replace the role ARN with the exact CodeBuild deploy role ARN:

```bash
export DEPLOY_ROLE_ARN='arn:aws:iam::ACCOUNT_ID:role/EXACT_CODEBUILD_DEPLOY_ROLE'
aws eks create-access-entry --region us-east-1 --cluster-name DB13ekscluster --principal-arn "$DEPLOY_ROLE_ARN" --type STANDARD
aws eks associate-access-policy --region us-east-1 --cluster-name DB13ekscluster --principal-arn "$DEPLOY_ROLE_ARN" --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy --access-scope type=cluster
```

If the access entry already exists, do not recreate it; inspect it with `aws eks list-access-entries --region us-east-1 --cluster-name DB13ekscluster`. Cluster-admin access is intentionally broad and **not recommended for production**. For production, use a namespace-scoped Kubernetes Role/RoleBinding and a least-privilege EKS access policy.

## Step 8: Create the CodeBuild projects

Create both projects in **AWS Console → CodeBuild → Build projects → Create build project**, in `us-east-1`.

### Project A: `DB13-nginx-build`

- Source provider: **CodePipeline** (the source artifact comes from the pipeline).
- Environment image: Managed image, Linux, Standard runtime; select a currently supported standard image.
- Environment type: Linux.
- Privileged mode: **Enable**. Docker builds in a standard CodeBuild Linux environment require privileged mode unless using a different supported build architecture.
- Service role: the build project role from Step 7A.
- Buildspec: **Override buildspec name** with `buildspec-build.yml`.
- Environment variables:
  - `ECR_REPOSITORY_NAME` = `db13-nginx`
  - `AWS_DEFAULT_REGION` = `us-east-1` (if not already supplied by the environment)
- Logs: enable CloudWatch Logs.

### Project B: `DB13-nginx-deploy`

- Source provider: **CodePipeline**.
- Environment image: Managed Linux Standard image.
- Privileged mode: not required for the deploy stage.
- Service role: the deploy project role from Step 7B.
- Buildspec: **Override buildspec name** with `buildspec-deploy.yml`.
- Environment variables:
  - `ECR_REPOSITORY_NAME` = `db13-nginx`
  - `EKS_CLUSTER_NAME` = `DB13ekscluster`
  - `AWS_DEFAULT_REGION` = `us-east-1`
- Logs: enable CloudWatch Logs.

Both CodeBuild projects need network connectivity to the relevant AWS APIs. If they are configured inside a VPC, verify subnet routing, DNS, and endpoints/NAT for Git-independent AWS API calls and the `kubectl` download. For the simplest lab, use the default non-VPC CodeBuild environment; the EKS endpoint must be reachable from CodeBuild. If the EKS endpoint is private-only, configure CodeBuild networking inside the cluster VPC and the required routes/security-group rules.

## Step 9: Create the CodePipeline

Open **AWS Console → CodePipeline → Create pipeline** in `us-east-1`.

1. Pipeline name: `DB13-nginx-pipeline`.
2. Choose the standard new service role option or a dedicated pipeline role with the required permissions.
3. Source provider: **GitHub (via GitHub App)** / CodeStar Connection.
4. Select `DB13-GitHub-Connection`, repository `OWNER/db13-nginx-cicd`, and branch `main`.
5. Enable change detection so a push to `main` triggers the pipeline.
6. Build stage provider: **AWS CodeBuild**; select `DB13-nginx-build`.
7. Add a separate deploy stage provider: **AWS CodeBuild**; select `DB13-nginx-deploy`.
8. Review the stages and create the pipeline.

If the wizard only offers one build stage, create the pipeline first and use **Edit → Add stage** to add the deploy CodeBuild action after the build action. Both CodeBuild actions should receive the source artifact. The build stage also emits `imageDetail.json`, but this guide's deploy stage derives the image tag from the same source commit, so it does not depend on parsing that file.

## Step 10: Run and verify the pipeline

Start the pipeline from the console. Inspect each action and open its CodeBuild logs if a stage fails. Expected sequence: Source succeeds → Build pushes an image to ECR → Deploy applies the manifests and waits for rollout.

Verify ECR images from the bastion:

```bash
aws ecr describe-images --region us-east-1 --repository-name db13-nginx --query 'sort_by(imageDetails,&imagePushedAt)[].{Tags:imageTags,Pushed:imagePushedAt,Digest:imageDigest}' --output table
```

Verify the Kubernetes rollout:

```bash
kubectl rollout status deployment/nginx
kubectl get deployment nginx
kubectl get pods -l app=nginx -o wide
kubectl get service nginx-service
kubectl get deployment nginx -o jsonpath='{.spec.template.spec.containers[?(@.name=="nginx")].image}{"\n"}'
```

The image shown by the final command should be the commit-tagged image in your account's `db13-nginx` ECR repository. The service type must remain `ClusterIP`:

```bash
kubectl get service nginx-service -o jsonpath='{.spec.type}{"\n"}'
```

To test the page from the bastion, run this command in one PuTTY session and keep it running:

```bash
kubectl port-forward service/nginx-service 8080:80 --address 127.0.0.1
```

In a second PuTTY session, run:

```bash
curl -s http://127.0.0.1:8080
```

You should see the `DB13 Nginx CI/CD deployment succeeded` page. Stop port-forwarding with `Ctrl+C` when done.

## Step 11: Prove continuous delivery with a code change

Edit the page and commit a visible change:

```bash
cd ~/db13-nginx-cicd
sed -i 's/CI\/CD deployment succeeded/CI\/CD deployment succeeded - version 2/' index.html
git add index.html
git commit -m "Update Nginx landing page"
git push origin main
```

The GitHub push should automatically trigger `DB13-nginx-pipeline`. Wait for all stages to succeed, then repeat the port-forward and `curl` test. If the old page appears, verify that the pipeline deployed the latest commit SHA, inspect the image in ECR, and check `kubectl describe deployment nginx` and the CodeBuild deploy logs.

## Troubleshooting

- **GitHub source action fails:** confirm the connection is `Available`, the repository and `main` branch are correct, and the GitHub App has access to that repository.
- **Docker build fails:** verify privileged mode is enabled for `DB13-nginx-build` and the build role can push to the ECR repository.
- **ECR login/push fails:** verify region, account, repository name, and ECR permissions. The CodeBuild role—not the bastion's IAM identity—must have ECR push permissions.
- **`kubectl` says `Unauthorized` or `Forbidden`:** verify the exact CodeBuild deploy role ARN is registered in the cluster access entries and has the intended Kubernetes/EKS access policy.
- **`kubectl` times out:** verify the EKS API endpoint access mode, CodeBuild networking, routes, DNS, and endpoint security-group rules. A public endpoint should still be restricted appropriately for security; do not open it broadly without a reason.
- **Image pull fails:** confirm the worker nodes can reach ECR and its image layers, and that the image URI/tag exists. Worker nodes normally need the appropriate ECR read permissions in their node role.
- **Pipeline succeeds but page does not change:** confirm the commit-specific image tag changed and that the Deployment references that tag; do not rely on `latest` for rollout detection.
- **Two replicas are not Ready on `t3.micro`:** inspect `kubectl describe pod` and `kubectl get events --sort-by=.lastTimestamp`. `t3.micro` has very limited capacity; a larger node type may be necessary and will cost more.

## Step 12: Cleanup the CI/CD resources

If you want to keep the EKS lab but remove only the CI/CD stack:

1. In CodePipeline, delete `DB13-nginx-pipeline`.
2. Delete CodeBuild projects `DB13-nginx-build` and `DB13-nginx-deploy`.
3. Delete the GitHub connection if it is no longer used by another pipeline.
4. Delete the ECR repository `db13-nginx` only after confirming its images are no longer needed. In ECR, choose the repository and delete it; image storage otherwise remains.
5. Remove the EKS access entry for the deploy role if it was created solely for this lab. Do this only after confirming the exact role and that no other workload uses it:

```bash
aws eks delete-access-entry --region us-east-1 --cluster-name DB13ekscluster --principal-arn "$DEPLOY_ROLE_ARN"
```

6. Delete only IAM roles/policies created for this lab and no longer in use. Remove attached policies before deleting the roles.

To remove the entire lab, first delete the pipeline/CodeBuild/ECR/access-entry resources as above, then follow the Part 1 cleanup instructions to delete the Nginx app, EKS cluster/node group, and manually created bastion. Do not delete the default VPC or shared networking resources.

## Final expected result

- GitHub push triggers CodePipeline.
- CodeBuild builds an Nginx image and pushes a commit-tagged image to ECR.
- A separate CodeBuild deploy action authenticates to `DB13ekscluster` and updates the Kubernetes Deployment.
- Two Nginx replicas become Ready.
- `nginx-service` remains `ClusterIP`; no Load Balancer is created by this application.
- A second Git commit causes a new image build and deployment without manually running `kubectl apply` from the bastion.
