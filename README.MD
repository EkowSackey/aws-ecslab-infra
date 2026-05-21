# ECS CI/CD Architecture Diagram as Code

```mermaid
flowchart TB
  user[User Browser]
  github[GitHub App Repo]
  gha[GitHub Actions<br/>OIDC auth]
  ecr[Amazon ECR<br/>Private repository]
  eb[Amazon EventBridge]
  cp[Amazon CodePipeline]
  cd[Amazon CodeDeploy<br/>Blue/Green]
  s3art[S3 Artifact Bucket]

  subgraph AWS["AWS Region"]
    subgraph VPC["Custom Multi-AZ VPC"]
      subgraph PUB1["Public Subnet AZ1"]
        alb1[Application Load Balancer]
      end

      subgraph PUB2["Public Subnet AZ2"]
        alb2[ALB node]
      end

      subgraph PRIV1["Private Subnet AZ1"]
        ecs1[ECS Fargate Task Set]
      end

      subgraph PRIV2["Private Subnet AZ2"]
        ecs2[ECS Fargate Task Set]
      end

      tgBlue[Blue Target Group]
      tgGreen[Green Target Group]

      ecrApi[ECR API VPC Endpoint]
      ecrDkr[ECR DKR VPC Endpoint]
      logsEp[CloudWatch Logs VPC Endpoint]
      s3ep[S3 Gateway Endpoint]

      ecsCluster[ECS Cluster]
      ecsService[ECS Service<br/>Min 1 / Desired 1 / Max 4<br/>CPU autoscaling]
      taskDef[Task Definition]
      cwlogs[CloudWatch Logs]
    end
  end

  github --> gha
  gha -->|Build image| ecr
  gha -->|Upload appspec/taskdef bundle| s3art

  ecr -->|Image push event| eb
  eb --> cp
  s3art --> cp
  cp --> cd

  user --> alb1
  alb1 --> tgBlue
  alb1 -. blue/green traffic shift .-> tgGreen
  tgBlue --> ecs1
  tgGreen --> ecs2

  ecsCluster --> ecsService
  ecsService --> taskDef
  taskDef --> ecs1
  taskDef --> ecs2

  ecs1 --> cwlogs
  ecs2 --> cwlogs

  ecs1 --> ecrApi
  ecs1 --> ecrDkr
  ecs1 --> logsEp
  ecs1 --> s3ep

  ecs2 --> ecrApi
  ecs2 --> ecrDkr
  ecs2 --> logsEp
  ecs2 --> s3ep

  cd -->|Deploy green task set and shift traffic| ecsService
```

## Notes

- Public entry point: Application Load Balancer
- Workload: ECS Fargate tasks in private subnets
- Image source: Amazon ECR
- Build pipeline: GitHub Actions with OIDC
- Deployment trigger: EventBridge on ECR image push
- Deployment orchestration: CodePipeline
- Deployment strategy: CodeDeploy blue/green
- Private service access: ECR API endpoint, ECR DKR endpoint, CloudWatch Logs endpoint, S3 gateway endpoint