---
title: Serverless Deployment
---

# Serverless Deployment

WSO2 Integrator projects can be deployed as serverless functions, enabling event-driven scaling with zero infrastructure management. Ballerina supports AWS Lambda and Azure Functions as first-class deployment targets.

## Overview

| Feature | AWS Lambda | Azure Functions |
|---------|-----------|-----------------|
| Trigger Types | HTTP (API Gateway), S3, SQS, DynamoDB Streams | HTTP, Timer, Blob Storage, Queue, Event Hub |
| Cold Start | ~2-5s (JVM), under 100ms (GraalVM) | ~2-5s (JVM), under 100ms (GraalVM) |
| Max Execution | 15 minutes | 10 minutes (Consumption plan) |
| Memory | 128 MB - 10 GB | 128 MB - 14 GB |
| Language Runtime | Java 17+ | Java 17+ |

## AWS lambda

### Step 1 -- create the lambda function

Write a Ballerina function annotated with `@awslambda:Function`:

```ballerina
import ballerinax/awslambda;

@awslambda:Function
public function orderProcessor(awslambda:Context ctx,
                                json payload) returns json|error {
    // Extract order details
    string orderId = check payload.orderId;

    // Process the order (call downstream services, transform data, etc.)
    json result = check processOrder(orderId);

    return {
        statusCode: 200,
        body: result
    };
}
```

### Step 2 -- build for lambda

```bash
bal build --cloud=aws_lambda
```

This generates the deployment artifacts:

```
target/
  aws/
    my_integration.zip       # Lambda deployment package
    aws-ballerina-lambda.yaml  # SAM template
```

### Step 3 -- deploy with AWS SAM

```bash
sam deploy \
  --template-file target/aws/aws-ballerina-lambda.yaml \
  --stack-name my-integration-stack \
  --capabilities CAPABILITY_IAM \
  --region us-east-1
```

Alternatively, deploy with the AWS CLI:

```bash
aws lambda create-function \
  --function-name orderProcessor \
  --runtime java17 \
  --handler my_integration.orderProcessor \
  --zip-file fileb://target/aws/my_integration.zip \
  --role arn:aws:iam::123456789:role/lambda-execution-role \
  --memory-size 512 \
  --timeout 30
```

### Step 4 -- add an API gateway trigger

```bash
aws apigatewayv2 create-api \
  --name order-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:us-east-1:123456789:function:orderProcessor
```

### Lambda configuration

Configure via environment variables in the Lambda console or template:

```yaml
Environment:
  Variables:
    DB_HOST: "db.internal.example.com"
    DB_PORT: "5432"
    API_KEY: "{{resolve:secretsmanager:my-api-key}}"
```

## Azure functions

### Step 1 -- create the Azure function

```ballerina
import ballerinax/azure.functions;

@functions:Function
public function httpTrigger(@functions:HTTPTrigger {}
                            functions:HTTPRequest request)
                            returns @functions:HTTPOutput functions:HTTPResponse {
    json payload = check request.getJsonPayload();
    string name = check payload.name;

    return {
        statusCode: 200,
        body: "Hello, " + name
    };
}
```

### Step 2 -- build for Azure functions

```bash
bal build --cloud=azure_functions
```

Generated artifacts:

```
target/
  azure_functions/
    my_integration/
      function.json
      handler.jar
    host.json
```

### Step 3 -- deploy to Azure

```bash
# Create resources
az group create --name integration-rg --location eastus
az storage account create --name integrationstorage --location eastus \
  --resource-group integration-rg --sku Standard_LRS
az functionapp create --resource-group integration-rg \
  --consumption-plan-location eastus \
  --runtime java --runtime-version 17 \
  --functions-version 4 \
  --name my-integration-func \
  --storage-account integrationstorage

# Deploy
az functionapp deployment source config-zip \
  --resource-group integration-rg \
  --name my-integration-func \
  --src target/azure_functions/my_integration.zip
```

### Azure configuration

Set application settings for environment-specific configuration:

```bash
az functionapp config appsettings set \
  --name my-integration-func \
  --resource-group integration-rg \
  --settings "DB_HOST=db.internal.example.com" "DB_PORT=5432"
```

## Reducing cold start times

### Use GraalVM native images

Compile to a native binary to dramatically reduce cold start times:

```bash
bal build --graalvm --cloud=aws_lambda
```

This produces a native executable that starts in under 100ms compared to 2-5 seconds for JVM-based deployments.

### Provisioned concurrency (AWS)

Keep warm instances ready to handle requests:

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name orderProcessor \
  --qualifier production \
  --provisioned-concurrent-executions 5
```

### Premium plan (Azure)

Use the Premium plan for pre-warmed instances:

```bash
az functionapp plan create --name premium-plan \
  --resource-group integration-rg \
  --location eastus \
  --sku EP1 --min-instances 1
```

## Best practices

| Practice | Recommendation |
|----------|---------------|
| Function Size | Keep functions focused on a single operation |
| Dependencies | Minimize package dependencies to reduce deployment size |
| Timeouts | Set appropriate timeouts (default is often too high) |
| Secrets | Use cloud-native secret managers (Secrets Manager, Key Vault) |
| Observability | Enable X-Ray (AWS) or Application Insights (Azure) |
| VPC Access | Configure VPC/VNet only when accessing private resources |

## What's next

- [GraalVM Native Images](graalvm-native-images.md) -- Compile to native binaries for minimal cold start
- [Managing Configurations](managing-configurations.md) -- Environment-specific configuration strategies
- [Deploy to AWS / Azure / GCP](aws-azure-gcp.md) -- Container-based cloud deployments
