# Secret Management Documentation

## Overview

This document outlines the secret management strategy for the finch application deployed on Kubernetes. It covers how secrets are created, stored, accessed, and managed securely.

## Secret Management Strategy

### Kubernetes Native Secrets

The primary approach uses Kubernetes Secrets for storing sensitive information. While Kubernetes Secrets are base64 encoded (not encrypted by default), they provide a foundation for secret management.

**Location**: `kubernetes/secrets/`

### Secret Types

1. **Database Credentials** (`db-credentials.yaml`)
   - PostgreSQL username
   - PostgreSQL password
   - Database name
   - Database connection URL

2. **API Keys** (`api-keys.yaml`)
   - External API keys
   - Service authentication tokens
   - Third-party service credentials

## Creating Secrets

### Method 1: Using YAML Manifests (Current Approach)

Secrets are defined in YAML files with base64-encoded values.

**Important**: Replace placeholder values with actual secrets before applying.

#### Encoding Values

To encode a value in base64:

```bash
echo -n 'your-secret-value' | base64
```

Example:
```bash
echo -n 'finch_password' | base64
# Output: ZmluY2hfcGFzc3dvcmQ=
```

#### Decoding Values (for verification)

```bash
echo 'ZmluY2hfcGFzc3dvcmQ=' | base64 -d
# Output: finch_password
```

#### Creating Database Credentials Secret

1. Encode your values:
   ```bash
   echo -n 'finch_user' | base64
   echo -n 'your_secure_password' | base64
   echo -n 'finch_db' | base64
   echo -n 'postgresql://finch_user:your_secure_password@finch-postgresql-service:5432/finch_db' | base64
   ```

2. Update `kubernetes/secrets/db-credentials.yaml` with encoded values

3. Apply the secret:
   ```bash
   kubectl apply -f kubernetes/secrets/db-credentials.yaml
   ```

### Method 2: Using kubectl Command

Create secrets directly using kubectl:

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=finch_user \
  --from-literal=password=your_secure_password \
  --from-literal=database-name=finch_db \
  --from-literal=database-url=postgresql://finch_user:your_secure_password@finch-postgresql-service:5432/finch_db
```

### Method 3: Using kubectl from File

Create secret from a file:

```bash
kubectl create secret generic db-credentials \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt
```

## Accessing Secrets in Pods

### Environment Variables

Secrets are accessed via environment variables in deployment manifests:

```yaml
env:
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: database-url
```

### Volume Mounts

Secrets can also be mounted as files:

```yaml
volumeMounts:
- name: secrets
  mountPath: /etc/secrets
  readOnly: true
volumes:
- name: secrets
  secret:
    secretName: db-credentials
```

## Updating Secrets

### Method 1: Update YAML and Reapply

1. Update the secret YAML file with new base64-encoded values
2. Apply the updated secret:
   ```bash
   kubectl apply -f kubernetes/secrets/db-credentials.yaml
   ```
3. Restart pods to pick up new secrets:
   ```bash
   kubectl rollout restart deployment finch-backend
   kubectl rollout restart deployment finch-postgresql
   ```

### Method 2: Using kubectl patch

```bash
kubectl patch secret db-credentials -p '{"data":{"password":"'$(echo -n 'new_password' | base64)'"}}'
```

### Method 3: Delete and Recreate

```bash
kubectl delete secret db-credentials
kubectl create secret generic db-credentials \
  --from-literal=username=finch_user \
  --from-literal=password=new_password
```

## Viewing Secrets

### List Secrets

```bash
kubectl get secrets
```

### View Secret Details

```bash
kubectl describe secret db-credentials
```

### Decode Secret Values

```bash
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

## Security Best Practices

### 1. Never Commit Secrets to Git

- Use `.gitignore` to exclude secret files
- Use placeholder values in version control
- Document the process for setting real secrets

### 2. Use External Secret Managers (Production)

For production environments, consider:

#### HashiCorp Vault

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com"
    roleName: "finch-role"
    objects: |
      - objectName: "db-credentials"
        secretPath: "secret/data/finch/db"
        secretKey: "password"
```

#### AWS Secrets Manager

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-db-creds
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "finch-db-password"
        objectType: "secretsmanager"
```

#### Azure Key Vault

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-db-creds
spec:
  provider: azure
  parameters:
    keyvaultName: "finch-keyvault"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
```

### 3. Enable Encryption at Rest

For production clusters, enable encryption at rest for etcd:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-secret>
```

### 4. Use RBAC

Limit access to secrets using Role-Based Access Control:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials"]
  verbs: ["get", "list"]
```

### 5. Rotate Secrets Regularly

Implement a secret rotation policy:
- Rotate database passwords quarterly
- Rotate API keys monthly
- Rotate immediately if compromised

## Secret Rotation Strategy

### Automated Rotation

1. **CronJob for Rotation**:
   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: rotate-secrets
   spec:
     schedule: "0 0 1 * *"  # First day of every month
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: rotate
               image: secret-rotator:latest
               command: ["/rotate-secrets.sh"]
   ```

2. **Update Secrets**:
   - Generate new credentials
   - Update Kubernetes secrets
   - Update application configurations
   - Restart affected pods

### Manual Rotation Process

1. Generate new secret values
2. Update secret in Kubernetes
3. Update application configuration if needed
4. Restart pods to pick up new secrets
5. Verify application functionality
6. Remove old secret values

## Troubleshooting

### Secret Not Found

```bash
# Verify secret exists
kubectl get secret db-credentials

# Check if secret is in correct namespace
kubectl get secret db-credentials -n <namespace>
```

### Pod Cannot Access Secret

1. Check pod has permission to read secret
2. Verify secret key names match
3. Check RBAC policies
4. Verify secret is in same namespace as pod

### Secret Value Incorrect

1. Decode and verify secret value:
   ```bash
   kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
   ```
2. Update secret if incorrect
3. Restart pods

## Current Secret Structure

### db-credentials.yaml

Contains:
- `username`: PostgreSQL username
- `password`: PostgreSQL password
- `database-name`: Database name
- `database-url`: Full connection string

### api-keys.yaml

Contains:
- `external-api-key`: External service API key
- Additional keys as needed

## Migration to External Secret Managers

### Steps for Migration

1. **Choose Secret Manager**: Vault, AWS Secrets Manager, Azure Key Vault, etc.
2. **Install CSI Driver**: Install appropriate CSI driver for your secret manager
3. **Create SecretProviderClass**: Define how to fetch secrets
4. **Update Deployments**: Modify deployments to use CSI volumes
5. **Test**: Verify secrets are accessible
6. **Remove Kubernetes Secrets**: Delete old Kubernetes secrets

### Example Migration to Vault

```yaml
# Install Vault CSI Driver
kubectl apply -f https://raw.githubusercontent.com/hashicorp/vault-csi-provider/main/deployment/vault-csi-provider.yaml

# Create SecretProviderClass
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com"
    roleName: "finch-role"
    objects: |
      - objectName: "db-credentials"
        secretPath: "secret/data/finch/db"
        secretKey: "password"
```

## Best Practices Summary

1. ✅ Use Kubernetes Secrets for development
2. ✅ Use external secret managers for production
3. ✅ Never commit secrets to version control
4. ✅ Enable encryption at rest
5. ✅ Implement RBAC for secret access
6. ✅ Rotate secrets regularly
7. ✅ Use least privilege principle
8. ✅ Monitor secret access
9. ✅ Audit secret changes
10. ✅ Document secret management process

## Next Steps

1. Set up external secret manager for production
2. Implement secret rotation automation
3. Configure RBAC policies
4. Set up secret access auditing
5. Document production secret management procedures
