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
- `SourceType` (String): Source type for the CodeBuild project (CODECOMMIT, S3, GITHUB). Default: S3
- `RepositoryUrl` (String): Git repository URL (leave empty to use S3 source). Default: ""
- `S3Bucket` (String): S3 bucket name for source code (required when SourceType is S3). Default: ""
- `S3Key` (String): S3 object key for source code (used when SourceType is S3). Default: source.zip
- `GitHubTokenSecretArn` (String): ARN of the AWS Secrets Manager secret containing the GitHub personal access token (required when SourceType is GITHUB). Default: ""
- `CreateECRRepository` (String): Whether to create an ECR repository for Docker images ("true" or "false"). Default: "true"
- `ECRRepositoryName` (String): Name of the ECR repository (defaults to ProjectName if empty). Default: ""
- `ImageTag` (String): Docker image tag to use for the built image. Default: latest
- `DockerfileContent` (String): Inline Dockerfile content for container builds. Default: Multi-stage Falcon MCP Dockerfile
- `BuildSpecContent` (String): Inline BuildSpec content for CodeBuild. Default: Complete Python + Docker build specification

## Source Control Options

This template supports three source control options:

### Option 1: AWS CodeCommit (Recommended - No External Dependencies)
- Fully AWS-managed Git repository
- No external tokens required
- Automatic webhook support

### Option 2: Amazon S3
- Upload source code as ZIP files to S3
- Manual build triggering only (no webhooks)
- No external dependencies

### Option 3: GitHub
- Requires GitHub personal access token
- Supports automatic webhooks on push
- External dependency on GitHub

## Prerequisites

### For AWS CodeCommit:
1. Create a CodeCommit repository:
   ```bash
   aws codecommit create-repository --repository-name falcon-mcp
   ```

2. Get the repository URL:
   ```bash
   aws codecommit get-repository --repository-name falcon-mcp --query 'repositoryMetadata.cloneUrlHttp'
   ```

### For GitHub (if using GitHub source):
1. **Create a GitHub Personal Access Token:**
   - Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Create a new token with `repo` scope (full control of private repositories)
   - Copy the token

2. **Store the token in AWS Secrets Manager:**
   ```bash
   aws secretsmanager create-secret \
     --name "falcon-mcp/github-token" \
     --description "GitHub personal access token for CodeBuild webhooks" \
     --secret-string '{"token":"YOUR_GITHUB_TOKEN_HERE"}'
   ```

3. **Get the secret ARN:**
   ```bash
   aws secretsmanager describe-secret --secret-id "falcon-mcp/github-token" --query 'ARN'
   ```

### For S3 (if using S3 source):
1. Create an S3 bucket:
   ```bash
   aws s3 mb s3://your-falcon-mcp-builds-bucket
   ```

2. Upload your source code as a ZIP file to the bucket

## Deployment Examples

### Deploy with AWS CodeCommit (Recommended):
```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --parameter-overrides \
    Environment=dev \
    ProjectName=falcon-mcp \
    SourceType=CODECOMMIT \
    RepositoryUrl=https://git-codecommit.region.amazonaws.com/v1/repos/falcon-mcp \
  --capabilities CAPABILITY_IAM
```

### Deploy with S3 Source:
```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --parameter-overrides \
    Environment=dev \
    ProjectName=falcon-mcp \
    SourceType=S3 \
    S3Bucket=your-falcon-mcp-builds-bucket \
    S3Key=source.zip \
  --capabilities CAPABILITY_IAM
```

### Deploy with GitHub:
```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --parameter-overrides \
    Environment=dev \
    ProjectName=falcon-mcp \
    SourceType=GITHUB \
    RepositoryUrl=https://github.com/CrowdStrike/falcon-mcp.git \
    GitHubTokenSecretArn=arn:aws:secretsmanager:region:account:secret:falcon-mcp/github-token \
  --capabilities CAPABILITY_IAM
```

