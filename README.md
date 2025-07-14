# Bedrock Access Gateway

A CloudFormation template that deploys an OpenAI-compatible RESTful API gateway for Amazon Bedrock, allowing you to use existing OpenAI clients and libraries with Amazon Bedrock models.

## Overview

The Bedrock Access Gateway provides a proxy service that translates OpenAI API requests to Amazon Bedrock API calls, enabling seamless integration with existing OpenAI-compatible applications, libraries, and tools.

## Features

- **OpenAI API Compatibility**: Supports standard OpenAI API endpoints and request/response formats
- **Multi-Model Support**: Works with all Amazon Bedrock foundation models
- **Cross-Region Inference**: Enable inference across different AWS regions
- **Application Inference Profiles**: Support for Bedrock application inference profiles
- **Secure API Key Management**: Uses AWS Secrets Manager for API key storage
- **Load Balancer**: Deployed behind an Application Load Balancer for high availability
- **Private VPC Deployment**: Designed for secure deployment in private subnets with VPC endpoints
- **Resource Tagging**: All resources tagged with CreatedBy parameter for tracking

## Architecture

The solution deploys the following AWS resources:

- **ECS Fargate Service**: Handles API requests and translates them to Bedrock calls
- **Application Load Balancer**: Provides HTTP endpoint and load balancing
- **Security Groups**: Controls network access to the load balancer and ECS tasks
- **IAM Roles & Policies**: Manages permissions for Bedrock, Secrets Manager, and ECR access
- **VPC Endpoints**: For secure access to AWS services (Secrets Manager, ECR, CloudWatch Logs, Bedrock)

## Prerequisites

- AWS CLI configured with appropriate permissions
- Existing VPC with at least 2 private subnets in different availability zones
- Amazon Bedrock access enabled in your AWS account
- Docker image available in ECR (366590864501.dkr.ecr.[region].amazonaws.com/bedrock-proxy-api-ecs:latest)
- VPC endpoints for AWS services (Secrets Manager, ECR, CloudWatch Logs, Bedrock) or NAT Gateway

## Deployment Architecture

The solution is designed for secure deployment in private subnets:

### Private Subnet Deployment
- **ECS Tasks**: Deployed in private subnets for enhanced security
- **Load Balancer**: Internal load balancer in private subnets
- **Network Access**: Controlled via security groups and CIDR blocks
- **AWS Service Access**: Via VPC endpoints or NAT Gateway

### Required VPC Endpoints
For optimal security and performance, ensure your VPC has endpoints for:
- **Secrets Manager**: For API key retrieval
- **ECR**: For container image pulls
- **CloudWatch Logs**: For log streaming
- **Bedrock**: For model inference calls

## Deployment

### 1. Prepare Your Environment

Ensure you have the following information ready:
- VPC ID where you want to deploy the gateway
- At least 2 private subnet IDs in different availability zones
- Secret ARN in AWS Secrets Manager containing your API key
- Default model ID (optional, defaults to `anthropic.claude-3-5-sonnet-20241022-v1:0`)
- Your name or team name for the CreatedBy tag

### 2. Create API Key Secret

First, create a secret in AWS Secrets Manager to store your API key in the correct JSON format:

```bash
# Generate a random 16-character API key
API_KEY=$(openssl rand -base64 12 | tr -dc 'a-zA-Z0-9' | head -c 16)

# Create the secret with proper JSON format
aws secretsmanager create-secret \
    --name "bedrock-gateway-api-key" \
    --description "API Key for Bedrock Gateway" \
    --secret-string "{\"api_key\":\"$API_KEY\"}"
```

**Important**: The secret must be stored in this exact JSON format:
```json
{"api_key":"your16charvalue"}
```

### 3. Deploy the CloudFormation Stack

```bash
aws cloudformation create-stack \
    --stack-name bedrock-gateway \
    --template-body file://bedrock-gateway-private-vpc.yaml \
    --parameters \
        ParameterKey=ApiKeySecretArn,ParameterValue=arn:aws:secretsmanager:region:account:secret:bedrock-gateway-api-key-xxxxx \
        ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxx \
        ParameterKey=PrivateSubnetIds,ParameterValue="subnet-xxxxxxxxx,subnet-yyyyyyyyy" \
        ParameterKey=CreatedBy,ParameterValue=your-name-or-team \
    --capabilities CAPABILITY_IAM
```

### 4. Get the API Base URL

After deployment, get the API base URL from the CloudFormation outputs:

