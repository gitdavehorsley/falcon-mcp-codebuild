# Falcon MCP CodeBuild

This repository contains AWS CloudFormation templates and build specifications for deploying a CodeBuild project for the CrowdStrike Falcon MCP Server.

## Files

- `template.yaml` - Main CloudFormation template that creates:
  - AWS CodeBuild project
  - IAM role with necessary permissions
  - CloudWatch log group for build logs

- `buildspec.yml` - Build specification file that defines the build phases and commands

## Parameters

- `Environment` (String): Environment name (dev, staging, prod). Default: dev
- `ProjectName` (String): Name of the CodeBuild project. Default: falcon-mcp
- `GitHubTokenSecretArn` (String): ARN of the AWS Secrets Manager secret containing the GitHub personal access token (required for webhooks)

## Prerequisites

### GitHub Access Token Setup

Before deploying, you need to create a GitHub personal access token and store it in AWS Secrets Manager:

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

   **Note:** The secret must be a JSON object with a `token` key containing your GitHub personal access token.

3. **Get the secret ARN:**
   ```bash
   aws secretsmanager describe-secret --secret-id "falcon-mcp/github-token" --query 'ARN'
   ```

## Deployment

To deploy this CloudFormation stack:

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --parameter-overrides \
    Environment=dev \
    ProjectName=falcon-mcp \
    GitHubTokenSecretArn=arn:aws:secretsmanager:region:account:secret:falcon-mcp/github-token \
  --capabilities CAPABILITY_IAM
```

## Architecture

The CloudFormation template creates:
1. A CodeBuild project that triggers on pushes to the main branch of the [CrowdStrike/falcon-mcp](https://github.com/CrowdStrike/falcon-mcp) repository
2. An IAM service role with permissions for CloudWatch logs and S3 access
3. A CloudWatch log group for storing build logs

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
