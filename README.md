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
- **Existing Infrastructure**: Designed to work with existing VPC and subnet infrastructure

## Architecture

The solution deploys the following AWS resources:

- **Lambda Function**: Handles API requests and translates them to Bedrock calls
- **Application Load Balancer**: Provides HTTP endpoint and load balancing
- **Security Group**: Controls network access to the load balancer
- **IAM Roles & Policies**: Manages permissions for Bedrock and Secrets Manager access

## Prerequisites

- AWS CLI configured with appropriate permissions
- Existing VPC with at least 2 public subnets
- Amazon Bedrock access enabled in your AWS account
- Docker image available in ECR (366590864501.dkr.ecr.[region].amazonaws.com/bedrock-proxy-api:latest)

## Deployment

### 1. Prepare Your Environment

Ensure you have the following information ready:
- VPC ID where you want to deploy the load balancer
- Two public subnet IDs in different availability zones
- Secret ARN in AWS Secrets Manager containing your API key
- Default model ID (optional, defaults to `anthropic.claude-3-sonnet-20240229-v1:0`)

### 2. Create API Key Secret

First, create a secret in AWS Secrets Manager to store your API key:

```bash
aws secretsmanager create-secret \
    --name "bedrock-gateway-api-key" \
    --description "API Key for Bedrock Gateway" \
    --secret-string "your-api-key-here"
```

### 3. Deploy the CloudFormation Stack

```bash
aws cloudformation create-stack \
    --stack-name bedrock-gateway \
    --template-body file://Description:\ Bedrock\ Access\ Gateway\ -\ Op.yaml \
    --parameters \
        ParameterKey=ApiKeySecretArn,ParameterValue=arn:aws:secretsmanager:region:account:secret:bedrock-gateway-api-key-xxxxx \
        ParameterKey=VpcId,ParameterValue=vpc-xxxxxxxxx \
        ParameterKey=PublicSubnet1Id,ParameterValue=subnet-xxxxxxxxx \
        ParameterKey=PublicSubnet2Id,ParameterValue=subnet-xxxxxxxxx \
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
    "model": "anthropic.claude-3-sonnet-20240229-v1:0",
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
    "model": "anthropic.claude-3-sonnet-20240229-v1:0",
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
    model="anthropic.claude-3-sonnet-20240229-v1:0",
    messages=[
        {"role": "user", "content": "Hello, how are you?"}
    ]
)

print(response.choices[0].message.content)
```

## Supported Models

The gateway supports all Amazon Bedrock foundation models. Some popular models include:

- `anthropic.claude-3-sonnet-20240229-v1:0`
- `anthropic.claude-3-haiku-20240307-v1:0`
- `anthropic.claude-3-opus-20240229-v1:0`
- `amazon.titan-text-express-v1`
- `meta.llama2-13b-chat-v1`
- `cohere.embed-multilingual-v3`

## Configuration

The Lambda function supports the following environment variables:

- `DEBUG`: Enable debug logging (default: false)
- `API_KEY_SECRET_ARN`: ARN of the secret containing the API key
- `DEFAULT_MODEL`: Default model ID for requests
- `DEFAULT_EMBEDDING_MODEL`: Default embedding model (default: cohere.embed-multilingual-v3)
- `ENABLE_CROSS_REGION_INFERENCE`: Enable cross-region inference (default: true)
- `ENABLE_APPLICATION_INFERENCE_PROFILES`: Enable application inference profiles (default: true)

## Security

- API keys are stored securely in AWS Secrets Manager
- All resources are tagged with a `CreatedBy` tag for tracking
- The load balancer is deployed in public subnets but can be secured with additional security groups
- IAM roles follow the principle of least privilege

## Monitoring

Monitor your deployment using:

- CloudWatch Logs for the Lambda function
- CloudWatch Metrics for the Application Load Balancer
- CloudTrail for API calls to Bedrock

## Troubleshooting

### Common Issues

1. **Lambda timeout**: Increase the timeout value in the CloudFormation template
2. **Permission denied**: Ensure the Lambda role has proper Bedrock permissions
3. **Model not found**: Verify the model ID is available in your AWS region
4. **Network connectivity**: Check security group rules and VPC configuration

### Logs

View Lambda function logs:

```bash
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/ProxyApiHandler"
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

## Related Links

- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference)
- [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/) 