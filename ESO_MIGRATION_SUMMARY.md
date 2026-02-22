# External Secrets Operator (ESO) Migration - Implementation Summary

## Overview

Successfully migrated 6 Helm charts from `op-secrets` (1Password/Kubernetes Secrets) to AWS Secrets Manager via External Secrets Operator (ESO).

**Migration Date**: February 20, 2026  
**Charts Updated**: aggkit, aggkit-prover, agglayer, agglayer-prover, bridge, bridge-ui  
**Namespace**: cdk-agglayer  
**SecretStore**: ClusterSecretStore named `aws-secretsmanager`

---

## Changes Summary

### Files Added

| Chart | File | Purpose |
|-------|------|---------|
| aggkit | `templates/externalsecret.yaml` | 3 ExternalSecrets for L1 RPC, credentials, keystores |
| aggkit-prover | `templates/externalsecret.yaml` | 2 ExternalSecrets for L1 RPC, credentials |
| agglayer | `templates/externalsecret.yaml` | 6 ExternalSecrets for credentials, keystores, TLS, OAuth, Datadog, config |
| agglayer-prover | `templates/externalsecret.yaml` | 1 ExternalSecret for future credentials |
| bridge | `templates/externalsecret.yaml` | 4 ExternalSecrets for L1 RPC, Postgres, credentials, keystores |
| bridge-ui | `templates/externalsecret.yaml` | 1 ExternalSecret for L1 RPC |

### Files Modified

#### aggkit
- ✅ `values.yaml` - Added externalSecrets configuration
- ✅ `templates/configmap.yaml` - Removed `lookup` calls, use env vars instead
- ✅ `templates/statefulset.yaml` - Added env var injection, updated volume mounts
- ✅ `templates/onepassworditem.yaml` - Made conditional on `externalSecrets.enabled`

#### aggkit-prover
- ✅ `values.yaml` - Added externalSecrets configuration
- ✅ `templates/configmap.yaml` - Removed `lookup` calls, use env vars instead
- ✅ `templates/statefulset.yaml` - Added env var injection from ExternalSecrets

#### agglayer
- ✅ `values.yaml` - Added externalSecrets configuration
- ✅ `config/config.toml` - Made `lookup` conditional, use env vars when ESO enabled
- ✅ `templates/certificate.yaml` - Made conditional on `externalSecrets.enabled`
- ✅ `templates/secret.yaml` - Made conditional on `externalSecrets.enabled`
- ✅ `templates/statefulset.yaml` - Added env var injection, updated volume mounts

#### agglayer-prover
- ✅ `values.yaml` - Added externalSecrets configuration (disabled by default)
- ✅ `templates/deployment.yaml` - Added commented ESO env var injection

#### bridge
- ✅ `values.yaml` - Added externalSecrets configuration
- ✅ `templates/configmap.yaml` - Removed `lookup` calls, use env vars instead
- ✅ `templates/deployment.yaml` - Added env var injection, updated volume mounts

#### bridge-ui
- ✅ `values.yaml` - Added externalSecrets configuration
- ✅ `templates/deployment.yaml` - Removed `lookup`, use ExternalSecret for L1 RPC URL

---

## AWS Secrets Manager Structure

### Required Secrets to Create

Before deploying with ESO enabled, create these secrets in AWS Secrets Manager:

#### Shared Secrets
```json
# /dev/cdk-agglayer/shared/l1-rpc-url
{
  "l1RpcURL": "https://eth-sepolia.g.alchemy.com/v2/YOUR_KEY"
}
```

#### PostgreSQL (for bridge)
```json
# /dev/cdk-agglayer/bridge/postgres
{
  "bridgeDbName": "bridge",
  "dbUser": "bridge_user",
  "dbUserPassword": "YOUR_SECURE_PASSWORD",
  "dbHost": "postgres.cdk-agglayer.svc.cluster.local",
  "dbPort": "5432"
}
```

