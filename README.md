# Falcon MCP CodeBuild

This repository contains AWS CloudFormation templates for deploying a CodeBuild project that builds Docker images from the [CrowdStrike Falcon MCP Server](https://github.com/CrowdStrike/falcon-mcp) source repository.

## Files

- `template-fixed.yaml` - **THE ONLY TEMPLATE TO USE** - Simplified CloudFormation template that creates:
  - AWS CodeBuild project that uses the existing Dockerfile from CrowdStrike's repository
  - IAM role with necessary permissions
  - CloudWatch log group for build logs
  - ECR repository for Docker images

## Parameters

- `Environment` (String): Environment name (dev, staging, prod). Default: dev
- `ProjectName` (String): Name of the CodeBuild project and ECR repository. Default: falcon-mcp
- `ImageTag` (String): Docker image tag to use for the built image. Default: latest
- `GitHubRepoUrl` (String): GitHub repository URL. Default: https://github.com/CrowdStrike/falcon-mcp

## Quick Deploy

```bash
aws cloudformation deploy \
  --template-file template-fixed.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

**Note**: The template defaults to the official CrowdStrike repository. You can override this to use your own fork if needed.

## What It Does

The template creates a simple and efficient CI/CD pipeline:

1. **ECR Repository**: Automatically created with lifecycle policies (keeps last 10 images)
2. **CodeBuild Project**: Configured for GitHub with webhook triggers on push to main branch
3. **Simplified Build Process**:
   - Uses the existing production-ready Dockerfile from CrowdStrike's repository
   - Builds Docker image directly from source
   - Pushes to ECR with dual tagging (latest + commit hash)
   - Generates deployment artifacts

### Benefits of This Approach
- **Simple & Reliable**: Uses the official Dockerfile maintained by CrowdStrike
- **Always Up-to-Date**: Benefits from improvements made to the upstream repository
- **Fast Builds**: Minimal build steps focus only on containerization
- **Production Ready**: Uses the same build process as the official images

### Manual Build Trigger
```bash
aws codebuild start-build --project-name falcon-mcp-dev
```

### ECR Repository Access
```bash
# Login to ECR
aws ecr get-login-password --region your-region | docker login --username AWS --password-stdin your-account-id.dkr.ecr.your-region.amazonaws.com

# Pull the image
docker pull your-account-id.dkr.ecr.your-region.amazonaws.com/falcon-mcp:latest
```

## Architecture

The CloudFormation template creates:
1. **ECR Repository**: For storing Docker images with automatic lifecycle management
2. **CodeBuild Project**: With embedded BuildSpec and Dockerfile for complete automation
3. **IAM Role**: With ECR permissions for seamless operation
4. **CloudWatch Logs**: For build monitoring and troubleshooting
5. **GitHub Webhooks**: Automatic builds triggered on GitHub push to main branch

**Self-Contained & Simple**: Everything is embedded in the template - no external files or complex configurations needed.

## Build Process

Streamlined automated pipeline that runs on every GitHub push to main:

1. **Pre-build Phase**: 
   - Login to Amazon ECR
   - Set up image tagging with commit hash
2. **Build Phase**:
   - Build Docker image using the existing Dockerfile from CrowdStrike repository
   - Tag images for ECR (both latest and commit-specific tags)
3. **Post-build Phase**:
   - Push images to ECR
   - Generate `imagedefinitions.json` for deployments

## Environment Variables

The build environment includes standard CodeBuild variables:
- `AWS_DEFAULT_REGION` - Current AWS region
- `AWS_ACCOUNT_ID` - Current AWS account ID
- `IMAGE_REPO_NAME` - ECR repository name
- `IMAGE_TAG` - Docker image tag (default: latest)

## Caching

Docker layer caching is enabled to speed up subsequent builds by reusing unchanged layers.
