# AWS Secure Architecture Blueprint

This document outlines a secure AWS architecture that follows security best practices and compliance requirements.

## Architecture Diagram

```
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          AWS CloudFront                             │
│                       (WAF Protection + SSL)                        │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           AWS Shield                                │
│                       (DDoS Protection)                             │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Application Load Balancer                     │
│                       (SSL Termination + WAF)                       │
└───────┬───────────────────────┬───────────────────────┬─────────────┘
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│  Auto Scaling │      │  Auto Scaling │      │  Auto Scaling │
│   Group (EC2) │      │   Group (EC2) │      │   Group (EC2) │
│ App Tier Zone A│      │ App Tier Zone B│      │ App Tier Zone C│
└───────┬───────┘      └───────┬───────┘      └───────┬───────┘
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────────────────────────────────────────────────────────┐
│                           Amazon RDS                              │
│                   (Multi-AZ, Encrypted at rest)                   │
└───────────────────────────────────────────────────────────────────┘
```

## Security Components

### Perimeter Security
- **AWS WAF**: Web Application Firewall to protect against common exploits
- **AWS Shield**: DDoS protection service
- **CloudFront**: Content delivery with SSL/TLS encryption and geo-restrictions
- **Network ACLs**: Stateless packet filtering

### Network Security
- **VPC Design**: Isolated networks with public and private subnets
- **Security Groups**: Stateful firewalls for EC2 instances
- **AWS Transit Gateway**: For secure VPC interconnections
- **VPC Flow Logs**: Network traffic monitoring

### Compute Security
- **EC2 Hardening**: Minimal OS installations with security patches
- **IMDSv2**: Instance metadata service requiring token authentication
- **Systems Manager**: For patch management and secure access
- **GuardDuty**: For threat detection

### Data Security
- **KMS**: For encryption key management
- **S3 Bucket Policies**: Least privilege access to objects
- **RDS Encryption**: Database encryption at rest and in transit
- **Secrets Manager**: For credential management

### Identity & Access
- **IAM**: Role-based access control with MFA
- **AWS Organizations**: For account segregation
- **AWS Control Tower**: For guardrails and compliance
- **AWS SSO**: For centralized access management

### Logging & Monitoring
- **CloudTrail**: API activity logging
- **CloudWatch**: Performance and security monitoring
- **AWS Config**: Configuration compliance
- **Security Hub**: Security posture management

## Implementation Guidelines

### Step 1: VPC Setup
```terraform
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "SecureVPC"
  }
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = "us-east-1${["a", "b", "c"][count.index]}"

  tags = {
    Name = "PrivateSubnet-${count.index + 1}"
  }
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 101}.0/24"
  availability_zone = "us-east-1${["a", "b", "c"][count.index]}"
  map_public_ip_on_launch = true

  tags = {
    Name = "PublicSubnet-${count.index + 1}"
  }
}

resource "aws_flow_log" "vpc_flow_log" {
  log_destination      = aws_cloudwatch_log_group.flow_log.arn
  log_destination_type = "cloud-watch-logs"
  traffic_type         = "ALL"
  vpc_id               = aws_vpc.main.id
}
```

### Step 2: Security Groups
```terraform
resource "aws_security_group" "alb_sg" {
  name        = "alb-security-group"
  description = "Security group for ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "app_sg" {
  name        = "app-security-group"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Step 3: IAM Roles
```terraform
resource "aws_iam_role" "ec2_role" {
  name = "ec2-secure-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ssm_policy" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
```

## Compliance Mapping

| Control | AWS Service/Feature | Implementation |
|---------|---------------------|----------------|
| Data Encryption | KMS, S3 Encryption | All data encrypted at rest and in transit |
| Access Control | IAM, Security Groups | Least privilege principle applied |
| Network Security | VPC, NACLs, WAF | Defense in depth with multiple boundaries |
| Logging & Monitoring | CloudTrail, CloudWatch | Comprehensive audit trail with alerting |
| Vulnerability Management | Inspector, SSM Patch Manager | Regular scanning and patching |

## Security Validation Checklist

- [ ] CIS AWS Foundations Benchmark compliance
- [ ] Penetration testing conducted 
- [ ] Data classification complete
- [ ] Disaster recovery plan tested
- [ ] Incident response plan documented
