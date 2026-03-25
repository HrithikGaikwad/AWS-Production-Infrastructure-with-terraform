# AWS-Production-Infrastructure-with-terraform

A production-ready AWS infrastructure that is highly available, secure, automated, scalable, and cost-optimized.



#Cost Optimization Strategies

Graviton2 (ARM64): 20% cheaper than x86 with better performance
Savings Plans: 1-year Compute Savings Plan saves ~20-30%
Scheduled Scaling: Scale to 1 instance nights/weekends (40% savings)
Spot Instances: For dev/test environments (up to 90% savings)
Reserved Instances: For predictable production workloads
Right-sizing: Start with t4g.micro, scale up based on metrics




#Monthly Cost Breakdown (us-east-1)


Service	Configuration	                  Monthly Cost
EC2 (t4g.micro)	2 instances, Graviton2	   ~$12.50
ALB	Application Load Balancer	           ~$16.43
Data Transfer	100 GB/month	           ~$9.00
CloudWatch	Logs, metrics, alarms	       ~$5.00
NAT Gateway	1 NAT, 100 GB processed	       ~$37.45
S3	ALB logs, 10 GB	                       ~$0.23
KMS	1 key	                               ~$1.00
VPC Flow Logs	10 GB ingested	           ~$5.00
Total		                               ~$86.61/month



Production Readiness Checklist

## Production Readiness Note

### Infrastructure ✅
- [x] Multi-AZ deployment (2 AZs)
- [x] Private subnets for compute
- [x] NAT Gateway for outbound connectivity
- [x] Encrypted EBS volumes (KMS)
- [x] HTTPS only (TLS 1.3)
- [x] Security groups with least privilege

### Security ✅
- [x] No public EC2 instances
- [x] S3 Block Public Access enabled
- [x] IAM roles (no static keys)
- [x] VPC Flow Logs enabled
- [x] IMDSv2 enforced
- [x] Secrets not in repository

### Observability ✅
- [x] CloudWatch Logs centralized
- [x] Custom metrics dashboard
- [x] CPU alarms configured
- [x] Health check alarms
- [x] VPC Flow Logs for audit

### Reliability ✅
- [x] Auto Scaling (min 2 instances)
- [x] Health checks on ALB
- [x] Rolling deployments
- [x] Multi-AZ redundancy

### Cost Management ✅
- [x] Budget alerts configured ($500/month)
- [x] Graviton2 instances selected
- [x] Scheduled scaling option
- [x] gp3 volumes (cheaper than gp2)


#DEPLOYMENT STEPS

Install tools
brew install terraform awscli

Configure AWS SSO
aws configure sso

Create state bucket (one-time)
aws s3 mb s3://terraform-state-saas-prod



#INITIALIZE AND DEPLOY

git clone <repo>
cd terraform-aws-saas/environments/production

terraform init
terraform plan -var="domain_name=yourdomain.com" -var="alert_email=ops@company.com"
terraform apply



#POST DEPLOYMENT

Update DNS to point to ALB DNS name
Verify SSL certificate validation
Test auto-scaling with load test
Review CloudWatch dashboard
This architecture provides enterprise-grade security, high availability, and cost optimization while maintaining simplicity for rapid deployment. The modular Terraform structure allows for easy extension as your SaaS grows.

      



## Architecture Diagram

```mermaid
flowchart TB
    subgraph AWS["AWS Cloud"]
        subgraph VPC["VPC (10.0.0.0/16)"]
            subgraph AZ1["Availability Zone 1a"]
                subgraph PUB1["Public Subnet (10.0.1.0/24)"]
                    ALB1["Application Load Balancer"]
                end
                subgraph PRIV1["Private Subnet (10.0.3.0/24)"]
                    ASG1["Auto Scaling Group<br/>EC2 Instances (t4g.micro)"]
                end
            end
            
            subgraph AZ2["Availability Zone 1b"]
                subgraph PUB2["Public Subnet (10.0.2.0/24)"]
                    ALB2["Application Load Balancer"]
                end
                subgraph PRIV2["Private Subnet (10.0.4.0/24)"]
                    ASG2["Auto Scaling Group<br/>EC2 Instances (t4g.micro)"]
                end
            end
        end
        
        IGW["Internet Gateway"]
        NAT["NAT Gateway<br/>(Cost Optimized: Single)"]
        KMS["KMS Key<br/>(Encryption)"]
        CW["CloudWatch<br/>(Logs, Metrics, Alarms)"]
        S3["S3 Bucket<br/>(ALB Logs)"]
    end
    
    USER["User/Client"]
    
    USER -->|HTTPS:443| ALB1
    USER -->|HTTPS:443| ALB2
    
    ALB1 -->|HTTP:8080| ASG1
    ALB2 -->|HTTP:8080| ASG2
    
    ALB1 -.->|Access Logs| S3
    ALB2 -.->|Access Logs| S3
    
    ASG1 -.->|Outbound via| NAT
    ASG2 -.->|Outbound via| NAT
    
    NAT --> IGW
    IGW -->|"Internet"| USER
    
    ASG1 -.->|Encrypted EBS| KMS
    ASG2 -.->|Encrypted EBS| KMS
    S3 -.->|Encrypted| KMS
    
    ASG1 -.->|Metrics & Logs| CW
    ASG2 -.->|Metrics & Logs| CW
    ALB1 -.->|Health Checks| CW
    ALB2 -.->|Health Checks| CW
    
    style AWS fill:#f4f4f4,stroke:#232f3e,stroke-width:2px
    style VPC fill:#e6f3ff,stroke:#232f3e,stroke-width:2px
    style AZ1 fill:#fff4e6,stroke:#ff9900,stroke-width:2px
    style AZ2 fill:#fff4e6,stroke:#ff9900,stroke-width:2px
    style PUB1 fill:#ffe6e6,stroke:#d32f2f,stroke-width:1px
    style PUB2 fill:#ffe6e6,stroke:#d32f2f,stroke-width:1px
    style PRIV1 fill:#e6ffe6,stroke:#388e3c,stroke-width:1px
    style PRIV2 fill:#e6ffe6,stroke:#388e3c,stroke-width:1px
    style IGW fill:#fff,stroke:#232f3e,stroke-width:2px
    style NAT fill:#fff,stroke:#232f3e,stroke-width:2px
    style USER fill:#e1f5fe,stroke:#0277bd,stroke-width:2px