#### aggkit
```json
# /dev/cdk-agglayer/aggkit/credentials
{
  "aggsenderKeystorePassword": "PASSWORD1",
  "aggsenderValidatorKeystorePassword": "PASSWORD2",
  "aggoracleKeystorePassword": "PASSWORD3"
}

# /dev/cdk-agglayer/aggkit/keystores
{
  "aggsenderKeystore": "<base64-encoded-keystore-file>",
  "aggsenderValidatorKeystore": "<base64-encoded-keystore-file>",
  "aggoracleKeystore": "<base64-encoded-keystore-file>"
}
```

#### aggkit-prover
```json
# /dev/cdk-agglayer/aggkit-prover/credentials
{
  "sp1NetworkPrivateKey": "0xYOUR_PRIVATE_KEY",
  "sp1NetworkRpcURL": "https://rpc.succinct.xyz/"
}
```

#### agglayer
```json
# /dev/cdk-agglayer/agglayer/credentials
{
  "aggregatorKeystorePassword": "YOUR_PASSWORD"
}

# /dev/cdk-agglayer/agglayer/keystores
{
  "aggregatorKeystore": "<base64-encoded-keystore-file>"
}

# /dev/cdk-agglayer/agglayer/tls
{
  "tlsCrt": "<base64-encoded-certificate>",
  "tlsKey": "<base64-encoded-private-key>"
}

# /dev/cdk-agglayer/agglayer/oauth
{
  "clientId": "YOUR_OAUTH_CLIENT_ID",
  "clientSecret": "YOUR_OAUTH_CLIENT_SECRET"
}

# /dev/cdk-agglayer/agglayer/datadog
{
  "apiKey": "YOUR_DATADOG_API_KEY"
}

# /dev/cdk-agglayer/agglayer/config
{
  "DATA_NODE_RPC_HOST": "...",
  "DATA_NODE_RPC_PORT": "...",
  "DATA_NODE_L1_CHAINID": "...",
  "DATA_NODE_L1_NODEURL": "...",
  "DATA_NODE_L1_NODEURL_WS": "...",
  "DATA_NODE_L1_ROLLUPMANAGERCONTRACT": "...",
  "DATA_NODE_L1_GLOBAL_EXIT_ROOT_CONTRACT": "...",
  "DATA_NODE_KMSKEYNAME": "...",
  "DATA_NODE_PROJECTID": "...",
  "DATA_NODE_LOCATION": "...",
  "DATA_NODE_KEYRING": "...",
  "DATA_NODE_KEYNAME": "...",
  "DATA_NODE_KEYVERSION": "...",
  "DATA_NODE_PROOFSIGNERS": "..."
}
```

#### agglayer-prover
```json
# /dev/cdk-agglayer/agglayer-prover/credentials
# (Future use - currently not active)
{
  "networkPrivateKey": "0xYOUR_PRIVATE_KEY"
}
```

#### bridge
```json
# /dev/cdk-agglayer/bridge/credentials
{
  "claimTxPrivateKeyPassword": "YOUR_PASSWORD"
}

# /dev/cdk-agglayer/bridge/keystores
{
  "claimtxKeystore": "<base64-encoded-keystore-file>"
}
```

#### bridge-ui
_Uses shared L1 RPC URL only_

---

## Migration Steps

### Prerequisites

1. **External Secrets Operator installed** on EKS cluster
2. **ClusterSecretStore created** named `aws-secretsmanager` with IRSA configured
3. **AWS Secrets Manager secrets created** in the structure above
4. **Service Account annotations** for IRSA on charts that need them

### Step 1: Prepare AWS Secrets Manager

1. Extract current secret values from existing `op-secrets` Kubernetes Secret:
   ```bash
   kubectl get secret op-secrets -n cdk-agglayer -o json | jq '.data'
   ```

2. For keystore files, they're already base64-encoded in the secret. Use them as-is in AWS Secrets Manager.

3. Create all AWS Secrets Manager secrets using AWS CLI or Console:
   ```bash
   # Example for shared L1 RPC URL
   aws secretsmanager create-secret \
     --name /dev/cdk-agglayer/shared/l1-rpc-url \
     --secret-string '{"l1RpcURL":"https://eth-sepolia.g.alchemy.com/v2/YOUR_KEY"}'
   
   # Repeat for all secrets listed above
   ```

