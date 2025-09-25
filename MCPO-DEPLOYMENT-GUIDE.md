# MCPO ECS Deployment Guide

This guide shows you how to deploy the [MCPO MCP-to-OpenAPI proxy server](https://github.com/open-webui/mcpo) on AWS ECS using the provided CloudFormation template.

## Overview

MCPO (MCP-to-OpenAPI) is a proxy server that exposes Model Context Protocol (MCP) tools as standard OpenAPI-compatible HTTP endpoints. This makes MCP tools instantly compatible with any system that expects REST APIs.

## Prerequisites

- AWS CLI configured with appropriate permissions
- An existing VPC with subnets
- An ECS cluster (Fargate compatible)
- Basic understanding of CloudFormation and ECS

## Quick Deployment

### Single MCP Server Mode

Deploy a single MCP server (e.g., time server):

```bash
# Auto-generate API key (recommended)
aws cloudformation deploy \
  --template-file mcpo-ecs-task-definition.yaml \
  --stack-name mcpo-time-server-dev \
  --parameter-overrides \
    Environment=dev \
    MCPServerCommand="uvx mcp-server-time --local-timezone=America/New_York" \
    VpcId="vpc-xxxxxxxxxx" \
    WebUISecurityGroupId="sg-xxxxxxxxxx" \
  --capabilities CAPABILITY_NAMED_IAM

# Or specify custom API key
aws cloudformation deploy \
  --template-file mcpo-ecs-task-definition.yaml \
  --stack-name mcpo-time-server-dev \
  --parameter-overrides \
    Environment=dev \
    MCPOApiKey="your-custom-api-key-here" \
    MCPServerCommand="uvx mcp-server-time --local-timezone=America/New_York" \
    VpcId="vpc-xxxxxxxxxx" \
    WebUISecurityGroupId="sg-xxxxxxxxxx" \
  --capabilities CAPABILITY_NAMED_IAM
```

### Multi-Server Configuration Mode

For multiple MCP servers using a config file:

1. **Create your config file** (see `mcpo-config-example.json`)
2. **Upload to S3 or mount as volume** (config file needs to be accessible to container)
3. **Deploy with config file parameter**:

```bash
aws cloudformation deploy \
  --template-file mcpo-ecs-task-definition.yaml \
  --stack-name mcpo-multi-server-dev \
  --parameter-overrides \
    Environment=dev \
    MCPOConfigFile="/path/to/config.json" \
    EnableHotReload="true" \
    VpcId="vpc-xxxxxxxxxx" \
    WebUISecurityGroupId="sg-xxxxxxxxxx" \
  --capabilities CAPABILITY_NAMED_IAM
```

## Configuration Parameters

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `VpcId` | VPC ID where ECS will run | `"vpc-1234567890abcdef0"` |
| `WebUISecurityGroupId` | Security Group ID of the WebUI server | `"sg-1234567890abcdef0"` |

### Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `MCPOApiKey` | Custom API key (leave empty to auto-generate) | `""` (auto-generated) |

### MCP Server Configuration

#### Single Server Mode
| Parameter | Description | Default |
|-----------|-------------|---------|
| `MCPServerCommand` | Command to start MCP server | `"uvx mcp-server-time --local-timezone=America/New_York"` |
| `MCPServerType` | Server type (stdio/sse/streamable-http) | `"stdio"` |
| `MCPServerURL` | URL for non-stdio servers | `""` |
| `MCPServerHeaders` | JSON headers for HTTP/SSE servers | `"{}"` |

#### Multi-Server Mode
| Parameter | Description | Default |
|-----------|-------------|---------|
| `MCPOConfigFile` | Path to config file | `""` (empty = single server mode) |
| `EnableHotReload` | Auto-reload on config changes | `"false"` |

### Infrastructure Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `TaskCpu` | CPU units (256=0.25 vCPU) | `"256"` |
| `TaskMemory` | Memory in MiB | `"512"` |
| `ContainerPort` | Port for MCPO service | `8000` |
| `AllowedCidrBlocks` | Optional additional IP ranges for admin access | `""` (empty by default) |

## MCP Server Examples

### Popular MCP Servers

```bash
# Time server
MCPServerCommand="uvx mcp-server-time --local-timezone=America/New_York"

# Memory server (for conversation context)
MCPServerCommand="npx -y @modelcontextprotocol/server-memory"

# Filesystem server
MCPServerCommand="npx -y @modelcontextprotocol/server-filesystem /tmp"

# SQLite database server
MCPServerCommand="uvx mcp-server-sqlite --db-path=/data/app.db"

# Weather server
MCPServerCommand="uvx mcp-server-weather"

# GitHub integration
MCPServerCommand="npx -y @modelcontextprotocol/server-github"
# (requires GITHUB_PERSONAL_ACCESS_TOKEN environment variable)

# Brave Search
MCPServerCommand="npx -y @modelcontextprotocol/server-brave-search"
# (requires BRAVE_API_KEY environment variable)
```

### SSE/HTTP Server Examples

```bash
# SSE server
MCPServerType="sse"
MCPServerURL="http://127.0.0.1:8001/sse"
MCPServerHeaders='{"Authorization": "Bearer token123"}'

# Streamable HTTP server
MCPServerType="streamable-http"
MCPServerURL="http://127.0.0.1:8002/mcp"
```

## Post-Deployment

After deployment, you'll have:

### 1. OpenAPI Endpoints

- **Service URL**: `http://your-load-balancer:8000`
- **Interactive Docs**: `http://your-load-balancer:8000/docs`
- **OpenAPI Schema**: `http://your-load-balancer:8000/openapi.json`

### 2. Multi-Server Routes (Config Mode)

Each server gets its own route:
- `http://your-load-balancer:8000/memory/docs`
- `http://your-load-balancer:8000/time/docs`
- `http://your-load-balancer:8000/github/docs`

### 3. Retrieve the API Key

The API key is stored securely in AWS Secrets Manager. To retrieve it:

```bash
# Get the secret name from CloudFormation outputs
SECRET_NAME=$(aws cloudformation describe-stacks \
  --stack-name mcpo-time-server-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`MCPOApiKeySecretName`].OutputValue' \
  --output text)

# Retrieve the API key
API_KEY=$(aws secretsmanager get-secret-value \
  --secret-id $SECRET_NAME \
  --query 'SecretString' \
  --output text | jq -r '.apiKey')

echo "Your MCPO API Key: $API_KEY"
```

### 4. Authentication

All endpoints require the API key in the Authorization header:
```bash
curl -H "Authorization: Bearer $API_KEY" \
  http://your-load-balancer:8000/docs
```

## Integration with Load Balancer

The task definition includes a security group, but you'll typically want to add an Application Load Balancer:

```bash
# Create target group
aws elbv2 create-target-group \
  --name mcpo-targets \
  --protocol HTTP \
  --port 8000 \
  --vpc-id vpc-xxxxxxxxxx \
  --target-type ip \
  --health-check-path "/docs"

# Register ECS service with target group in your service definition
```

## Environment Variables

You can extend the container environment by modifying the `Environment` section in the CloudFormation template:

```yaml
Environment:
  - Name: MCPO_API_KEY
    Value: !Ref MCPOApiKey
  - Name: MCPO_PORT
    Value: !Ref ContainerPort
  # Add custom environment variables for your MCP servers
  - Name: GITHUB_PERSONAL_ACCESS_TOKEN
    Value: !Ref GitHubToken  # Add as parameter
  - Name: BRAVE_API_KEY
    Value: !Ref BraveApiKey  # Add as parameter
```

## Monitoring and Logging

### CloudWatch Logs
Logs are automatically sent to CloudWatch:
- **Log Group**: `/ecs/mcpo-{environment}`
- **Retention**: 30 days (configurable)

### Health Checks
The task definition includes a health check that:
- Calls `/docs` endpoint with API key
- Runs every 30 seconds
- Fails after 3 consecutive failures

### Monitoring Queries

```bash
# View recent logs
aws logs tail /ecs/mcpo-dev --follow

# Check health check status
aws ecs describe-services \
  --cluster your-cluster \
  --services mcpo-service
```

## Troubleshooting

### Common Issues

1. **Container fails to start**
   - Check CloudWatch logs for MCP server errors
   - Verify the MCP server command is correct
   - Ensure required environment variables are set

2. **Health check failures**
   - Verify API key is correct
   - Check if MCPO is listening on the expected port
   - Review container logs for startup errors

3. **MCP server connection issues**
   - For stdio: Check if the command runs in the container
   - For SSE/HTTP: Verify URL accessibility and headers
   - Check network connectivity between services

4. **Permission errors**
   - Review ECS task role permissions
   - Check if MCP server needs additional AWS permissions
   - Verify file system access for file-based servers

### Debug Commands

```bash
# Get task logs
aws logs tail /ecs/mcpo-dev --follow

# Describe task to see failure reasons
aws ecs describe-tasks \
  --cluster your-cluster \
  --tasks task-id

# Test API endpoint manually
curl -H "Authorization: Bearer your-api-key" \
  http://your-load-balancer:8000/docs
```

## Security Considerations

1. **API Key Management**
   - ✅ **Automatic**: API keys are auto-generated and stored in AWS Secrets Manager
   - ✅ **Secure**: Keys are never exposed in CloudFormation parameters or logs
   - ✅ **Retrievable**: Use AWS CLI to securely retrieve keys when needed
   - **Best Practice**: Rotate keys regularly using Secrets Manager rotation

2. **Network Security**
   - **Primary Access**: Only the WebUI server security group can access MCPO by default
   - **Optional Admin Access**: Additional CIDR blocks can be specified for debugging/admin access
   - Use private subnets when possible
   - Consider VPC endpoints for AWS service access

3. **Security Group Architecture**
   ```
   WebUI Server (sg-webui) → MCPO Server (sg-mcpo) → Internet (HTTPS/HTTP out)
   ```
   - MCPO only accepts inbound traffic from the WebUI security group
   - No direct internet access to MCPO unless explicitly configured
   - MCPO can make outbound calls for MCP server package downloads

3. **MCP Server Security**
   - Review what each MCP server can access
   - Use least-privilege principles
   - Monitor for unusual API usage patterns

## Cost Optimization

- Start with minimal resources (`256` CPU, `512` memory)
- Use Fargate Spot for development environments
- Monitor CloudWatch metrics to right-size resources
- Consider scheduled scaling for predictable workloads

## Next Steps

1. **Set up Application Load Balancer** for production traffic
2. **Configure auto-scaling** based on CPU/memory metrics
3. **Set up monitoring alerts** for health check failures
4. **Implement CI/CD pipeline** for automated deployments
5. **Add custom MCP servers** for your specific use cases

For more information about MCPO, visit the [official repository](https://github.com/open-webui/mcpo).
