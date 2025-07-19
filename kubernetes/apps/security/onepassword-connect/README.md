# 1Password Connect Setup

This directory contains the configuration for 1Password Connect, which enables secure integration between your Kubernetes cluster and 1Password for secrets management using Service Account tokens.

## Prerequisites

1. A 1Password account with a team or business plan
2. Access to create a 1Password Service Account
3. The `op` CLI tool installed and configured (optional)

## Setup Steps

### 1. Create 1Password Service Account

1. Log into your 1Password account
2. Go to **Integrations** → **1Password Connect**
3. Create a new integration
4. Copy the JWT token that's generated

### 2. Encrypt Token with SOPS

Replace the placeholder values in `secret.sops.yaml`:

```bash
# Create a temporary file with your token
cat > temp-secret.yaml <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: onepassword-secret
type: Opaque
stringData:
  token: "YOUR_JWT_TOKEN_HERE"
EOF

# Encrypt with SOPS
sops --encrypt --in-place temp-secret.yaml

# Replace the content in secret.sops.yaml
cp temp-secret.yaml secret.sops.yaml

# Clean up
rm temp-secret.yaml
```

### 3. Create Vault and Items

In your 1Password account, create:
- A vault named "Kubernetes" (or update the vault reference in clustersecretstore.yaml)
- Secret items for your applications

### 4. Deploy

The configuration will be automatically deployed by Flux once the secret is properly encrypted.

## Verification

Check deployment status:

```bash
kubectl get pods -n security
kubectl get clustersecretstore
kubectl get externalsecrets -A
```

## Usage

Create ExternalSecret resources to sync 1Password items to Kubernetes secrets:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: home-assistant-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: onepassword-connect
    kind: ClusterSecretStore
  target:
    name: home-assistant-secret
    creationPolicy: Owner
  data:
    - secretKey: api-token
      remoteRef:
        key: home-assistant
        property: api-token
```