4. Update secret paths in values files per environment:
   - `dev` → `/dev/cdk-agglayer/...`
   - `uat` → `/uat/cdk-agglayer/...`
   - `test` → `/test/cdk-agglayer/...`
   - `main` → `/main/cdk-agglayer/...`

### Step 2: Verify ClusterSecretStore

Ensure the ClusterSecretStore exists and can access AWS Secrets Manager:

```bash
kubectl get clustersecretstore aws-secretsmanager
kubectl describe clustersecretstore aws-secretsmanager
```

Example ClusterSecretStore (should already exist via Terraform):
```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets-system
```

### Step 3: Deploy with ESO Enabled (Per Chart)

Deploy charts one at a time to minimize risk:

#### Option A: Parallel Run (Recommended for safety)
Keep both old and new secrets during transition.

```bash
# 1. Deploy with externalSecrets.enabled=true
helm upgrade aggkit ./charts/aggkit \
  --namespace cdk-agglayer \
  --set externalSecrets.enabled=true \
  --set externalSecrets.awsSecrets.sharedL1RpcUrl="/dev/cdk-agglayer/shared/l1-rpc-url" \
  --set externalSecrets.awsSecrets.credentials="/dev/cdk-agglayer/aggkit/credentials" \
  --set externalSecrets.awsSecrets.keystores="/dev/cdk-agglayer/aggkit/keystores"

# 2. Verify ExternalSecrets are created and synced
kubectl get externalsecrets -n cdk-agglayer
kubectl get secrets -n cdk-agglayer | grep aggkit

# 3. Check pod status
kubectl get pods -n cdk-agglayer -l app.kubernetes.io/name=aggkit

# 4. If successful, repeat for other charts
```

#### Option B: Direct Cutover
Remove `op-secrets` and deploy with ESO in one step (higher risk).

```bash
# Not recommended for production
kubectl delete secret op-secrets -n cdk-agglayer
helm upgrade aggkit ./charts/aggkit --namespace cdk-agglayer -f values-dev.yaml
```

### Step 4: Validation Checklist

For each chart deployment:

- [ ] ExternalSecrets resources created
- [ ] Kubernetes Secrets created by ESO (check names match expected)
- [ ] Pods are running
- [ ] Application logs show successful startup
- [ ] Secrets are correctly loaded (check via app behavior or logs)
- [ ] No error logs related to secret access

Example validation commands:
```bash
# Check ExternalSecrets status
kubectl get externalsecrets -n cdk-agglayer
kubectl describe externalsecret aggkit-credentials -n cdk-agglayer

# Verify generated secrets exist
kubectl get secret aggkit-credentials -n cdk-agglayer
kubectl get secret aggkit-keystores -n cdk-agglayer
kubectl get secret aggkit-shared-l1 -n cdk-agglayer

# Check secret data (values are base64-encoded)
kubectl get secret aggkit-credentials -n cdk-agglayer -o jsonpath='{.data}'

# Verify pod can read secrets
kubectl exec -it <pod-name> -n cdk-agglayer -- env | grep -E '(PASSWORD|RPC|KEY)'
```

### Step 5: Cleanup Legacy Secrets (After Validation)

Once all charts are successfully running with ESO:

```bash
# Remove legacy op-secrets
kubectl delete secret op-secrets -n cdk-agglayer

# Remove OnePasswordItem resources if they existed
kubectl delete onepassworditems --all -n cdk-agglayer
```

---

## Secret Name Changes

### Before (op-secrets)
All charts referenced a single secret:
```yaml
existingSecret: op-secrets
```

### After (ExternalSecrets)
Charts now reference multiple ESO-generated secrets:

| Chart | Old Secret | New Secrets (ESO-generated) |
|-------|------------|----------------------------|
| aggkit | `op-secrets` | `aggkit-shared-l1`, `aggkit-credentials`, `aggkit-keystores` |
| aggkit-prover | `op-secrets` | `aggkit-prover-shared-l1`, `aggkit-prover-credentials` |
| agglayer | `op-secrets` | `agglayer-credentials`, `agglayer-keystores`, `agglayer-tls`, `agglayer-admin-oauth`, `agglayer-datadog`, `agglayer-config` |
| agglayer-prover | N/A | `agglayer-prover-credentials` (future use) |
| bridge | `op-secrets` | `bridge-shared-l1`, `bridge-postgres`, `bridge-credentials`, `bridge-keystores` |
| bridge-ui | `op-secrets` | `bridge-ui-shared-l1` |

