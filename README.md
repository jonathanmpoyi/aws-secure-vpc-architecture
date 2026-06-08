# Secure VPC Architecture — AWS Multi-AZ Deployment

A production-grade secure VPC architecture deployed on AWS across multiple Availability Zones with private subnet isolation, NAT Gateway for outbound internet access, and AWS Systems Manager Session Manager for secure instance access without SSH exposure.

---

## Architecture

```
VPC: 10.0.0.0/16 (ca-central-1)
│
├── Availability Zone 1
│     ├── Public Subnet (10.0.1.0/24)
│     │     └── NAT Gateway 1 (Elastic IP)
│     └── Private Subnet (10.0.3.0/24)
│           └── EC2 Instance 1
│
└── Availability Zone 2
      ├── Public Subnet (10.0.2.0/24)
      │     └── NAT Gateway 2 (Elastic IP)
      └── Private Subnet (10.0.4.0/24)
            └── EC2 Instance 2

VPC Endpoints (Interface — both private subnets):
- com.amazonaws.ca-central-1.ssm
- com.amazonaws.ca-central-1.ssmmessages
- com.amazonaws.ca-central-1.ec2messages

Access: AWS Systems Manager Session Manager (no SSH)
```

---

## Services Used

| Service | Purpose |
|---|---|
| Amazon VPC | Isolated private network with custom CIDR |
| Public Subnets | Host NAT Gateways for outbound internet access |
| Private Subnets | Host EC2 instances, no direct internet exposure |
| Internet Gateway | Outbound internet access for public subnets |
| NAT Gateway | Outbound internet access for private subnet instances |
| VPC Endpoints | Private connectivity to Systems Manager without internet |
| Security Groups | Instance and endpoint level traffic controls |
| IAM Role | Grants EC2 instances permission to use Session Manager |
| Systems Manager | Secure instance access without SSH or open ports |

---

## Security Decisions

**Private Subnet Deployment**
All EC2 instances are deployed in private subnets with no public IP addresses assigned. Instances are not directly reachable from the internet. Access is controlled and auditable through Systems Manager Session Manager only.

**No SSH — Session Manager Only**
No key pairs were created. Port 22 is not open on any security group. Instance access is managed entirely through AWS Systems Manager Session Manager, which requires no open inbound ports and logs every session to CloudWatch.

**Two NAT Gateways for High Availability**
One NAT Gateway is deployed per Availability Zone. Each private subnet routes outbound internet traffic to the NAT Gateway in its own AZ. If one AZ goes down, the other AZ continues operating independently with its own NAT Gateway. A single NAT Gateway would create a single point of failure.

**VPC Endpoints for Private Systems Manager Access**
Three interface VPC endpoints are deployed in both private subnets for Systems Manager connectivity. Without these endpoints, private instances with no internet access cannot reach Systems Manager. The endpoints create a private connection through AWS PrivateLink, keeping all management traffic on the AWS private network and never traversing the internet.

**Private DNS Enabled on Endpoints**
Private DNS is enabled on all three VPC endpoints. This allows EC2 instances to use the standard AWS service URLs which automatically resolve to the private endpoint IP addresses within the VPC. No special configuration is required on the instances.

**Least Privilege Security Groups**

*VPC Endpoint Security Group:*
- Inbound: HTTPS 443 from VPC CIDR 10.0.0.0/16
- Outbound: default allow all

*EC2 Instance Security Group:*
- Inbound: HTTPS 443 from VPC endpoint security group only
- Outbound: default allow all

Traffic is restricted to only what is necessary. Instances can only receive traffic from the VPC endpoints, not from any other source.

**IAM Role with Least Privilege**
EC2 instances are assigned an IAM role with only the AmazonSSMManagedInstanceCore managed policy attached. This grants the minimum permissions required for Session Manager functionality. No additional permissions are granted.

---

## Route Table Design

```
Public Route Table (shared across both public subnets):
- 10.0.0.0/16 → local
- 0.0.0.0/0   → Internet Gateway

Private Route Table AZ1 (Private Subnet AZ1 only):
- 10.0.0.0/16 → local
- 0.0.0.0/0   → NAT Gateway AZ1

Private Route Table AZ2 (Private Subnet AZ2 only):
- 10.0.0.0/16 → local
- 0.0.0.0/0   → NAT Gateway AZ2
```

Public subnets share one route table because they both point to the same Internet Gateway. Private subnets have separate route tables because each points to a different NAT Gateway in its own AZ.

---

## Prerequisites

- AWS account
- Basic understanding of VPC, subnets, and routing
- IAM permissions to create VPC resources, EC2 instances, and IAM roles

---

## Deployment Steps

1. Create VPC with CIDR 10.0.0.0/16 in ca-central-1
2. Enable DNS hostnames and DNS resolution on the VPC
3. Create 4 subnets across 2 AZs with /24 CIDR blocks
4. Create and attach an Internet Gateway to the VPC
5. Create a public route table with a route to the Internet Gateway, associate with both public subnets
6. Create 2 Elastic IPs and 2 NAT Gateways, one per public subnet
7. Create 2 private route tables, each with a route to the NAT Gateway in the same AZ
8. Associate each private route table to its corresponding private subnet
9. Create a security group for VPC endpoints allowing inbound HTTPS 443 from VPC CIDR
10. Create 3 interface VPC endpoints for SSM, SSMmessages, and EC2messages with private DNS enabled, associated to both private subnets, with the endpoint security group attached
11. Create a security group for EC2 instances allowing inbound HTTPS 443 from the endpoint security group only
12. Create an IAM role for EC2 with AmazonSSMManagedInstanceCore policy attached
13. Launch EC2 instances in each private subnet with the IAM role and instance security group attached, no key pair, no public IP
14. Verify connectivity via Systems Manager Session Manager

---

## Verification

Connected to both private instances via AWS Systems Manager Session Manager. Confirmed private IP addresses assigned from the correct subnet CIDR ranges. No SSH keys used. No inbound ports open on instances.

---

## Repository Structure

```
aws-secure-vpc-architecture/
└── README.md        # Architecture documentation and security decisions
```

---

## Author

**Joseph Nathan Mpoyi**
Cybersecurity Professional | Cloud Security Engineer (in progress)
[LinkedIn](https://www.linkedin.com/in/joseph-nathan-panzu-mpoyi-416757173) · [josephnathanmpoyi.com](https://josephnathanmpoyi.com)
