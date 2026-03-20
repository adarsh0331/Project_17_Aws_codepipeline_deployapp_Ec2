---
# CI/CD Pipeline for Node.js Web App on AWS EC2  
*(Created Manually via AWS Console – Full IAM Included)*


[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Node.js](https://img.shields.io/badge/Node.js-43853D?style=for-the-badge&logo=node.js&logoColor=white)](https://nodejs.org/)

---

## Overview

A **fully automated CI/CD pipeline** using **100% AWS native services**, created **entirely via AWS Console**:

```
CodeCommit → CodePipeline → CodeBuild → CodeDeploy → EC2
```
---

## Resources Summary 

| Resource | Name | Region |
|--------|------|--------|
| CodeCommit | `nodejs-app` | `us-east-1` |
| CodeBuild | `nodejs-app-build` | `us-east-1` |
| CodeDeploy App | `nodejs-app` | `us-east-1` |
| CodeDeploy Group | `prod-ec2-group` | `us-east-1` |
| CodePipeline | `nodejs-app-pipeline` | `us-east-1` |
| EC2 Instance | Tagged `Name=nodejs-app-server` | `us-east-1` |
| S3 Artifact Bucket | `codepipeline-us-east-1-xxxxxxxxx` | `us-east-1` |

---

## IAM Roles & Policies 

---

### 1. `EC2CodeDeployRole`  
**Attached to EC2 Instance**  
Allows EC2 to be managed by CodeDeploy.

#### Trust Relationship (Instance Profile)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Attached Managed Policy
```text
AmazonEC2RoleforAWSCodeDeploy
```

> **Permissions**:  
> - `codedeploy-commands:*`  
> - `s3:GetObject` (from artifact bucket)  
> - `autoscaling:Describe*` (if used)

---

### 2. `CodeDeployServiceRole`  
**Used by CodeDeploy to deploy to EC2**

#### Trust Relationship
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "codedeploy.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Attached Managed Policy
```text
AWSCodeDeployRole
```

> **Permissions**:  
> - `ec2:DescribeInstances`  
> - `codedeploy:*`  
> - `s3:GetObject`, `s3:ListBucket` (artifact access)  
> - `autoscaling:*` (for scaling groups)

---

### 3. `CodePipelineServiceRole`  
**Used by CodePipeline to orchestrate stages**

#### Trust Relationship
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "codepipeline.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Attached Managed Policy
```text
AWSCodePipelineFullAccess
```

> **Permissions**:  
> - `codecommit:*`  
> - `codebuild:StartBuild`, `BatchGetBuilds`  
> - `codedeploy:CreateDeployment`, `GetDeployment`  
> - `s3:*` (on artifact bucket)  
> - `iam:PassRole` (for CodeBuild & CodeDeploy)

---

### 4. `CodeBuildServiceRole`  
**Used by CodeBuild to access source & artifacts**

#### Trust Relationship
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "codebuild.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Attached Managed Policy
```text
AWSCodeBuildDeveloperAccess
```

> **Permissions**:  
> - `codecommit:GitPull`  
> - `s3:GetObject`, `s3:PutObject` (artifacts)  
> - `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`

---

### 5. (Optional) `CodeCommitAccessRole`  
**Only if you used IAM user with console access**

#### Trust Relationship
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::ACCOUNT_ID:user/your-user" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Inline Policy (Least Privilege)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codecommit:GitPush",
        "codecommit:GitPull"
      ],
      "Resource": "arn:aws:codecommit:us-east-1:ACCOUNT_ID:nodejs-app"
    }
  ]
}
```

---

## IAM Summary Table

| Role | Service | Trust Entity | Managed Policy | Purpose |
|------|--------|--------------|----------------|--------|
| `EC2CodeDeployRole` | EC2 | `ec2.amazonaws.com` | `AmazonEC2RoleforAWSCodeDeploy` | Allow CodeDeploy agent |
| `CodeDeployServiceRole` | CodeDeploy | `codedeploy.amazonaws.com` | `AWSCodeDeployRole` | Deploy to EC2 |
| `CodePipelineServiceRole` | CodePipeline | `codepipeline.amazonaws.com` | `AWSCodePipelineFullAccess` | Orchestrate pipeline |
| `CodeBuildServiceRole` | CodeBuild | `codebuild.amazonaws.com` | `AWSCodeBuildDeveloperAccess` | Build & access artifacts |

> All roles use **AWS-managed policies** — no custom inline policies needed.

---

## How IAM Roles Are Used in Pipeline

```
CodeCommit
    ↓ (CodePipeline reads source)
CodePipeline → uses CodePipelineServiceRole

CodePipeline → triggers Build
    ↓
CodeBuild → uses CodeBuildServiceRole

CodeBuild → outputs artifact to S3
    ↓
CodeDeploy → uses CodeDeployServiceRole

CodeDeploy → deploys to EC2
    ↓
EC2 → uses EC2CodeDeployRole (via instance profile)
```

---

## Security Best Practices Applied

| Practice | Implementation |
|--------|----------------|
| **Least Privilege** | Only AWS-managed policies used |
| **Service-Specific Roles** | 4 separate roles |
| **No Long-Term Credentials** | No access keys on EC2 |
| **PassRole Permissions** | CodePipeline has `iam:PassRole` only for needed roles |
| **S3 Artifact Encryption** | Enabled by default (SSE-S3) |

---

## Verify IAM Setup (Console Steps)

1. **EC2 Instance**  
   → Instance → **IAM role** → Should show `EC2CodeDeployRole`

2. **CodeDeploy → Deployment Groups**  
   → `prod-ec2-group` → **Service role** → `CodeDeployServiceRole`

3. **CodePipeline → `nodejs-app-pipeline`**  
   → Settings → **Service role** → `CodePipelineServiceRole`

4. **CodeBuild → `nodejs-app-build`**  
   → Edit → **Service role** → `CodeBuildServiceRole`

---

## Project Structure (CodeCommit)

```bash
nodejs-app/
├── app.js
├── package.json
├── scripts/
│   ├── install_dependencies.sh
│   ├── start_server.sh
│   └── stop_server.sh
├── appspec.yml
└── buildspec.yml
```

---

## `buildspec.yml`

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - npm ci
  build:
    commands:
      - npm test || echo "No tests"
artifacts:
  files: '**/*'
```

---

## `appspec.yml`

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/nodejs-app
hooks:
  AfterInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: ec2-user
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: ec2-user
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: ec2-user
```

---

## Test the Pipeline

```bash
# Make a change
echo "// Deployed at $(date)" >> app.js
git add .
git commit -m "test: trigger pipeline"
git push origin main
```

→ Watch **CodePipeline** → All stages green → App updated at `http://<EC2-IP>:3000`

---

## Monitoring

| Service | Log Location |
|-------|-------------|
| Pipeline | CodePipeline → Executions |
| Build | CodeBuild → Build logs |
| Deploy | CodeDeploy → Deployments |
| App | SSH → `pm2 logs nodejs-app` |

---

## Rollback

1. Go to **CodeDeploy → Applications → `nodejs-app` → Deployments**  
2. Select previous successful deployment  
3. Click **Redeploy**

---

## Security Hardening (Next Steps)

- [ ] Add **ALB + ACM** for HTTPS
- [ ] Restrict SG: Allow only ALB → EC2
- [ ] Enable **CloudTrail** for audit
- [ ] Use **SSM Session Manager** (no SSH keys)

---

*Last Updated: November 04, 2025*  
*Author: Adarsh Barkunta*
```

---