```bash
aws cloudformation describe-stacks \
    --stack-name bedrock-gateway \
    --query 'Stacks[0].Outputs[?OutputKey==`APIBaseUrl`].OutputValue' \
    --output text
```

## Usage

### Environment Variables

Set the following environment variables in your application:

```bash
export OPENAI_API_BASE="http://your-load-balancer-dns/api/v1"
export OPENAI_API_KEY="your-api-key-here"
```

### Example API Calls

#### Chat Completion

```bash
curl -X POST "http://your-load-balancer-dns/api/v1/chat/completions" \
  -H "Authorization: Bearer your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "anthropic.claude-3-5-sonnet-20241022-v1:0",
    "messages": [
      {"role": "user", "content": "Hello, how are you?"}
    ]
  }'
```

#### Text Completion

```bash
curl -X POST "http://your-load-balancer-dns/api/v1/completions" \
  -H "Authorization: Bearer your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "anthropic.claude-3-5-sonnet-20241022-v1:0",
    "prompt": "Hello, how are you?",
    "max_tokens": 100
  }'
```

#### Embeddings

```bash
curl -X POST "http://your-load-balancer-dns/api/v1/embeddings" \
  -H "Authorization: Bearer your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "cohere.embed-multilingual-v3",
    "input": "Hello, world!"
  }'
```

### Using with OpenAI Python Library

```python
import openai

# Configure the client
openai.api_base = "http://your-load-balancer-dns/api/v1"
openai.api_key = "your-api-key-here"

# Make a chat completion request
response = openai.ChatCompletion.create(
    model="anthropic.claude-3-5-sonnet-20241022-v1:0",
    messages=[
        {"role": "user", "content": "Hello, how are you?"}
    ]
)

print(response.choices[0].message.content)
```

## Supported Models

The gateway supports all Amazon Bedrock foundation models. Some popular models include:

- `anthropic.claude-3-5-sonnet-20241022-v1:0` (default)
- `anthropic.claude-3-5-haiku-20241022-v1:0`
- `anthropic.claude-3-5-opus-20241022-v1:0`
- `amazon.titan-text-express-v1`
- `meta.llama2-13b-chat-v1`
- `cohere.embed-multilingual-v3`

## Configuration

The ECS container supports the following environment variables:

- `DEBUG`: Enable debug logging (default: false)
- `DEFAULT_MODEL`: Default model ID for requests (default: anthropic.claude-3-5-sonnet-20241022-v1:0)
- `DEFAULT_EMBEDDING_MODEL`: Default embedding model (default: cohere.embed-multilingual-v3)
- `ENABLE_CROSS_REGION_INFERENCE`: Enable cross-region inference (default: true)
- `ENABLE_APPLICATION_INFERENCE_PROFILES`: Enable application inference profiles (default: true)

## Security

- API keys are stored securely in AWS Secrets Manager with proper JSON format
- All resources are tagged with a `CreatedBy` tag for tracking
- ECS tasks run in private subnets for enhanced security
- IAM roles follow the principle of least privilege
- Network access controlled via security groups and CIDR blocks

## Monitoring

Monitor your deployment using:

- CloudWatch Logs for the ECS tasks
- CloudWatch Metrics for the Application Load Balancer
- CloudTrail for API calls to Bedrock

## Troubleshooting

### Common Issues

1. **Secret format error**: Ensure the secret is stored in correct JSON format `{"api_key":"value"}`
2. **Network connectivity**: Check VPC endpoints or NAT Gateway configuration
3. **Permission denied**: Ensure the ECS task role has proper Bedrock permissions
4. **Model not found**: Verify the model ID is available in your AWS region
5. **Container pull failures**: Check ECR permissions and VPC endpoint configuration

### Logs

View ECS task logs:

```bash
aws logs describe-log-groups --log-group-name-prefix "/ecs/BedrockProxyFargate"
```

### Updating API Key

To update the API key with a new random value:

```bash
aws secretsmanager put-secret-value \
    --secret-id YOUR_SECRET_ARN \
    --secret-string "{\"api_key\":\"$(openssl rand -base64 12 | tr -dc 'a-zA-Z0-9' | head -c 16)\"}"
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:
- Create an issue in this repository
- Check the CloudWatch logs for detailed error information
- Verify your AWS account has Bedrock access enabled
- Ensure VPC endpoints are properly configured

## Related Links

- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference)
- [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
- [AWS VPC Endpoints Documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html) 