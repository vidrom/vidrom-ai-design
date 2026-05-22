# AWS Deployment Topology

```mermaid
flowchart TB
    Dev[Developer / CI] --> CDK[vidrom-cdk]
    Dev --> DeploySSM[deploy-server-ssm.sh]

    subgraph AWS
        Route53[Route 53]
        CF[CloudFront\nportal.vidrom.com]
        S3[S3 portal bucket]
        ApiGw[HTTP API Gateway]
        Lambda[API Lambda\nadmin + management]

        ALB[Application Load Balancer\nsignaling.vidrom.com]
        EC2[EC2 Signaling Host\nNode runtime + coturn]
        RDS[(PostgreSQL RDS\nprivate isolated subnets)]
        Secrets[Secrets Manager]
    end

    CDK --> Route53
    CDK --> CF
    CDK --> S3
    CDK --> ApiGw
    CDK --> Lambda
    CDK --> ALB
    CDK --> EC2
    CDK --> RDS
    CDK --> Secrets

    DeploySSM --> EC2

    Route53 --> CF
    Route53 --> ALB
    CF --> S3
    CF --> ApiGw
    ApiGw --> Lambda
    Lambda --> RDS
    ALB --> EC2
    EC2 --> RDS
    EC2 --> Secrets
    Lambda --> Secrets
```

## Deployment Boundaries

- CDK owns infrastructure, DNS, portal assets, and the Lambda API deployment.
- The EC2 signaling code is still deployed separately over SSM.
- RDS is private and shared by both the EC2 signaling service and the Lambda portal APIs.