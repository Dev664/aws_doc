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
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
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
