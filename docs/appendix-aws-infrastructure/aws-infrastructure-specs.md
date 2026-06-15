# Appendix: AWS Infrastructure Specifications

> This appendix documents the exact AWS resources provisioned for this Kubernetes cluster, along with generalized templates for adaptation to your own environment.

---

## Table of Contents

- [Exact Resources Deployed](#exact-resources-deployed)
- [Generalized Resource Templates](#generalized-resource-templates)
- [AWS Cost Estimation](#aws-cost-estimation)
- [Security Considerations](#security-considerations)
- [Adaptation Guide](#adaptation-guide)
- [Troubleshooting AWS-Specific Issues](#troubleshooting-aws-specific-issues)

---

## Exact Resources Deployed

### EC2 Instances

| Name | Instance ID | Type | AMI | Private IP | Public IP | AZ |
|------|------------|------|-----|------------|-----------|-----|
| jumpbox | i-0123456789abcdef0 | t3.micro | ami-0b75f821522bcff85 | <JUMPBOX_PRIVATE_IP> | <JUMPBOX_PUBLIC_IP> | us-east-1a |
| server | i-0fedcba9876543210 | t3.small | ami-0b75f821522bcff85 | <SERVER_PRIVATE_IP> | None | us-east-1a |
| node-1 | i-0a1b2c3d4e5f6789a | t3.small | ami-0b75f821522bcff85 | <NODE_1_PRIVATE_IP> | None | us-east-1a |
| node-2 | i-0b2c3d4e5f6789a1b | t3.small | ami-0b75f821522bcff85 | <NODE_2_PRIVATE_IP> | None | us-east-1a |

**AMI Details**:
- ID: `ami-0b75f821522bcff85`
- OS: Debian 12 (Bookworm)
- Architecture: x86_64
- Kernel: 6.12.74+deb13+1-cloud-amd64

### Security Groups

**Jump Server Security Group** (`sg-0053e3f4589ec3ad5`)

| Direction | Type | Protocol | Port Range | Source | Description |
|-----------|------|----------|------------|--------|-------------|
| Inbound | SSH | TCP | 22 | 0.0.0.0/0 | Admin SSH access |
| Outbound | All traffic | All | All | 0.0.0.0/0 | Full outbound |

**Control Plane Security Group** (`sg-0945b3734714ba7e2`)

| Direction | Type | Protocol | Port Range | Source | Description |
|-----------|------|----------|------------|--------|-------------|
| Inbound | SSH | TCP | 22 | sg-0053e3f4589ec3ad5 | SSH from jumpbox |
| Inbound | Custom TCP | TCP | 6443 | sg-0053e3f4589ec3ad5 | Kubernetes API from jumpbox |
| Inbound | All ICMP | ICMP | All | sg-0053e3f4589ec3ad5 | Ping from jumpbox |
| Inbound | All traffic | All | All | sg-009e0dcf3f5a70fa9 | All from worker nodes |
| Outbound | All traffic | All | All | sg-009e0dcf3f5a70fa9 | All to worker nodes |

**Worker Node Security Group** (`sg-009e0dcf3f5a70fa9`)

| Direction | Type | Protocol | Port Range | Source | Description |
|-----------|------|----------|------------|--------|-------------|
| Inbound | SSH | TCP | 22 | sg-0053e3f4589ec3ad5 | SSH from jumpbox |
| Inbound | HTTP | TCP | 80 | sg-0053e3f4589ec3ad5 | HTTP testing from jumpbox |
| Inbound | All ICMP | ICMP | All | sg-0053e3f4589ec3ad5 | Ping from jumpbox |
| Inbound | All traffic | All | All | sg-0945b3734714ba7e2 | All from control plane |
| Outbound | All traffic | All | All | sg-0945b3734714ba7e2 | All to control plane |

### VPC and Subnet

| Resource | ID | CIDR | Details |
|----------|-----|------|---------|
| VPC | vpc-0abc123def4567890 | 172.31.0.0/16 | Default VPC |
| Subnet | subnet-0123456789abcdef0 | 172.31.16.0/20 | us-east-1a |
| Internet Gateway | igw-0123456789abcdef0 | - | Attached to VPC |
| Route Table | rtb-0123456789abcdef0 | 0.0.0.0/0 -> igw | Main route table |

### IAM (Minimal)

| Resource | Details |
|----------|---------|
| Instance Profile | Default EC2 role with basic permissions |
| Key Pair | `<YOUR_KEY_PAIR>` (RSA 2048-bit, .pem format) |

---

## Generalized Resource Templates

Use these templates to recreate this infrastructure in your own AWS account.

### Terraform Template

```hcl
# Variables
variable "key_name" {
  default = "your-key-pair-name"
}

variable "ami_id" {
  # Debian 12 AMD64 in us-east-1
  default = "ami-0b75f821522bcff85"
}

# VPC (using default)
data "aws_vpc" "default" {
  default = true
}

data "aws_subnet" "default" {
  vpc_id            = data.aws_vpc.default.id
  availability_zone = "us-east-1a"
}

# Security Groups
resource "aws_security_group" "jump" {
  name_prefix = "k8s-jump-"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Restrict to your IP in production
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "control_plane" {
  name_prefix = "k8s-control-plane-"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.jump.id]
  }

  ingress {
    from_port       = 6443
    to_port         = 6443
    protocol        = "tcp"
    security_groups = [aws_security_group.jump.id]
  }

  ingress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.worker.id]
  }

  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.worker.id]
  }
}

resource "aws_security_group" "worker" {
  name_prefix = "k8s-worker-"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.jump.id]
  }

  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.jump.id]
  }

  ingress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.control_plane.id]
  }

  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.control_plane.id]
  }
}

# EC2 Instances
resource "aws_instance" "jumpbox" {
  ami                    = var.ami_id
  instance_type          = "t3.micro"
  key_name               = var.key_name
  subnet_id              = data.aws_subnet.default.id
  vpc_security_group_ids = [aws_security_group.jump.id]
  associate_public_ip_address = true

  tags = {
    Name = "jumpbox"
  }
}

resource "aws_instance" "server" {
  ami                    = var.ami_id
  instance_type          = "t3.small"
  key_name               = var.key_name
  subnet_id              = data.aws_subnet.default.id
  vpc_security_group_ids = [aws_security_group.control_plane.id]
  associate_public_ip_address = false

  tags = {
    Name = "server"
  }
}

resource "aws_instance" "node_1" {
  ami                    = var.ami_id
  instance_type          = "t3.small"
  key_name               = var.key_name
  subnet_id              = data.aws_subnet.default.id
  vpc_security_group_ids = [aws_security_group.worker.id]
  associate_public_ip_address = false

  tags = {
    Name = "node-1"
  }
}

resource "aws_instance" "node_2" {
  ami                    = var.ami_id
  instance_type          = "t3.small"
  key_name               = var.key_name
  subnet_id              = data.aws_subnet.default.id
  vpc_security_group_ids = [aws_security_group.worker.id]
  associate_public_ip_address = false

  tags = {
    Name = "node-2"
  }
}

# Outputs
output "jumpbox_public_ip" {
  value = aws_instance.jumpbox.public_ip
}

output "server_private_ip" {
  value = aws_instance.server.private_ip
}

output "node_1_private_ip" {
  value = aws_instance.node_1.private_ip
}

output "node_2_private_ip" {
  value = aws_instance.node_2.private_ip
}
```

### AWS CLI Commands

```bash
# Create security groups
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query 'Vpcs[0].VpcId' --output text)

# Jump security group
aws ec2 create-security-group \
  --group-name k8s-jump \
  --description "Jump server for KTHW" \
  --vpc-id $VPC_ID

# Control plane security group
aws ec2 create-security-group \
  --group-name k8s-control-plane \
  --description "Control plane for KTHW" \
  --vpc-id $VPC_ID

# Worker security group
aws ec2 create-security-group \
  --group-name k8s-worker \
  --description "Worker nodes for KTHW" \
  --vpc-id $VPC_ID

# Get security group IDs
JUMP_SG=$(aws ec2 describe-security-groups --group-names k8s-jump --query 'SecurityGroups[0].GroupId' --output text)
CP_SG=$(aws ec2 describe-security-groups --group-names k8s-control-plane --query 'SecurityGroups[0].GroupId' --output text)
WORKER_SG=$(aws ec2 describe-security-groups --group-names k8s-worker --query 'SecurityGroups[0].GroupId' --output text)

# Add rules (run these after all SGs are created)
# Jump inbound SSH
aws ec2 authorize-security-group-ingress \
  --group-id $JUMP_SG \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Control plane inbound from jump
aws ec2 authorize-security-group-ingress \
  --group-id $CP_SG \
  --protocol tcp --port 22 --source-group $JUMP_SG

aws ec2 authorize-security-group-ingress \
  --group-id $CP_SG \
  --protocol tcp --port 6443 --source-group $JUMP_SG

# Workers inbound from control plane
aws ec2 authorize-security-group-ingress \
  --group-id $WORKER_SG \
  --protocol tcp --port 22 --source-group $JUMP_SG

# Bidirectional between CP and workers
aws ec2 authorize-security-group-ingress \
  --group-id $CP_SG \
  --protocol all --source-group $WORKER_SG

aws ec2 authorize-security-group-ingress \
  --group-id $WORKER_SG \
  --protocol all --source-group $CP_SG

# Launch instances
SUBNET_ID=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID --query 'Subnets[0].SubnetId' --output text)
AMI_ID="ami-0b75f821522bcff85"  # Update for your region
KEY_NAME="your-key-pair"

# Jumpbox
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --key-name $KEY_NAME \
  --subnet-id $SUBNET_ID \
  --security-group-ids $JUMP_SG \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=jumpbox}]'

# Control plane
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.small \
  --key-name $KEY_NAME \
  --subnet-id $SUBNET_ID \
  --security-group-ids $CP_SG \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=server}]'

# Workers
for i in 1 2; do
  aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.small \
    --key-name $KEY_NAME \
    --subnet-id $SUBNET_ID \
    --security-group-ids $WORKER_SG \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=node-$i}]"
done
```

---

## AWS Cost Estimation

### On-Demand Pricing (us-east-1, approximate)

| Instance | Type | vCPU | RAM | Hourly Cost | Monthly Cost (730 hrs) |
|----------|------|------|-----|-------------|------------------------|
| jumpbox | t3.micro | 2 | 1 GB | $0.0104 | $7.59 |
| server | t3.small | 2 | 2 GB | $0.0208 | $15.18 |
| node-1 | t3.small | 2 | 2 GB | $0.0208 | $15.18 |
| node-2 | t3.small | 2 | 2 GB | $0.0208 | $15.18 |
| **Total** | | **8** | **7 GB** | **$0.0728** | **$53.13** |

### Cost Optimization Tips

1. **Spot Instances**: Save up to 90% on worker nodes (not recommended for control plane)
2. **Stop when not in use**: EC2 instances can be stopped to halt charges (only pay for storage)
3. **Use Graviton (ARM)**: t4g instances are ~20% cheaper than t3
4. **Reserved Instances**: For long-term learning, 1-year reserved saves ~40%

### Estimated Costs for Learning

| Scenario | Duration | Estimated Cost |
|----------|----------|----------------|
| Single session (8 hours) | 1 day | $0.58 |
| Weekend project | 2 days | $1.17 |
| Week-long deep dive | 7 days | $4.10 |
| Month-long study | 30 days | $17.57 |

---

## Security Considerations

### Current Setup (Educational)

| Aspect | Current | Risk Level |
|--------|---------|------------|
| SSH from internet | 0.0.0.0/0 | HIGH |
| No MFA | - | MEDIUM |
| Default VPC | - | LOW |
| No encryption in transit (etcd HTTP) | - | MEDIUM |
| No NetworkPolicies | - | MEDIUM |

### Production Hardening

| Measure | Implementation |
|---------|---------------|
| Restrict SSH | CIDR to your office/home IP only |
| Bastion host | Hardened jumpbox with MFA |
| Private subnets | Control plane and workers in private subnets |
| NAT Gateway | For outbound internet from private subnets |
| VPC Flow Logs | Monitor network traffic |
| AWS Systems Manager | Session Manager instead of SSH |
| etcd TLS | Enable peer and client TLS |
| Pod Security | Enable PodSecurityAdmission |
| Network Policies | Calico/ Cilium NetworkPolicies |
| Encryption | EBS encryption, Secrets encryption |
| Logging | CloudWatch or centralized logging |
| Backup | Automated etcd snapshots to S3 |

---

## Adaptation Guide

### Changing Regions

1. Find Debian 12 AMI for your region:
```bash
aws ec2 describe-images \
  --owners 136693071363 \
  --filters "Name=name,Values=debian-12-*" "Name=architecture,Values=x86_64" \
  --query 'Images[*].[ImageId,Name]' --output table
```

2. Update AMI ID in all templates

### Using Ubuntu Instead of Debian

| Change | From | To |
|--------|------|-----|
| AMI | Debian 12 | Ubuntu 22.04 LTS |
| User | `admin` | `ubuntu` |
| Package manager | `apt` | `apt` (same) |
| Systemd | Same | Same |

Ubuntu AMI example: `ami-0c7217cdde317cfec` (us-east-1)

### Using ARM (Graviton) Instances

| Change | From | To |
|--------|------|-----|
| Instance type | t3.* | t4g.* |
| Architecture | amd64 | arm64 |
| Binaries | linux/amd64 | linux/arm64 |

Download ARM binaries:
```bash
wget https://dl.k8s.io/v1.32.3/bin/linux/arm64/kubectl
```

### Scaling Workers

To add more worker nodes:
1. Launch new EC2 instance with worker security group
2. Add entry to `machines.txt`
3. Generate certificate (Step 2)
4. Generate kubeconfig (Step 3)
5. Follow Step 6 installation
6. Add pod CIDR route on all nodes

---

## Troubleshooting AWS-Specific Issues

### Issue: "Permission denied (publickey)"

**Cause**: Wrong key pair or wrong username
**Fix**: Ensure using correct `.pem` file and correct user (`admin` for Debian, `ubuntu` for Ubuntu, `ec2-user` for Amazon Linux)

### Issue: "Connection timed out"

**Cause**: Security group blocking traffic or instance in wrong subnet
**Fix**: Verify security group rules and subnet has route to internet gateway

### Issue: Worker nodes can't reach internet

**Cause**: No NAT gateway in private subnet
**Fix**: This is expected in our setup. Use jumpbox as proxy or add NAT gateway.

### Issue: "No space left on device"

**Cause**: Default EBS volume (8GB) is too small
**Fix**: Increase root volume size during launch or expand after launch:
```bash
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

### Issue: AZ mismatch

**Cause**: Instances launched in different AZs can't communicate via private IP
**Fix**: Ensure all instances are in the same subnet (and thus same AZ), or peer subnets across AZs.

---

## Resource Cleanup

When finished, clean up to avoid charges:

```bash
# Terminate instances
aws ec2 terminate-instances --instance-ids i-xxx i-yyy i-zzz

# Delete security groups (after instances are terminated)
aws ec2 delete-security-group --group-id sg-xxx
aws ec2 delete-security-group --group-id sg-yyy
aws ec2 delete-security-group --group-id sg-zzz
```

Or use Terraform:
```bash
terraform destroy
```

---

## Summary

This appendix provides:
- **Exact specifications** of the deployed AWS infrastructure
- **Generalized templates** (Terraform, AWS CLI) for recreation
- **Cost estimates** for budget planning
- **Security hardening** recommendations
- **Adaptation guide** for different regions, OSs, and architectures
- **Cleanup procedures** to avoid unexpected charges

Remember: This infrastructure is designed for learning. Apply security hardening before using any similar setup in production.
