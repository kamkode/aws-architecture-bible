# AWS Deployment Guide for Spring Boot Microservices with React and PostgreSQL

## Prerequisites

Before starting the deployment process, ensure you have:

1. AWS Account with appropriate permissions
2. AWS CLI installed and configured
3. Docker and Docker Compose installed
4. Maven installed for Spring Boot applications
5. Node.js and npm installed for React frontend
6. Git installed for version control

## Architecture Overview

The deployment will consist of the following components:
- React Frontend (AWS Amplify)
- Spring Boot Microservices (AWS ECS)
- PostgreSQL Database (AWS RDS)
- CI/CD Pipeline (AWS CodePipeline)

## Step-by-Step Deployment Guide

### 1. AWS Infrastructure Setup

#### A. Create VPC and Network Configuration
```bash
# Create VPC
terraform init
terraform apply -var-file="vpc.tfvars"
```

#### B. Set up RDS PostgreSQL Database
1. Go to AWS RDS Console
2. Create a PostgreSQL instance:
   - DB Instance Class: db.t3.small
   - Storage: 20GB
   - Multi-AZ: No (for development)
   - Publicly Accessible: Yes (temporarily)
   - Security Group: Allow inbound traffic from your ECS cluster

#### C. Configure ECS Cluster
1. Create ECS Cluster:
   - Launch Type: EC2 Linux + Networking
   - Instance Type: t3.medium
   - Instance Count: 2
   - Security Group: Allow HTTP/HTTPS from ALB

### 2. Spring Boot Microservices Deployment

#### A. Dockerize Microservices
```dockerfile
# Dockerfile for Spring Boot application
FROM openjdk:17-slim
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

#### B. Create ECS Task Definition
```json
{
    "family": "spring-service",
    "containerDefinitions": [
        {
            "name": "spring-app",
            "image": "your-ecr-repo:latest",
            "memory": 512,
            "portMappings": [
                {
                    "containerPort": 8080,
                    "hostPort": 8080
                }
            ],
            "environment": [
                {
                    "name": "DB_URL",
                    "value": "your-rds-endpoint"
                }
            ]
        }
    ]
}
```

### 3. React Frontend Deployment

#### A. Configure React App
```javascript
// .env.production
REACT_APP_API_URL=https://your-api-endpoint
```

#### B. Deploy to AWS Amplify
```bash
# Initialize Amplify
amplify init

# Add hosting
amplify add hosting

# Deploy
amplify publish
```

### 4. CI/CD Pipeline Setup

#### A. Create CodePipeline
1. Source: GitHub/Bitbucket
2. Build: AWS CodeBuild
3. Deploy: ECS and Amplify

#### B. CodeBuild Buildspec
```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      java: corretto8
      nodejs: 16
  build:
    commands:
      - mvn clean package
      - docker build -t $IMAGE_NAME .
      - docker tag $IMAGE_NAME:latest $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:latest
```

## Security Best Practices

1. Use IAM roles and policies with least privilege
2. Enable VPC Flow Logs
3. Use AWS Secrets Manager for sensitive credentials
4. Enable AWS CloudTrail for audit logging
5. Use AWS WAF for web application protection

## Monitoring and Logging

1. Enable AWS CloudWatch Logs
2. Set up CloudWatch Alarms
3. Use AWS X-Ray for distributed tracing
4. Enable AWS CloudTrail for audit logging

## Cost Optimization Tips

1. Use Spot Instances for ECS
2. Enable RDS Multi-AZ only for production
3. Use Auto Scaling for ECS services
4. Enable RDS Read Replicas if needed
5. Use AWS Lambda for periodic tasks

## Troubleshooting

1. Check CloudWatch Logs for application errors
2. Verify Security Group rules
3. Check ECS Task Definitions for errors
4. Verify RDS connectivity
5. Check Amplify deployment logs

## Maintenance

1. Regular security patches
2. Database backups
3. Monitor resource usage
4. Update dependencies regularly
5. Review security configurations

## Next Steps

1. Set up automated backups
2. Configure disaster recovery
3. Implement monitoring dashboards
4. Set up alert notifications
5. Document deployment procedures

## Infrastructure as Code (IaC) Setup

### Terraform Configuration for Complete Infrastructure

Create a `main.tf` file for your infrastructure:

```terraform
provider "aws" {
  region = "us-east-1"
}

