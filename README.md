# Falcon MCP CodeBuild

This repository contains AWS CloudFormation templates and build specifications for deploying a CodeBuild project for the CrowdStrike Falcon MCP Server with flexible source control options.

## Files

- `template.yaml` - Main CloudFormation template that creates:
  - AWS CodeBuild project with embedded BuildSpec and Dockerfile
  - IAM role with necessary permissions
  - CloudWatch log group for build logs
  - Optional ECR repository for Docker images

## Parameters

- `Environment` (String): Environment name (dev, staging, prod). Default: dev
- `ProjectName` (String): Name of the CodeBuild project. Default: falcon-mcp
- `RepositoryUrl` (String): **Required** - CodeCommit repository URL
- `ImageTag` (String): Docker image tag to use for the built image. Default: latest
- `DockerfileContent` (String): Inline Dockerfile content for container builds (auto-populated)
- `BuildSpecContent` (String): Inline BuildSpec content for CodeBuild (auto-populated)

## Prerequisites

1. **Create a CodeCommit repository:**
   ```bash
   aws codecommit create-repository --repository-name falcon-mcp
   ```

2. **Get the repository URL:**
   ```bash
   aws codecommit get-repository --repository-name falcon-mcp --query 'repositoryMetadata.cloneUrlHttp'
   ```

## Quick Deploy

```bash
aws cloudformation deploy \
  --template-file template-fixed.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --parameter-overrides \
    RepositoryUrl=https://git-codecommit.region.amazonaws.com/v1/repos/falcon-mcp \
  --capabilities CAPABILITY_IAM
```

## What It Does

The template creates a complete CI/CD pipeline:

1. **ECR Repository**: Automatically created with lifecycle policies (keeps last 10 images)
2. **CodeBuild Project**: Configured for CodeCommit with webhook triggers on push to main branch
3. **Automated Build Process**:
   - Runs Python tests (ruff, mypy, black, pytest)
   - Builds Python package with uv
   - Builds Docker image using embedded multi-stage Dockerfile
   - Pushes to ECR with dual tagging (latest + commit hash)
   - Generates deployment artifacts

### Dockerfile Features
- **Multi-stage build** with uv for fast Python dependency management
- **Security hardened** (non-root user, minimal Alpine base)
- **Optimized caching** with bytecode compilation
- **Production ready** container image

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
3. **IAM Role**: With CodeCommit and ECR permissions for seamless operation
4. **CloudWatch Logs**: For build monitoring and troubleshooting
5. **Webhook Triggers**: Automatic builds on CodeCommit push to main branch

**Self-Contained & Simple**: Everything is embedded in the template - no external files or complex configurations needed.

## Build Process

Complete automated pipeline that runs on every CodeCommit push:

1. **Install Phase**: Sets up Python 3.13 and installs uv package manager
2. **Pre-build Phase**: Locks dependencies and prepares virtual environment
3. **Build Phase**:
   - Linting with ruff
   - Type checking with mypy
   - Code formatting check with black
   - Unit tests with pytest
4. **Post-build Phase**:
   - Creates Python distribution packages
   - Builds Docker image using embedded multi-stage Dockerfile
   - Pushes to ECR with dual tagging (latest + commit hash)
   - Generates `imagedefinitions.json` for deployments

## Environment Variables

The build environment is configured with:
- `UV_COMPILE_BYTECODE=1` - Enables Python bytecode compilation
- `UV_LINK_MODE=copy` - Copies dependencies instead of linking (required for CodeBuild)

## Caching

The buildspec includes caching for:
- uv cache directory (`$HOME/.cache/uv/**/*`)
- uv binary installations (`$HOME/.local/bin/**/*`)

This ensures faster subsequent builds by reusing downloaded packages and tools.
