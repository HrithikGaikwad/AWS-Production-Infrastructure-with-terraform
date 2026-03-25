# AWS-Production-Infrastructure-with-terraform
A production-ready AWS infrastructure that is highly available, secure, automated, scalable, and cost-optimized.

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











┌─────────────────────────────────────────────────────────────────┐
│                           AWS Cloud                               │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    VPC: 10.0.0.0/16                        │ │
│  │                                                             │ │
│  │  ┌─────────────────────┐    ┌─────────────────────┐        │ │
│  │  │   AZ: us-east-1a    │    │   AZ: us-east-1b    │        │ │
│  │  │  ┌───────────────┐  │    │  ┌───────────────┐  │        │ │
│  │  │  │  Public       │  │    │  │  Public       │  │        │ │
│  │  │  │  10.0.1.0/24  │  │    │  │  10.0.2.0/24  │  │        │ │
│  │  │  │  ┌─────────┐  │  │    │  │  ┌─────────┐  │  │        │ │
│  │  │  │  │   ALB   │  │  │    │  │  │   ALB   │  │  │        │ │
│  │  │  │  │ (Active)│◄─┼──┼────┼──┼──►│ (Standby)│  │  │        │ │
│  │  │  │  └────┬────┘  │  │    │  │  └────┬────┘  │  │        │ │
│  │  │  └───────┼───────┘  │    │  └───────┼───────┘  │        │ │
│  │  │          │          │    │          │          │        │ │
│  │  │  ┌───────▼───────┐  │    │  ┌───────▼───────┐  │        │ │
│  │  │  │   Private     │  │    │  │   Private     │  │        │ │
│  │  │  │  10.0.3.0/24  │  │    │  │  10.0.4.0/24  │  │        │ │
│  │  │  │  ┌─────────┐  │  │    │  │  ┌─────────┐  │  │        │ │
│  │  │  │  │  EC2/  │  │  │    │  │  │  EC2/  │  │  │        │ │
│  │  │  │  │  ASG   │  │  │    │  │  │  ASG   │  │  │        │ │
│  │  │  │  │(Docker)│  │  │    │  │  │(Docker)│  │  │        │ │
│  │  │  │  └─────────┘  │  │    │  │  └─────────┘  │  │        │ │
│  │  │  └───────────────┘  │    │  └───────────────┘  │        │ │
│  │  └─────────────────────┘    └─────────────────────┘        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│           │                           │                          │
│           ▼                           ▼                          │
│  ┌─────────────────┐          ┌─────────────────┐                 │
│  │  NAT Gateway    │          │  Internet GW    │                 │
│  │  (Single AZ)    │          │                 │                 │
│  └────────┬────────┘          └────────┬───────┘                 │
│           │                            │                         │
└───────────┼────────────────────────────┼─────────────────────────┘
│                            │
▼                            ▼
┌─────────────┐                ┌─────────────┐
│  CloudWatch │                │    User     │
│  (Logs/Alarms)│              │   (HTTPS)   │
└─────────────┘                └─────────────┘
plain


