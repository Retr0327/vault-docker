# **vault-docker**

This guide documents the approach I've developed for deploying and configuring Vault with MySQL using Docker.

## **Documentation**

### 1. Run Containers

To set up and run, make sure you have Git and Docker installed on your system. Then run the command for your system:

```bash
git clone git@github.com:Retr0327/vault-docker.git && cd vault-docker && docker compose up -d
```

### 2. MySQL Storage Configuration

Create an `.env` file under the `vault-docker/` directory with the following content:

```bash
MYSQL_ROOT_PASSWORD=<root-password>
MYSQL_DATABASE=<database-name>
MYSQL_USER=<username>
MYSQL_PASSWORD=<password>
```

Then adjust the `vault-docker/vault/config/vault-config.json` with the corresponding values:

```json
{
  "storage": {
    "mysql": {
      "username": "<username>",
      "password": "<password>",
      "database": "<database-name>",
      "address": "mysql:3306",
      "table": "vault_data",
      "ha_enabled": "true",
      "lock_table": "vault_lock"
    }
  },
  "listener": {
    "tcp": {
      "address": "0.0.0.0:8200",
      "tls_disable": 1
    }
  },
  "ui": true,
  "api_addr": "http://0.0.0.0:8200",
  "disable_mlock": true
}
```

### 3. Vault Initialization

After deployment, you need to initialize the Vault to generate the unseal keys and root token:

1. Access the Vault container:

   ```bash
   # Find the container ID
   docker ps

   # Enter the container
   docker exec -it <container_id> sh
   ```

2. Initialize Vault:

   ```bash
   vault operator init
   ```

   The command will output five unseal keys and a root token:

   > Store these keys and the root token securely. They are critical for vault administration and recovery

   ```bash
   Unseal Key 1: shAqI1lC8HRoPTsSbd7ZlV83opt1HNfUxoiZT80gm6VV
   Unseal Key 2: xiKtgeFsuNaaBNbGxg1sAAONNsbB9q3vPy+xbHG1XTdI
   Unseal Key 3: usvM8vj8GFKUH6Hxykcl3GctThdEtLqdhi4vZUaphEHq
   Unseal Key 4: AVr6v9mN/tQslgOirxOJx3G6s8IZChqMkSJ4keGagZoU
   Unseal Key 5: djjjO+rmU+nzMrv0ZMRVvVbOhBoPG0x6HvTJ3xovk3cO

   Initial Root Token: hvs.wYY1sJcgAlaDQJAoyTSVCMWW
   ```

3. Unseal the Vault using three of the five unseal keys:

   ```bash
   vault operator unseal <unseal-key-1>
   vault operator unseal <unseal-key-2>
   vault operator unseal <unseal-key-3>
   ```

4. Authentication

   For the first-time setup, you MUST use the root token to login and configure the system:

   - via CLI

     ```bash
     vault login <root-token>
     ```

   - via GUI
     - Access the Vault UI by navigating to http://localhost:8200
     - Enter the root token in the login screen

   After initial setup and user creation, you can authenticate using either root token or username/password credentials:

   - via CLI

     ```bash
       # Using the root token (for administrative tasks)
       vault login <root-token>

       # OR using username/password credentials
       vault login -method=userpass username=<username>
     ```

   - via GUI
     - Access the Vault UI by navigating to http://localhost:8200
     - Enter the root token or username/password credentials in the login screen

### 4. ACL (Access Control Configuration) Setup

There are various ways to setup up your customized ACL (see [vault-policy-guide](https://github.com/jeffsanicola/vault-policy-guide) for more information). Here is how I configure my ACL policies to separate admin and standard user roles at http://localhost:8200/ui/vault/policies/acl/create:

> In this sceanrio, a user can only view the path `secrets/` under http://localhost:8200/ui/vault/secrets

1.  Admin Policy

    ```hcl
    # Allow management of all auth methods
    path "auth/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    # System management
    path "sys/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    # Policy management
    path "sys/policies/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    # Allow listing all mount points and auth methods
    path "sys/mounts/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    path "sys/auth/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    # Full access to identity management
    path "identity/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    # Access to audit logs
    path "sys/audit/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    # Access to any other secret engines that might be mounted
    path "+/*" {
      capabilities = ["create", "read", "update", "delete", "list", "sudo"]
    }

    # User management capabilities
    path "auth/userpass/users/*" {
      capabilities = ["update", "create"]
    }
    ```

2.  Standard User Policy

    ```hcl
    # Allow read and update operations on secrets under the KV v2 secrets/ path
    path "secrets/data/*" {
      capabilities = ["create", "read", "update", "list"]
    }

    # Allow listing and viewing metadata for secrets
    path "secrets/metadata/*" {
      capabilities = ["list", "read"]
    }

    # Allow deleting secret versions but not the entire secret
    path "secrets/delete/*" {
      capabilities = ["update"]
    }

    # Allow undeleting secret versions
    path "secrets/undelete/*" {
      capabilities = ["update"]
    }

    # Allow users to update their own passwords
    path "auth/userpass/users/{{identity.entity.aliases.<accessor>.name}}/password" {
      capabilities = ["update"]
    }
    ```

    To determine the correct `<accessor>` value for the user policy, run:

    ```bash
    vault auth list
    ```

    This prints:

    ```bash
    Path         Type        Accessor                  Description                Version
    ----         ----        --------                  -----------                -------
    token/       token       auth_token_1bac132b       token based credentials    n/a
    userpass/    userpass    auth_userpass_3e11533d    n/a                        n/a
    ```

### 5. User Management

First, enable the userpass authentication method:

```bash
vault auth enable userpass
```

1. Creating Users

   To create a user, run:

   ```bash
   vault write auth/userpass/users/<username> password=<password> policies=<policy>
   ```

   For example:

   ```bash
   vault write auth/userpass/users/retr0 password=pass policies=user
   ```

2. Updating User Passwords

   As an administrator, you can update any user's password:

   ```bash
   # Method 1
   vault write auth/userpass/users/john password="newPassword123!"

   # Method 2
   vault write auth/userpass/users/john/password password="newPassword123!"
   ```

   Standard users can only update their own passwords after authenticating:

   ```bash
   vault write auth/userpass/users/john/password password="newPassword123!"
   ```

## Contact Me

If you have any suggestion or question, please do not hesitate to email me at retr0327.dev@gmail.com
