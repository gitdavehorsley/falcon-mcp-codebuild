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

## Quick Deploy

```bash
aws cloudformation deploy \
  --template-file template-fixed.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

**Note**: The template is hardcoded to build from the official CrowdStrike Falcon MCP repository at `https://github.com/CrowdStrike/falcon-mcp`.

## What It Does

The template creates a simple and efficient build system:

1. **ECR Repository**: Automatically created with lifecycle policies (keeps last 10 images)
2. **CodeBuild Project**: Configured for manual builds from the CrowdStrike GitHub repository
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

Trigger a build manually when you want to update your ECR image with the latest version from CrowdStrike:

```bash
aws codebuild start-build --project-name falcon-mcp-dev
```

**When to trigger builds:**
- When CrowdStrike releases a new version of Falcon MCP
- When you want to rebuild with the latest commits from the main branch
- For initial setup to get your first image

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
2. **CodeBuild Project**: With embedded BuildSpec for manual builds
3. **IAM Role**: With ECR permissions for seamless operation
4. **CloudWatch Logs**: For build monitoring and troubleshooting

**Self-Contained & Simple**: Everything is embedded in the template - no external files or complex configurations needed.

## Build Process

Streamlined build pipeline that runs when manually triggered:

1. **Pre-build Phase**: 
   - Enable Docker BuildKit for advanced Dockerfile features
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
- `DOCKER_BUILDKIT=1` - Enables Docker BuildKit for advanced Dockerfile features

## Caching

Docker layer caching is enabled to speed up subsequent builds by reusing unchanged layers.