# VPC and Networking
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "microservices-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = false
  
  tags = {
    Environment = "production"
    Project     = "spring-microservices"
  }
}

# RDS PostgreSQL
resource "aws_db_instance" "postgres" {
  allocated_storage    = 20
  storage_type         = "gp2"
  engine               = "postgres"
  engine_version       = "13.4"
  instance_class       = "db.t3.small"
  name                 = "microservices_db"
  username             = "dbadmin"
  password             = var.db_password
  parameter_group_name = "default.postgres13"
  db_subnet_group_name = aws_db_subnet_group.default.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  skip_final_snapshot  = true
  multi_az             = true
  backup_retention_period = 7
}
```

## Secrets Management

### Using AWS Secrets Manager for Credentials

1. Store database credentials:
```bash
aws secretsmanager create-secret \
    --name microservices/db-credentials \
    --description "RDS PostgreSQL credentials" \
    --secret-string '{"username":"dbadmin","password":"your-secure-password"}'
```

2. Configure Spring Boot to use Secrets Manager:
```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://${rds.endpoint}:5432/microservices_db
    username: ${aws-secretsmanager:microservices/db-credentials:username}
    password: ${aws-secretsmanager:microservices/db-credentials:password}

# Add AWS Secrets Manager dependency
# build.gradle
implementation 'io.awspring.cloud:spring-cloud-starter-aws-secrets-manager-config'
```

## Database Migration Strategy

### Using Flyway for Database Migrations

1. Add Flyway dependency to your Spring Boot services:
```gradle
implementation 'org.flywaydb:flyway-core'
```

2. Create migration scripts in `src/main/resources/db/migration`:
```sql
-- V1__init.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__add_roles.sql
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE user_roles (
    user_id INTEGER REFERENCES users(id),
    role_id INTEGER REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);
```

3. Configure Flyway in `application.yml`:
```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
```

## Service Mesh Implementation

### Using AWS App Mesh for Service-to-Service Communication

1. Create an App Mesh service mesh:
```bash
aws appmesh create-mesh --mesh-name microservices-mesh
```

2. Create virtual services for each microservice:
```bash
aws appmesh create-virtual-service \
    --mesh-name microservices-mesh \
    --virtual-service-name service1.local \
    --spec '{ 
      "provider": { 
        "virtualNode": { 
          "virtualNodeName": "service1-vn"
        }
      }
    }'
```

3. Update ECS task definitions to include Envoy proxy:
```json
{
  "containerDefinitions": [
    {
      "name": "envoy",
      "image": "840364872350.dkr.ecr.us-west-2.amazonaws.com/aws-appmesh-envoy:v1.15.1.0-prod",
      "essential": true,
      "environment": [
        {
          "name": "APPMESH_VIRTUAL_NODE_NAME",
          "value": "mesh/microservices-mesh/virtualNode/service1-vn"
        }
      ],
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -s http://localhost:9901/server_info | grep state | grep -q LIVE"
        ],
        "interval": 5,
        "timeout": 2,
        "retries": 3
      }
    },
    {
      "name": "spring-app",
      "image": "your-ecr-repo/service1:latest",
      "essential": true
    }
  ]
}
```

## Blue/Green Deployment Strategy

### Using AWS CodeDeploy for Blue/Green Deployments

1. Create a CodeDeploy application:
```bash
aws deploy create-application \
    --application-name spring-microservices \
    --compute-platform ECS
```

2. Create a deployment group:
```bash
aws deploy create-deployment-group \
    --application-name spring-microservices \
    --deployment-group-name production \
    --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
    --ecs-service-info "clusterName=microservices-cluster,serviceName=service1" \
    --load-balancer-info "targetGroupPairInfoList=[{targetGroupPair={targetGroup={name=service1-blue-tg},testTrafficRoute={listenerArns=[arn:aws:elasticloadbalancing:region:account:listener/app/load-balancer-name/load-balancer-id/listener-id]}}}]" \
    --service-role-arn arn:aws:iam::account:role/CodeDeployServiceRole
