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

## Deployment

To deploy this CloudFormation stack:

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name falcon-mcp-codebuild-dev \
  --parameter-overrides Environment=dev ProjectName=falcon-mcp \
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
