# AWS Platform Instructions

## General Principles
- **ALWAYS use Infrastructure as Code** (Terraform, CloudFormation, CDK)
- **ALWAYS follow AWS Well-Architected Framework** principles
- **ALWAYS enable CloudTrail** for audit logging
- **ALWAYS use IAM roles** instead of access keys where possible
- **NEVER hardcode credentials** in code or configuration
- **NEVER use root account** for day-to-day operations

---

# IAM Best Practices

## Principles
- **Least Privilege**: Grant only permissions needed for the task
- **Use Roles**: Prefer IAM roles over long-lived access keys
- **MFA**: Require MFA for console access and sensitive operations
- **Regular Audit**: Review permissions periodically

## IAM Policy Structure
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescriptiveName",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::bucket-name/prefix/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

## IAM Checklist
- [ ] No wildcard (*) permissions on sensitive resources
- [ ] Conditions used to restrict access scope
- [ ] Service control policies (SCPs) for organizational guardrails
- [ ] Permission boundaries for delegated administration
- [ ] Regular access review with IAM Access Analyzer

---

# Compute Services

## EC2 Best Practices
- Use latest generation instance types
- Enable detailed monitoring for production
- Use placement groups for low-latency requirements
- Implement auto-scaling for variable workloads
- Use Spot instances for fault-tolerant workloads

## Lambda Best Practices
- Set appropriate memory and timeout values
- Use environment variables for configuration
- Implement proper error handling and retries
- Use layers for shared dependencies
- Enable X-Ray tracing for observability

## ECS/EKS Best Practices
- Use Fargate for serverless containers when appropriate
- Implement proper health checks
- Use task/pod IAM roles (not instance roles)
- Enable container insights for monitoring
- Use AWS App Mesh for service mesh requirements

---

# Data Services

## S3 Best Practices
```hcl
# GOOD: Secure S3 bucket configuration
resource "aws_s3_bucket" "secure_bucket" {
  bucket = "my-secure-bucket"
}

resource "aws_s3_bucket_versioning" "versioning" {
  bucket = aws_s3_bucket.secure_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "encryption" {
  bucket = aws_s3_bucket.secure_bucket.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.bucket_key.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "block_public" {
  bucket = aws_s3_bucket.secure_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## RDS Best Practices
- Enable Multi-AZ for production databases
- Use encryption at rest (KMS)
- Enable automated backups with appropriate retention
- Use parameter groups for configuration management
- Enable Performance Insights for monitoring
- Use IAM database authentication when possible

## DynamoDB Best Practices
- Design for access patterns (single-table design when appropriate)
- Use on-demand capacity for variable workloads
- Enable point-in-time recovery
- Use DAX for read-heavy workloads
- Implement TTL for data lifecycle management

---

# Networking

## VPC Best Practices
```
VPC Design:
├── Public Subnets (per AZ)
│   ├── NAT Gateway
│   ├── Load Balancers
│   └── Bastion hosts (if needed)
├── Private Subnets (per AZ)
│   ├── Application servers
│   └── ECS/EKS nodes
└── Data Subnets (per AZ)
    ├── RDS instances
    └── ElastiCache
```

## Security Groups
- **NEVER allow 0.0.0.0/0** on SSH/RDP ports
- Use security group references instead of CIDR where possible
- Implement least privilege for ingress and egress
- Document the purpose of each rule

## VPC Endpoints
- Use VPC endpoints for AWS service access (S3, DynamoDB, etc.)
- Reduces data transfer costs
- Keeps traffic within AWS network
- Improves security posture

---

# Monitoring & Logging

## CloudWatch Best Practices
- Create dashboards for key metrics
- Set up alarms for critical thresholds
- Use metric math for derived metrics
- Implement log insights queries for analysis
- Use contributor insights for top-N analysis

## Required Logging
- [ ] CloudTrail enabled (all regions, all events)
- [ ] VPC Flow Logs enabled
- [ ] S3 access logging enabled
- [ ] Load balancer access logs enabled
- [ ] CloudWatch Logs for application logs

## Alerting Standards
```yaml
# Critical alerts (PagerDuty/immediate response)
- High CPU utilization (>90% for 5 min)
- Memory exhaustion
- Disk space critical (<10%)
- Application errors spike
- Security group changes

# Warning alerts (Slack/email)
- Elevated error rates
- Latency increase
- Cost anomalies
- Certificate expiration (<30 days)
```

---

# Cost Optimization

## Principles
- Right-size resources based on actual usage
- Use Reserved Instances/Savings Plans for predictable workloads
- Implement auto-scaling to match demand
- Use Spot instances for fault-tolerant workloads
- Clean up unused resources regularly

## Cost Checklist
- [ ] Cost allocation tags implemented
- [ ] Budget alerts configured
- [ ] Unused resources identified and removed
- [ ] Reserved Instance utilization reviewed
- [ ] Data transfer costs analyzed

---

# Security Services

## Required Security Services
- **AWS Config**: Configuration compliance monitoring
- **GuardDuty**: Threat detection
- **Security Hub**: Security posture management
- **IAM Access Analyzer**: Permission analysis
- **Macie**: Sensitive data discovery (for S3)

## KMS Best Practices
- Use customer-managed keys for sensitive data
- Implement key rotation
- Define key policies with least privilege
- Use grants for temporary access
- Log all key usage via CloudTrail

---

# Disaster Recovery

## DR Strategies
| Strategy | RTO | RPO | Cost |
|----------|-----|-----|------|
| Backup & Restore | Hours | Hours | $ |
| Pilot Light | 10s of min | Minutes | $$ |
| Warm Standby | Minutes | Seconds | $$$ |
| Multi-Site Active | Near zero | Near zero | $$$$ |

## DR Checklist
- [ ] Backup strategy documented and tested
- [ ] Cross-region replication configured
- [ ] Recovery procedures documented
- [ ] DR drills scheduled regularly
- [ ] RPO/RTO requirements met
