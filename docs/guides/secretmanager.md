---
doc_type: guide
---

# Secret Manager

Secret Manager provides a secure, centralized solution for managing confidential resources such as API keys, tokens, database credentials, and other sensitive information.
It integrates seamlessly with Tailor Platform services, allowing them to access secrets without exposing them in configuration files.
The service also supports versioning and fine-grained access control.

Common use cases include:

- Storing OAuth client secrets for Auth service integration
- Managing API keys for third-party service integrations
- Securing database connection strings and credentials
- Storing encryption keys and SSL/TLS certificates

## How to set up secrets

There are two ways to register secrets in Secret Manager:

You can define secrets using the SDK with `defineSecret()`:

```typescript {{ title: 'secrets.ts' }}
import { defineSecret } from "@tailor-platform/sdk";

export const secrets = defineSecret("my-secrets", {
  "api-key": { description: "External API key" },
});
```

- **name**: The first argument is the vault name for organizing secrets
- **secrets**: An object where each key is a secret name and the value contains metadata like description

You can manage secrets using the Terraform provider with dedicated vault and secret resources:

```sh {{ title: 'vaults.tf' }}
resource "tailor_secretmanager_vault" "default" {
  workspace_id = tailor_workspace.starwars.id
  name         = "default"
}

resource "tailor_secretmanager_secret" "my-secret" {
  workspace_id = tailor_workspace.starwars.id
  vault_name   = tailor_secretmanager_vault.default.name

  name             = "my-secret"
  value_wo         = "my-secret-value"
  value_wo_version = 1
}
```

- **workspace_id**:(String) The ID of the workspace where the vault will be created
- **name**:(String) A unique name for the vault within the workspace
- **vault_name**:(String) Reference to the vault where the secret will be stored
- **value_wo**:(String) The actual secret value (write-only for security)
- **value_wo_version**:(Number) Version number for the secret value

You can also manage secrets using the tailor-sdk command-line interface:

```bash
# Create a vault
tailor-sdk secret vault create default

# Create a secret in the vault
tailor-sdk secret create --vault-name default --name my-secret --value my-secret-value
```

- `tailor-sdk secret vault create`: Creates a new vault in the current workspace (vault name as positional argument)
- `tailor-sdk secret create`: Adds a new secret to an existing vault
- Use `--vault-name` to specify the target vault name
- Use `--name` to set the secret identifier
- Use `--value` to provide the secret value

## Use cases

### Auth Service Integration

Secret Manager is commonly used with the Auth service to store sensitive authentication credentials:

```sh {{ title: 'auth.tf' }}
resource "tailor_auth_idp_config" "oidc_local" {
  workspace_id = tailor_workspace.ims.id
  namespace    = tailor_auth.ims_auth.namespace

  name = "oidc-local"

  oidc_config = {
    client_id = "<client-id>"
    client_secret = {
      vault_name  = tailor_secretmanager_vault.default.name
      secret_name = tailor_secretmanager_secret.oidc-client-secret.name
    }
    provider_url = "<your_auth_provider_url>"
  }
}
```

### Function Runtime Integration

You can access Secret Manager values from Function Runtime using the built-in `tailor.secretmanager` API.
Use secret values only for in-process operations such as initializing a client or calling a downstream service, and avoid logging or returning the raw values.

```js {{ title: 'function_example.js' }}
export default async () => {
  console.log("start");

  // Get multiple secrets from a vault
  const secrets = await tailor.secretmanager.getSecrets("default", [
    "test-secret-1",
    "test-secret-2",
  ]);

  // Validate that the required secrets are available without exposing their values
  if (!secrets["test-secret-1"] || !secrets["test-secret-2"]) {
    throw new Error("required secrets are missing");
  }

  // Get a single secret
  const secret = await tailor.secretmanager.getSecret("default", "test-secret-1");

  if (!secret) {
    throw new Error("required secret is missing");
  }

  // Use the secret in process only. Do not log or return the raw value.
  return {
    ok: true,
    retrievedSecretCount: Object.keys(secrets).length,
    primarySecretLoaded: true,
  };
};
```

**API Methods:**

- `tailor.secretmanager.getSecrets(vaultName, secretNames)`: Retrieves multiple secrets from a vault
- `tailor.secretmanager.getSecret(vaultName, secretName)`: Retrieves a single secret from a vault

### Pipeline and Executor Integration

Secrets can be referenced in Pipeline resolvers and Executor configurations to securely access external services without hardcoding credentials.

## Security considerations

Secret values are write-only when using Terraform. Once stored, they cannot be retrieved through Terraform state files, ensuring your sensitive data remains secure.

- Secrets are encrypted at rest and in transit
- Access to secrets is controlled through workspace permissions
- Avoid exposing secret values in logs, API responses, or configuration files
- Use descriptive names for secrets to make them easily identifiable
- Regularly rotate sensitive credentials and update secret values accordingly

## Best practices

1. **Organize secrets by environment**: Use separate vaults for development, staging, and production environments
2. **Use descriptive naming**: Choose clear, consistent naming conventions for your secrets
3. **Limit access**: Only grant access to secrets that are necessary for each service
4. **Regular rotation**: Implement a process for regularly updating sensitive credentials
5. **Version management**: Use the versioning feature to track changes to secret values