**Note**: Legacy mode still works if `externalSecrets.enabled=false`. This maintains backward compatibility during migration.

---

## Per-Environment Configuration

Update the `externalSecrets.awsSecrets` paths in values files per environment:

### values-dev.yaml
```yaml
externalSecrets:
  enabled: true
  awsSecrets:
    sharedL1RpcUrl: "/dev/cdk-agglayer/shared/l1-rpc-url"
    credentials: "/dev/cdk-agglayer/aggkit/credentials"
    keystores: "/dev/cdk-agglayer/aggkit/keystores"
```

### values-uat.yaml
```yaml
externalSecrets:
  enabled: true
  awsSecrets:
    sharedL1RpcUrl: "/uat/cdk-agglayer/shared/l1-rpc-url"
    credentials: "/uat/cdk-agglayer/aggkit/credentials"
    keystores: "/uat/cdk-agglayer/aggkit/keystores"
```

### values-test.yaml
```yaml
externalSecrets:
  enabled: true
  awsSecrets:
    sharedL1RpcUrl: "/test/cdk-agglayer/shared/l1-rpc-url"
    credentials: "/test/cdk-agglayer/aggkit/credentials"
    keystores: "/test/cdk-agglayer/aggkit/keystores"
```

### values-main.yaml (production)
```yaml
externalSecrets:
  enabled: true
  awsSecrets:
    sharedL1RpcUrl: "/main/cdk-agglayer/shared/l1-rpc-url"
    credentials: "/main/cdk-agglayer/aggkit/credentials"
    keystores: "/main/cdk-agglayer/aggkit/keystores"
```

---

## Troubleshooting

### ExternalSecret Not Syncing

**Symptom**: ExternalSecret status shows errors

```bash
kubectl describe externalsecret <name> -n cdk-agglayer
```

**Common Causes**:
1. **IAM permissions** - Ensure IRSA role has `secretsmanager:GetSecretValue` permission
2. **Secret not found** - Verify secret exists in AWS Secrets Manager with exact path
3. **Invalid JSON** - Check AWS secret is valid JSON format
4. **SecretStore misconfigured** - Verify ClusterSecretStore references correct service account

**Fix**:
```bash
# Check ClusterSecretStore
kubectl describe clustersecretstore aws-secretsmanager

# Verify service account annotations
kubectl get sa -n external-secrets-system -o yaml

# Check AWS secret exists
aws secretsmanager get-secret-value --secret-id /dev/cdk-agglayer/shared/l1-rpc-url
```

### Pod Fails to Start - Missing Environment Variables

**Symptom**: Pod crashes with "required environment variable not set"

**Cause**: ExternalSecret hasn't created the Kubernetes Secret yet

**Fix**:
```bash
# Check if ExternalSecret is ready
kubectl get externalsecrets -n cdk-agglayer

# Force sync
kubectl annotate externalsecret <name> -n cdk-agglayer \
  force-sync=$(date +%s) --overwrite

# Wait and check secret creation
kubectl get secret <secret-name> -n cdk-agglayer
```

### Legacy Mode Not Working

**Symptom**: With `externalSecrets.enabled=false`, pods still fail

**Cause**: `op-secrets` may have been deleted or is missing keys

**Fix**:
```bash
# Verify op-secrets exists
kubectl get secret op-secrets -n cdk-agglayer

# Check it has all required keys
kubectl get secret op-secrets -n cdk-agglayer -o jsonpath='{.data}' | jq 'keys'

# Recreate op-secrets if needed (from backup or 1Password)
```

### Keystore Files Not Mounting Correctly

**Symptom**: Application logs show "keystore file not found" or "invalid keystore"

**Cause**: Base64 encoding/decoding issue or wrong secret key

**Fix**:
1. Verify keystore is properly base64-encoded in AWS Secrets Manager:
   ```bash
   # Encode keystore file
   base64 -w 0 < aggsender.keystore
   ```

