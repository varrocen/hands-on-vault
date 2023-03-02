# Hands-on HachiCorp Vault

## Setup (macOS)

Install requirements:

```
brew install openssl jq
```

Install HachiCorp Vault:

```
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

Start a Vault server in "dev" mode:

```
vault server -dev
```

Launch a new terminal session.

Copy and run the `export VAULT_ADDR=...` command from the terminal output.

Example:

```
export VAULT_ADDR='http://127.0.0.1:8200'
```

Set the `VAULT_TOKEN` environment variable value to the generated Root Token value displayed in the terminal output.

Example:

```
export VAULT_TOKEN="<ROOT_TOKEN>"
```

Verify the server is running:

```
vault status
```

## Hands-on

 * [PKI Secrets Engine](PKI.md)