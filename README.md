# Webhook Repeater

A Terraform module that deploys an AWS Lambda function and API Gateway to receive webhooks and repeat/forward them to a specified endpoint.

## Overview

This module creates:
- An AWS Lambda function that receives webhook requests and forwards them to your specified endpoint
- An API Gateway (HTTP API) with a POST /webhook route that triggers the Lambda function
- Custom domain name support for the API Gateway
- CloudWatch logging for both Lambda and API Gateway

## Use Cases

- Forward webhooks to internal services that are not publicly accessible
- Standardize webhook handling across different services
- Log and monitor webhook traffic
- Modify or filter webhook requests before forwarding

## Requirements

- Terraform >= 0.13.0
- AWS provider
- Python 3.11
- Docker (for building the Lambda package)

## Usage

```hcl
module "webhook_repeater" {
  source = "github.com/theoutdoorprogrammer/repeater"

  name = "github"  # Used in resource naming
  webhook_endpoint = "https://internal-service.example.com/webhooks/receive"
  
  # VPC Configuration for Lambda
  lambda_subnet_ids = ["subnet-abc123", "subnet-def456"]
  lambda_security_group_ids = ["sg-abc123"]
  
  # API Gateway Custom Domain Configuration
  api_gateway_domain_name = "webhooks.example.com"
  acm_certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/abcdef-1234-5678-abcd-123456789012"
}
```

## Inputs

| Name | Description | Type | Required |
|------|-------------|------|----------|
| name | Name of the repeater (used in resource naming) | string | yes |
| webhook_endpoint | Endpoint URL to forward/repeat webhook requests to | string | yes |
| lambda_subnet_ids | List of subnet IDs for Lambda VPC configuration | list(string) | yes |
| lambda_security_group_ids | List of security group IDs for Lambda | list(string) | yes |
| api_gateway_domain_name | Custom domain name for API Gateway | string | yes |
| acm_certificate_arn | ARN of ACM certificate for the API Gateway domain | string | yes |

## Outputs

This module doesn't currently define outputs. To access internal resources, you can reference them using module attributes such as `module.webhook_repeater.module.function.lambda_function_arn`.

## Building and Deployment

The module includes a build script (`src/build.sh`) that:
1. Builds a Docker container for packaging the Lambda function
2. Installs Python dependencies
3. Creates the `code.zip` file used by the Lambda function

To update the Lambda package:

```bash
cd src
./build.sh
```

## How It Works

1. Incoming webhook requests are sent to the API Gateway endpoint (`https://{api_gateway_domain_name}/webhook`)
2. The Lambda function receives the request and:
   - Verifies it's a POST request
   - Handles Base64 encoded bodies
   - Removes the host header to avoid confusion
   - Forwards the request to your specified endpoint
   - Returns the response from your endpoint back to the original sender

## Logging

- Lambda logs are sent to CloudWatch Logs with a 30-day retention period
- API Gateway access logs are also configured with a 30-day retention period
- Request and response details are logged for debugging purposes

## License

See the [LICENSE](LICENSE) file for details.