```

3. Create an AppSpec file for deployment:
```yaml
# appspec.yml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "spring-app"
          ContainerPort: 8080
        PlatformVersion: "LATEST"
```

## Cost Optimization Strategies

### Implementing Cost-Effective AWS Resources

1. Use Spot Instances for ECS tasks:
```terraform
resource "aws_ecs_capacity_provider" "spot" {
  name = "spot-capacity-provider"

  auto_scaling_group_provider {
    auto_scaling_group_arn = aws_autoscaling_group.spot.arn
    
    managed_scaling {
      maximum_scaling_step_size = 10
      minimum_scaling_step_size = 1
      status                    = "ENABLED"
      target_capacity           = 100
    }
  }
}
```

2. Configure Auto Scaling based on custom metrics:
```bash
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --scalable-dimension ecs:service:DesiredCount \
    --resource-id service/microservices-cluster/service1 \
    --policy-name cpu-tracking-scaling-policy \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
      "TargetValue": 70.0,
      "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
      },
      "ScaleOutCooldown": 60,
      "ScaleInCooldown": 60
    }'
```

3. Use S3 lifecycle policies for logs and backups:
```terraform
resource "aws_s3_bucket_lifecycle_configuration" "logs_lifecycle" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id = "log-rotation"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}
```

## Security Hardening

### Implementing AWS Security Best Practices

1. Enable AWS Config for compliance monitoring:
```bash
aws configservice put-configuration-recorder \
    --configuration-recorder name=default,roleARN=arn:aws:iam::account:role/ConfigRole \
    --recording-group allSupported=true,includeGlobalResources=true
```

2. Implement AWS WAF rules for API Gateway:
```terraform
resource "aws_wafv2_web_acl" "api_waf" {
  name        = "api-gateway-waf"
  description = "WAF for API Gateway"
  scope       = "REGIONAL"

  default_action {
    allow {}
  }

  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSetMetric"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ApiGatewayWafAcl"
    sampled_requests_enabled   = true
  }
}
```

3. Implement AWS Shield Advanced for DDoS protection:
```bash
aws shield create-protection \
    --name "api-gateway-protection" \
    --resource-arn "arn:aws:apigateway:region::/restapis/api-id/stages/prod"
```

## Observability Setup

### Comprehensive Monitoring and Logging

1. Set up distributed tracing with AWS X-Ray:
```java
// Add to Spring Boot application
@SpringBootApplication
@EnableXRay
public class MicroserviceApplication {
    public static void main(String[] args) {
        SpringApplication.run(MicroserviceApplication.class, args);
    }
}
```

2. Create CloudWatch dashboards for monitoring:
```bash
aws cloudwatch put-dashboard \
    --dashboard-name "MicroservicesOverview" \
    --dashboard-body '{
        "widgets": [
            {
                "type": "metric",
                "x": 0,
                "y": 0,
                "width": 12,
                "height": 6,
                "properties": {
                    "metrics": [
                        [ "AWS/ECS", "CPUUtilization", "ServiceName", "service1", "ClusterName", "microservices-cluster" ]
                    ],
                    "period": 300,
                    "stat": "Average",
                    "region": "us-east-1",
                    "title": "ECS CPU Utilization"
                }
            }
        ]
    }'
```

3. Set up CloudWatch Alarms for critical metrics:
```terraform
resource "aws_cloudwatch_metric_alarm" "service_errors" {
  alarm_name          = "service1-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "5XXError"
  namespace           = "AWS/ApiGateway"
  period              = "60"
  statistic           = "Sum"
  threshold           = "5"
  alarm_description   = "This alarm monitors for 5XX errors"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  dimensions = {
    ApiName = "microservices-api"
    Stage   = "prod"
  }
}
```

---

This guide provides a comprehensive overview of deploying a Spring Boot microservices architecture on AWS. Each section can be expanded based on specific requirements and scale needs.