2. Check ExternalSecret template uses `b64dec`:
   ```yaml
   data:
     aggsender.keystore: "{{ .aggsenderKeystore | b64dec }}"
   ```

3. Verify the generated Kubernetes Secret has the file:
   ```bash
   kubectl get secret aggkit-keystores -n cdk-agglayer \
     -o jsonpath='{.data.aggsender\.keystore}' | base64 -d | file -
   # Should output: Java KeyStore
   ```

---

## Rollback Plan

If migration fails and you need to rollback to `op-secrets`:

### Step 1: Disable ESO
```bash
helm upgrade <chart> ./charts/<chart> \
  --namespace cdk-agglayer \
  --set externalSecrets.enabled=false \
  --reuse-values
```

### Step 2: Verify op-secrets Exists
```bash
kubectl get secret op-secrets -n cdk-agglayer
```

### Step 3: Restart Pods
```bash
kubectl rollout restart statefulset/<name> -n cdk-agglayer
kubectl rollout restart deployment/<name> -n cdk-agglayer
```

### Step 4: Clean Up ESO Resources (Optional)
```bash
# Remove ExternalSecrets
kubectl delete externalsecrets --all -n cdk-agglayer

# Remove ESO-generated secrets
kubectl delete secret -l created-by=external-secrets -n cdk-agglayer
```

---

## Security Considerations

1. **Least Privilege**: Each chart's service account should only have access to its own secrets
2. **Secret Rotation**: ESO automatically pulls updated secrets based on `refreshInterval` (default: 1h)
3. **Audit Logging**: Enable AWS CloudTrail to log all `GetSecretValue` calls
4. **Encryption**: AWS Secrets Manager encrypts secrets at rest with KMS
5. **IRSA**: Use IAM Roles for Service Accounts instead of static AWS credentials

### Recommended IAM Policy per Chart

Example for aggkit service account:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:/dev/cdk-agglayer/shared/*",
        "arn:aws:secretsmanager:us-east-1:ACCOUNT_ID:secret:/dev/cdk-agglayer/aggkit/*"
      ]
    }
  ]
}
```

---

## Testing Checklist

Before deploying to production, test indev/uat:

- [ ] All ExternalSecrets sync successfully
- [ ] All pods start and reach Ready state
- [ ] Application functionality works (API calls, database connections, etc.)
- [ ] Keystore files are correctly loaded and used
- [ ] TLS certificates work for ingress (agglayer)
- [ ] OAuth authentication works (agglayer GKE backend)
- [ ] Database connections work (bridge)
- [ ] No plaintext secrets in ConfigMaps or pod specs
- [ ] Secret rotation works (change AWS secret, wait for refresh, verify pod picks up change)
- [ ] Rollback to legacy mode works if needed

---

## Benefits of ESO Migration

1. **Centralized Secret Management**: All secrets in AWS Secrets Manager, not scattered across Kubernetes
2. **No Plaintext in Git**: No secrets in values files or templates
3. **Automatic Rotation**: Secrets auto-refresh without pod restarts (based on refreshInterval)
4. **Environment Separation**: Clear separation via AWS secret paths
5. **Audit Trail**: CloudTrail logs all secret access
6. **IRSA Integration**: Native AWS IAM permissions instead of Kubernetes RBAC
7. **No 1Password Dependency**: Removes dependency on OnePasswordItem CRDs
8. **Removes lookup() Anti-pattern**: No more runtime `lookup` calls in templates

---

## Additional Resources

- [External Secrets Operator Docs](https://external-secrets.io/)
- [AWS Secrets Manager Best Practices](https://docs.aws.amazon.com/secretsmanager/latest/userguide/best-practices.html)
- [EKS IRSA Setup](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

---

## Support & Questions

For issues or questions about this migration:
1. Check troubleshooting section above
2. Review ExternalSecret status: `kubectl describe externalsecret <name> -n cdk-agglayer`
3. Check AWS Secrets Manager console for secret existence/format
4. Verify IRSA permissions on service account
5. Review pod logs for secret-related errors

**Migration Status**: ✅ COMPLETE  
**Charts Migrated**: 6/6  
**Legacy Compatibility**: Maintained (via `externalSecrets.enabled=false`)