### Deploy with ECR Support (Recommended for Docker deployment):
```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --parameter-overrides \
    Environment=dev \
    ProjectName=falcon-mcp \
    SourceType=CODECOMMIT \
    RepositoryUrl=https://git-codecommit.region.amazonaws.com/v1/repos/falcon-mcp \
    CreateECRRepository=true \
    ECRRepositoryName=falcon-mcp \
    ImageTag=latest \
  --capabilities CAPABILITY_IAM
```

## ECR Integration

When ECR is enabled (`CreateECRRepository=true`), the template provides:

### Features:
- **ECR Repository**: Created with lifecycle policies (keeps last 10 images)
- **Docker Image Scanning**: Automated security scanning on push
- **Build Integration**: Automatically builds and pushes Docker images if a Dockerfile is present
- **Image Tagging**: Tags images with both the specified tag and commit hash

### Dockerfile Overview:
The included `Dockerfile` implements a multi-stage build optimized for the Falcon MCP project:

- **Stage 1 (uv)**: Uses Astral's uv for ultra-fast Python dependency management
- **Dependencies**: Locks and installs dependencies using uv's efficient caching
- **Cleanup**: Removes unnecessary files from virtual environment
- **Stage 2 (runtime)**: Uses Python 3.13 Alpine for minimal runtime image
- **Security**: Runs as non-root user 'app'
- **Optimization**: Enables bytecode compilation and uses copy mode for caching

### Build Behavior:
- If a `Dockerfile` is present in the source repository, the build will:
  1. Build the Python package with `uv build`
  2. Authenticate with ECR
  3. Build the Docker image using multi-stage build
  4. Tag with both the specified tag and short commit hash
  5. Push both tags to ECR
  6. Generate `imagedefinitions.json` for CodePipeline integration

### Manual Build Trigger:
```bash
# Trigger build with ECR support
aws codebuild start-build --project-name falcon-mcp-dev
```

### ECR Repository Access:
After deployment, you can access the ECR repository:
```bash
# Login to ECR
aws ecr get-login-password --region your-region | docker login --username AWS --password-stdin your-account-id.dkr.ecr.your-region.amazonaws.com

# Pull the image
docker pull your-account-id.dkr.ecr.your-region.amazonaws.com/falcon-mcp:latest
```

## Architecture

The CloudFormation template creates:
1. A CodeBuild project with embedded BuildSpec and Dockerfile for complete build automation
2. Configurable source control (AWS CodeCommit, S3, or GitHub) with automatic webhooks
3. An optional ECR repository for Docker image storage with lifecycle policies
4. An IAM service role with appropriate permissions based on source type and ECR requirements
5. A CloudWatch log group for storing build logs

**Everything is Self-Contained**: Unlike external BuildSpec files, this template embeds all build logic, making deployments completely independent of repository contents.

## Build Process

The build process includes Python package building and optional Docker image creation:

- **Install Phase**: Installs uv (Python package manager) and sets up Python 3.13 runtime
- **Pre-build Phase**: Locks dependencies and syncs the virtual environment
- **Build Phase**:
  - Runs linting with ruff
  - Performs type checking with mypy
  - Checks code formatting with black
  - Runs tests with pytest
- **Post-build Phase**:
  - **Python Build**: Creates distribution packages with `uv build`
  - **Docker Build (when ECR enabled)**: Builds and pushes Docker images to ECR if a Dockerfile is present

## Environment Variables

The build environment is configured with:
- `UV_COMPILE_BYTECODE=1` - Enables Python bytecode compilation
- `UV_LINK_MODE=copy` - Copies dependencies instead of linking (required for CodeBuild)

## Caching

The buildspec includes caching for:
- uv cache directory (`$HOME/.cache/uv/**/*`)
- uv binary installations (`$HOME/.local/bin/**/*`)

This ensures faster subsequent builds by reusing downloaded packages and tools.
