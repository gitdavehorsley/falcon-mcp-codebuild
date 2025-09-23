# Falcon MCP CodeBuild

This repository contains AWS CloudFormation templates and build specifications for deploying a CodeBuild project for the CrowdStrike Falcon MCP Server with flexible source control options.

## Files

- `template.yaml` - Main CloudFormation template that creates:
  - AWS CodeBuild project
  - IAM role with necessary permissions
  - CloudWatch log group for build logs

- `buildspec.yml` - Build specification file that defines the build phases and commands

## Parameters

- `Environment` (String): Environment name (dev, staging, prod). Default: dev
- `ProjectName` (String): Name of the CodeBuild project. Default: falcon-mcp
- `SourceType` (String): Source type for the CodeBuild project (CODECOMMIT, S3, GITHUB). Default: S3
- `RepositoryUrl` (String): Git repository URL (leave empty to use S3 source). Default: ""
- `S3Bucket` (String): S3 bucket name for source code (required when SourceType is S3). Default: ""
- `S3Key` (String): S3 object key for source code (used when SourceType is S3). Default: source.zip
- `GitHubTokenSecretArn` (String): ARN of the AWS Secrets Manager secret containing the GitHub personal access token (required when SourceType is GITHUB). Default: ""

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

## Architecture

The CloudFormation template creates:
1. A CodeBuild project with configurable source (AWS CodeCommit, S3, or GitHub)
2. Automatic webhooks for Git-based sources (CodeCommit/GitHub) that trigger on main branch pushes
3. An IAM service role with appropriate permissions based on source type
4. A CloudWatch log group for storing build logs

## Build Process

The build process mirrors the Docker build from the source repository and includes:

- **Install Phase**: Installs uv (Python package manager) and sets up Python 3.13 runtime
- **Pre-build Phase**: Locks dependencies and syncs the virtual environment
- **Build Phase**:
  - Runs linting with ruff
  - Performs type checking with mypy
  - Checks code formatting with black
  - Runs tests with pytest
- **Post-build Phase**: Creates distribution packages with `uv build`

## Environment Variables

The build environment is configured with:
- `UV_COMPILE_BYTECODE=1` - Enables Python bytecode compilation
- `UV_LINK_MODE=copy` - Copies dependencies instead of linking (required for CodeBuild)

## Caching

The buildspec includes caching for:
- uv cache directory (`$HOME/.cache/uv/**/*`)
- uv binary installations (`$HOME/.local/bin/**/*`)

This ensures faster subsequent builds by reusing downloaded packages and tools.
