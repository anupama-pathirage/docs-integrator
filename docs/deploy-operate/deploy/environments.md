---
title: Environments
---

# Environments (Dev, Test, Prod)

Manage environment-specific configuration to promote your integrations safely from development through testing to production.

## Promotion workflow

A typical promotion workflow moves your integration through three stages:

```
Development --> Testing --> Production
```

Each environment uses the same compiled artifact (JAR or Docker image) but with different configuration values. This ensures that the code tested in staging is identical to what runs in production -- only the configuration changes.

### Recommended workflow

1. **Development** -- Engineers run locally or deploy to a shared dev environment. Configuration points to sandbox APIs, local databases, and debug-level logging.
2. **Testing** -- The built artifact is deployed to a testing environment with production-like configuration. Integration tests and end-to-end tests run here.
3. **Production** -- After testing passes and approval is granted, the same artifact is promoted to production with production credentials and settings.

## Environment-Specific Config.toml

Ballerina uses `Config.toml` for runtime configuration. Maintain separate configuration files for each environment:

```
project-root/
  Config.toml              # Default (local development)
  config/
    dev-Config.toml
    test-Config.toml
    prod-Config.toml
```

### Development configuration

```toml
# config/dev-Config.toml
[orderService]
port = 9090
logLevel = "DEBUG"

[orderService.database]
host = "localhost"
port = 3306
name = "orders_dev"
user = "dev_user"
password = "dev_password"

[orderService.externalApi]
baseUrl = "https://sandbox.api.example.com"
apiKey = "dev-key-12345"
timeoutSeconds = 30
```

### Testing configuration

```toml
# config/test-Config.toml
[orderService]
port = 9090
logLevel = "INFO"

[orderService.database]
host = "db-test.internal.example.com"
port = 3306
name = "orders_test"
user = "test_user"
password = "${TEST_DB_PASSWORD}"

[orderService.externalApi]
baseUrl = "https://staging.api.example.com"
apiKey = "${TEST_API_KEY}"
timeoutSeconds = 15
```

### Production configuration

```toml
# config/prod-Config.toml
[orderService]
port = 9090
logLevel = "WARN"

[orderService.database]
host = "db-prod.internal.example.com"
port = 3306
name = "orders_prod"
user = "prod_user"
password = "${PROD_DB_PASSWORD}"

[orderService.externalApi]
baseUrl = "https://api.example.com"
apiKey = "${PROD_API_KEY}"
timeoutSeconds = 10
```

## Environment variables per target

Override configuration values using environment variables without modifying `Config.toml` files. This is the recommended approach for secrets in containerized deployments.

### Configurable variables in Ballerina

Declare configurable variables in your Ballerina code:

```ballerina
configurable int port = 9090;
configurable string dbHost = "localhost";
configurable string dbPassword = ?;  // Required, no default
```

### Setting values via environment variables

```bash
# Convention: BAL_CONFIG_VAR__
export BAL_CONFIG_VAR_ORDER_SERVICE_PORT=8080
export BAL_CONFIG_VAR_ORDER_SERVICE_DB_HOST=db-prod.internal.example.com
export BAL_CONFIG_VAR_ORDER_SERVICE_DB_PASSWORD=s3cret
```

### Kubernetes ConfigMaps and secrets

In Kubernetes, use ConfigMaps for non-sensitive values and Secrets for credentials:

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: integrator-config
  namespace: production
data:
  Config.toml: |
    [orderService]
    port = 9090
    logLevel = "WARN"
    [orderService.database]
    host = "db-prod.internal.example.com"
    port = 3306
    name = "orders_prod"
```

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: integrator-secrets
  namespace: production
type: Opaque
stringData:
  DB_PASSWORD: "s3cret-prod-password"
  API_KEY: "prod-api-key-67890"
```

Mount the ConfigMap and inject secrets as environment variables in your deployment:

```yaml
# k8s/deployment.yaml
spec:
  containers:
    - name: app
      env:
        - name: BAL_CONFIG_FILES
          value: "/config/Config.toml"
        - name: BAL_CONFIG_VAR_ORDER_SERVICE_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: integrator-secrets
              key: DB_PASSWORD
      volumeMounts:
        - name: config
          mountPath: /config
  volumes:
    - name: config
      configMap:
        name: integrator-config
```

## Blue-Green and canary deployments

### Blue-Green deployment

Run two identical environments (blue and green). Route traffic to the active one and deploy updates to the inactive one. After verification, switch traffic.

```bash
# Deploy new version to the inactive (green) environment
kubectl apply -f k8s/green/deployment.yaml

# Verify the green deployment is healthy
kubectl rollout status deployment/integrator-green -n production

# Switch traffic from blue to green
kubectl patch service integrator-service -n production \
  -p '{"spec":{"selector":{"version":"green"}}}'

# After verification, scale down the old (blue) deployment
kubectl scale deployment/integrator-blue --replicas=0 -n production
```

### Canary deployment

Route a small percentage of traffic to the new version. Gradually increase if metrics are healthy.

```yaml
# k8s/canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: integrator-canary
  namespace: production
spec:
  replicas: 1  # Small fraction of total replicas
  selector:
    matchLabels:
      app: integrator
      track: canary
  template:
    metadata:
      labels:
        app: integrator
        track: canary
    spec:
      containers:
        - name: app
          image: registry.example.com/integrator:v2.0.0
```

Use a single Kubernetes Service with a shared label (`app: integrator`) so both the stable and canary deployments receive traffic proportional to their replica count.

## What's next

- [Scaling & High Availability](scaling-high-availability.md) -- Configure scaling and resilience
- [Secrets & Encryption](../secure/secrets-encryption.md) -- Secure your configuration values
- [CI/CD Pipelines](../cicd/github-actions.md) -- Automate environment promotions
