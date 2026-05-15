---
title: Actions
---

# Actions

The `ballerinax/aws.marketplace.mpe` package exposes the following clients:

| Client | Purpose |
|--------|---------|
| [`Client`](#client) | Retrieves customer entitlements for AWS Marketplace products. |

---

## Client

Retrieves customer entitlements for AWS Marketplace products.

### Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `region` | `Region` | Required | The AWS region for the Marketplace Entitlement Service endpoint (e.g., `US_EAST_1`). |
| `auth` | `AuthConfig` | Required | AWS credentials including `accessKeyId`, `secretAccessKey`, and optional `sessionToken`. |

### Initializing the client

```ballerina
import ballerinax/aws.marketplace.mpe;

configurable string accessKeyId = ?;
configurable string secretAccessKey = ?;

mpe:Client mpeClient = check new (
    region = mpe:US_EAST_1,
    auth = {
        accessKeyId,
        secretAccessKey
    }
);
```

### Operations

#### Entitlements

<details>
<summary>getEntitlements</summary>

Retrieves the entitlement values for a given product, with optional filtering by customer identifier or dimension and pagination support.

Parameters:

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `productCode` | `string` | Yes | The AWS Marketplace product code (1–255 characters). |
| `filter` | `EntitlementFilter` | No | Optional filter with `customerIdentifier` and/or `dimension` string arrays to narrow results. |
| `maxResults` | `int` | No | Maximum number of entitlements to return per request. |
| `nextToken` | `string` | No | Pagination token from a previous response to retrieve the next page of results. |

Returns: `EntitlementsResponse|Error`

Sample code:

```ballerina
mpe:EntitlementsResponse response = check mpeClient->getEntitlements(
    productCode = "abc123def456"
);
```

Sample response:

```ballerina
{
  "entitlements": [
    {
      "productCode": "abc123def456",
      "dimension": "users",
      "customerIdentifier": "CUST-abcdef123456",
      "expirationDate": [1735689600, 0],
      "value": 10
    }
  ],
  "nextToken": null
}
```

</details>

<details>
<summary>close</summary>

Closes the AWS MPE client and releases associated resources.

Returns: `Error?`

Sample code:

```ballerina
check mpeClient->close();
```

</details>

#### Filtered queries

<details>
<summary>getEntitlements (filtered by customer)</summary>

Retrieves entitlements filtered by a specific customer identifier.

Parameters:

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `productCode` | `string` | Yes | The AWS Marketplace product code. |
| `filter` | `EntitlementFilter` | Yes | Filter containing `customerIdentifier` array to filter by specific customers. |

Returns: `EntitlementsResponse|Error`

Sample code:

```ballerina
mpe:EntitlementsResponse response = check mpeClient->getEntitlements(
    productCode = "abc123def456",
    filter = {
        customerIdentifier: ["CUST-abcdef123456"]
    }
);
```

Sample response:

```ballerina
{
  "entitlements": [
    {
      "productCode": "abc123def456",
      "dimension": "users",
      "customerIdentifier": "CUST-abcdef123456",
      "expirationDate": [1735689600, 0],
      "value": 10
    },
    {
      "productCode": "abc123def456",
      "dimension": "storage_gb",
      "customerIdentifier": "CUST-abcdef123456",
      "expirationDate": [1735689600, 0],
      "value": 50
    }
  ],
  "nextToken": null
}
```

</details>
