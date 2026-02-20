# External Secrets Deployment & Troubleshooting Guide

## Prerequisites Checklist

- [ ] EKS cluster with OIDC provider configured
- [ ] IAM role created with IRSA trust policy for External Secrets Operator
- [ ] IAM policy granting `secretsmanager:GetSecretValue` and `secretsmanager:DescribeSecret` permissions
- [ ] ClusterSecretStore `aws-secretsmanager` deployed (typically via Terraform/IaC)
- [ ] kubectl configured to access your EKS cluster
- [ ] Helm 3.x installed
- [ ] aws CLI configured with appropriate credentials

## Quick Deployment Commands

### 1. Create Secrets in AWS Secrets Manager

```bash
# Set your environment
export ENV="dev"  # or uat, test, main
export AWS_REGION="ap-south-1"

# Aggkit secrets
aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/aggkit/aggsender-keystore" \
  --secret-string '{"aggsenderKeystore":"<base64-encoded-keystore>"}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/aggkit/trusted-sequencer-keystore" \
  --secret-string '{"trustedSequencerKeystore":"<base64-encoded-keystore>"}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/aggkit/keystore-passwords" \
  --secret-string '{"aggsenderKeystorePassword":"...","trustedSequencerKeystorePassword":"..."}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/aggkit/l1-rpc" \
  --secret-string '{"l1RpcUrl":"https://..."}' \
  --region ${AWS_REGION}

# Aggkit-prover secrets
aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/aggkit-prover/network-private-key" \
  --secret-string '{"networkPrivateKey":"0x..."}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/aggkit-prover/l1-rpc" \
  --secret-string '{"l1RpcUrl":"https://..."}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/aggkit-prover/sp1-network-rpc" \
  --secret-string '{"sp1NetworkRpcUrl":"https://..."}' \
  --region ${AWS_REGION}

# Agglayer secrets
aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/agglayer/l1-keystore" \
  --secret-string '{"l1Keystore":"<base64-encoded-keystore>"}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/agglayer/agglayer-keystore" \
  --secret-string '{"agglayerKeystore":"<base64-encoded-keystore>"}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/agglayer/keystore-passwords" \
  --secret-string '{"l1KeystorePassword":"...","agglayerKeystorePassword":"..."}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/agglayer/l1-rpc" \
  --secret-string '{"l1RpcUrl":"https://..."}' \
  --region ${AWS_REGION}

# Agglayer-prover secrets
aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/agglayer-prover/network-private-key" \
  --secret-string '{"networkPrivateKey":"0x..."}' \
  --region ${AWS_REGION}

# Bridge secrets
aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/bridge/postgres" \
  --secret-string '{"user":"bridge_user","password":"...","name":"bridge_db","host":"postgres.cdk-agglayer.svc.cluster.local","port":"5432"}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/bridge/l1-rpc" \
  --secret-string '{"l1RpcUrl":"https://..."}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/bridge/l2-rpc" \
  --secret-string '{"l2RpcUrl":"https://..."}' \
  --region ${AWS_REGION}

aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/bridge/claim-tx-private-key" \
  --secret-string '{"claimTxPrivateKeyPassword":"..."}' \
  --region ${AWS_REGION}

# Bridge-UI secrets
aws secretsmanager create-secret \
  --name "/${ENV}/cdk-agglayer/bridge-ui/l1-rpc" \
  --secret-string '{"l1RpcUrl":"https://..."}' \
  --region ${AWS_REGION}
```

### 2. Deploy Charts with External Secrets Enabled

```bash
# Deploy aggkit
helm install aggkit ./charts/aggkit \
  -n cdk-agglayer \
  --create-namespace \
  --set externalSecrets.enabled=true \
  --set externalSecrets.env=dev \
  -f ./charts/aggkit/values.dev.yaml

# Deploy aggkit-prover
helm install aggkit-prover ./charts/aggkit-prover \
  -n cdk-agglayer \
  --set externalSecrets.enabled=true \
  --set externalSecrets.env=dev \
  -f ./charts/aggkit-prover/values.dev.yaml

# Deploy agglayer
helm install agglayer ./charts/agglayer \
  -n cdk-agglayer \
  --set externalSecrets.enabled=true \
  --set externalSecrets.env=dev \
  -f ./charts/agglayer/values.dev.yaml

# Deploy agglayer-prover
helm install agglayer-prover ./charts/agglayer-prover \
  -n cdk-agglayer \
  --set externalSecrets.enabled=true \
  --set externalSecrets.env=dev \
  -f ./charts/agglayer-prover/values.dev.yaml

# Deploy bridge
helm install bridge ./charts/bridge \
  -n cdk-agglayer \
  --set externalSecrets.enabled=true \
  --set externalSecrets.env=dev \
  -f ./charts/bridge/values.dev.yaml

# Deploy bridge-ui
helm install bridge-ui ./charts/bridge-ui \
  -n cdk-agglayer \
  --set externalSecrets.enabled=true \
  --set externalSecrets.env=dev \
  -f ./charts/bridge-ui/values.local.yaml
```

### 3. Deploy with Legacy Mode (op-secrets)

```bash
# Deploy with externalSecrets.enabled=false
helm install aggkit ./charts/aggkit \
  -n cdk-agglayer \
  --create-namespace \
  --set externalSecrets.enabled=false \
  -f ./charts/aggkit/values.dev.yaml

# Ensure op-secrets secret exists in the namespace
kubectl get secret op-secrets -n cdk-agglayer
```

## Verification Steps

### 1. Verify ClusterSecretStore

```bash
# Check ClusterSecretStore status
kubectl get clustersecretstore aws-secretsmanager

# Should show:
# NAME                  AGE   STATUS   CAPABILITIES   READY
# aws-secretsmanager    5d    Valid    ReadWrite      True

# Detailed status
kubectl describe clustersecretstore aws-secretsmanager

# Verify IRSA ServiceAccount
kubectl get sa external-secrets-sa -n external-secrets-system -o yaml | grep eks.amazonaws.com
```

### 2. Verify ExternalSecret Resources

```bash
# List all ExternalSecrets in namespace
kubectl get externalsecret -n cdk-agglayer

# Check specific ExternalSecret status
kubectl describe externalsecret aggkit-external-secrets -n cdk-agglayer

# Should show:
# Status:
#   Conditions:
#     Status: True
#     Type:   Ready
#   Refresh Time: 2026-02-20T10:30:00Z
#   Synced Resource Version: 1-abc123def456
```

### 3. Verify Generated Kubernetes Secrets

```bash
# List all secrets in namespace
kubectl get secrets -n cdk-agglayer

# Check specific secret exists
kubectl get secret aggkit-secrets -n cdk-agglayer
kubectl get secret aggkit-keystore-secrets -n cdk-agglayer
kubectl get secret agglayer-secrets -n cdk-agglayer
kubectl get secret agglayer-keystore-secrets -n cdk-agglayer
kubectl get secret bridge-secrets -n cdk-agglayer
kubectl get secret bridge-ui-secrets -n cdk-agglayer

# List secret keys (without exposing values)
kubectl get secret aggkit-secrets -n cdk-agglayer -o jsonpath='{.data}' | jq 'keys'

# Expected output for aggkit-secrets:
# [
#   "aggsenderKeystorePassword",
#   "l1RpcUrl",
#   "trustedSequencerKeystorePassword"
# ]

# Verify keystore secret contains binary data
kubectl get secret aggkit-keystore-secrets -n cdk-agglayer -o jsonpath='{.data}' | jq 'keys'

# Expected output:
# [
#   "aggsender.keystore",
#   "trustedSequencer.keystore"
# ]
```

### 4. Verify Pods Are Running

```bash
# Check all pods in namespace
kubectl get pods -n cdk-agglayer

# Check specific pod status
kubectl describe pod aggkit-0 -n cdk-agglayer

# Verify environment variables are injected (without showing values)
kubectl exec -n cdk-agglayer aggkit-0 -- env | grep -E "L1_RPC_URL|AGGSENDER_KEYSTORE_PASSWORD" || echo "Using config file injection"

# Check logs for startup errors
kubectl logs -n cdk-agglayer aggkit-0 --tail=50
```

### 5. Verify Secret Mounting in Pods

```bash
# Check mounted secrets in pod
kubectl exec -n cdk-agglayer aggkit-0 -- ls -la /etc/aggkit/

# Expected output:
# -rw-r--r-- 1 root root <size> <date> aggsender.keystore
# -rw-r--r-- 1 root root <size> <date> trustedSequencer.keystore
# -rw-r--r-- 1 root root <size> <date> config.toml

# Verify keystore files are binary (not base64 text)
kubectl exec -n cdk-agglayer aggkit-0 -- file /etc/aggkit/aggsender.keystore

# Should show: Java KeyStore or similar binary format
```

## Troubleshooting Guide

### Issue 1: ExternalSecret Status Shows "SecretSyncedError"

**Symptoms:**
```bash
kubectl describe externalsecret aggkit-external-secrets -n cdk-agglayer
# Shows:
# Status:
#   Conditions:
#     Message: could not get secret data from provider
#     Status: False
#     Type:   Ready
```

**Diagnosis:**
```bash
# Check External Secrets Operator logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets --tail=100

# Common error messages:
# - "AccessDeniedException: User is not authorized"
# - "ResourceNotFoundException: Secrets Manager can't find the specified secret"
# - "ValidationException: Invalid request"
```

**Solutions:**

1. **Check IAM permissions:**
```bash
# Verify IAM role has correct policy
aws iam get-role-policy --role-name <eso-irsa-role> --policy-name SecretsManagerReadPolicy

# Should include:
# {
#   "Effect": "Allow",
#   "Action": [
#     "secretsmanager:GetSecretValue",
#     "secretsmanager:DescribeSecret"
#   ],
#   "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:/ENV/cdk-agglayer/*"
# }
```

2. **Verify secret exists in AWS:**
```bash
aws secretsmanager describe-secret \
  --secret-id /dev/cdk-agglayer/aggkit/l1-rpc \
  --region ap-south-1
```

3. **Check secret path matches:**
```bash
# Compare ExternalSecret remoteRef with actual AWS path
kubectl get externalsecret aggkit-external-secrets -n cdk-agglayer -o yaml | grep -A 5 remoteRef

# Verify environment variable is set correctly
helm get values aggkit -n cdk-agglayer | grep externalSecrets.env
```

4. **Force secret refresh:**
```bash
kubectl annotate externalsecret aggkit-external-secrets \
  -n cdk-agglayer \
  force-sync=$(date +%s) \
  --overwrite
```

### Issue 2: ClusterSecretStore Status Shows "NotReady"

**Symptoms:**
```bash
kubectl get clustersecretstore aws-secretsmanager
# NAME                  AGE   STATUS   READY
# aws-secretsmanager    5d    Invalid  False
```

**Diagnosis:**
```bash
kubectl describe clustersecretstore aws-secretsmanager

# Check for:
# - "ServiceAccount not found"
# - "IRSA annotation missing"
# - "Failed to authenticate to AWS"
```

**Solutions:**

1. **Verify ServiceAccount exists:**
```bash
kubectl get sa external-secrets-sa -n external-secrets-system
```

2. **Check IRSA annotation:**
```bash
kubectl get sa external-secrets-sa -n external-secrets-system -o yaml

# Should contain:
# metadata:
#   annotations:
#     eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/ROLE_NAME
```

3. **Test AWS access from ESO pod:**
```bash
# Get ESO pod name
ESO_POD=$(kubectl get pods -n external-secrets-system -l app.kubernetes.io/name=external-secrets -o jsonpath='{.items[0].metadata.name}')

# Test AWS credentials
kubectl exec -n external-secrets-system ${ESO_POD} -- env | grep AWS

# Test Secrets Manager access
kubectl exec -n external-secrets-system ${ESO_POD} -- \
  aws secretsmanager list-secrets --region ap-south-1
```

### Issue 3: Pod Crashes with "Config File Error"

**Symptoms:**
```bash
kubectl logs -n cdk-agglayer aggkit-0
# Error: unable to read private key from /etc/aggkit/aggsender.keystore
# Error: incorrect password for keystore
```

**Diagnosis:**
```bash
# Check if keystore files are mounted
kubectl exec -n cdk-agglayer aggkit-0 -- ls -la /etc/aggkit/

# Check if keystore is binary (not base64 text)
kubectl exec -n cdk-agglayer aggkit-0 -- file /etc/aggkit/aggsender.keystore

# Check if password secret exists
kubectl get secret aggkit-secrets -n cdk-agglayer -o jsonpath='{.data.aggsenderKeystorePassword}' | base64 -d
```

**Solutions:**

1. **Verify keystore base64 encoding in AWS:**
```bash
# Get keystore from AWS
aws secretsmanager get-secret-value \
  --secret-id /dev/cdk-agglayer/aggkit/aggsender-keystore \
  --region ap-south-1 \
  --query SecretString \
  --output text | jq -r '.aggsenderKeystore' | base64 -d > /tmp/test.keystore

# Verify it's a valid keystore
file /tmp/test.keystore
# Should show: Java KeyStore or similar

# Test keystore with password
keytool -list -keystore /tmp/test.keystore -storepass <password>
```

2. **Verify ExternalSecret uses b64dec filter:**
```bash
kubectl get externalsecret aggkit-keystore-external-secrets -n cdk-agglayer -o yaml | grep b64dec

# Should show:
# template:
#   data:
#     aggsender.keystore: "{{ .aggsenderKeystore | b64dec }}"
```

3. **Check password matches:**
```bash
# Compare passwords
aws secretsmanager get-secret-value \
  --secret-id /dev/cdk-agglayer/aggkit/keystore-passwords \
  --region ap-south-1 \
  --query SecretString \
  --output text | jq -r '.aggsenderKeystorePassword'

# vs

kubectl get secret aggkit-secrets -n cdk-agglayer \
  -o jsonpath='{.data.aggsenderKeystorePassword}' | base64 -d
```

### Issue 4: Legacy Mode Not Working (externalSecrets.enabled=false)

**Symptoms:**
```bash
kubectl logs -n cdk-agglayer aggkit-0
# Error: environment variable L1_RPC_URL not set
# Error: unable to find config value for AggsenderPrivateKey
```

**Diagnosis:**
```bash
# Check if op-secrets secret exists
kubectl get secret op-secrets -n cdk-agglayer

# Check if secret has required keys
kubectl get secret op-secrets -n cdk-agglayer -o jsonpath='{.data}' | jq 'keys'

# Verify externalSecrets.enabled is false
helm get values aggkit -n cdk-agglayer | grep externalSecrets.enabled
```

**Solutions:**

1. **Create op-secrets if missing:**
```bash
kubectl create secret generic op-secrets \
  -n cdk-agglayer \
  --from-literal=l1-rpc-url='https://...' \
  --from-literal=aggsender-keystore-password='...' \
  --from-literal=trusted-sequencer-keystore-password='...' \
  --from-file=aggsender.keystore=/path/to/aggsender.keystore \
  --from-file=trustedSequencer.keystore=/path/to/trustedSequencer.keystore
```

2. **Verify ConfigMap uses conditional logic:**
```bash
kubectl get configmap aggkit-config -n cdk-agglayer -o yaml

# Should see env var references: ${L1_RPC_URL}, ${AGGSENDER_KEYSTORE_PASSWORD}
# NOT direct values or empty fields
```

3. **Check StatefulSet has else block:**
```bash
kubectl get statefulset aggkit -n cdk-agglayer -o yaml | grep -A 20 "if .Values.externalSecrets.enabled"

# Should have else block with:
# - name: L1_RPC_URL
#   valueFrom:
#     secretKeyRef:
#       name: op-secrets
#       key: l1-rpc-url
```

### Issue 5: Secret Not Updating After AWS Change

**Symptoms:**
```bash
# Updated secret in AWS, but pod still uses old value
aws secretsmanager update-secret \
  --secret-id /dev/cdk-agglayer/aggkit/l1-rpc \
  --secret-string '{"l1RpcUrl":"https://new-endpoint"}' \
  --region ap-south-1

# But pod logs still show old endpoint
```

**Solutions:**

1. **Check ExternalSecret refresh interval:**
```bash
kubectl get externalsecret aggkit-external-secrets -n cdk-agglayer -o yaml | grep refreshInterval

# Default is 1h, can be adjusted:
# spec:
#   refreshInterval: 5m
```

2. **Force immediate refresh:**
```bash
kubectl annotate externalsecret aggkit-external-secrets \
  -n cdk-agglayer \
  force-sync=$(date +%s) \
  --overwrite

# Wait a few seconds
sleep 5

# Verify secret updated
kubectl get secret aggkit-secrets -n cdk-agglayer \
  -o jsonpath='{.data.l1RpcUrl}' | base64 -d
```

3. **Restart pods to pick up new secret:**
```bash
# For StatefulSets
kubectl rollout restart statefulset aggkit -n cdk-agglayer
kubectl rollout restart statefulset agglayer -n cdk-agglayer

# For Deployments
kubectl rollout restart deployment bridge -n cdk-agglayer
kubectl rollout restart deployment bridge-ui -n cdk-agglayer

# Check rollout status
kubectl rollout status statefulset aggkit -n cdk-agglayer
```

### Issue 6: Multiple Components, One Secret Path Issue

**Symptoms:**
```bash
# Bridge can't find postgres credentials
kubectl logs -n cdk-agglayer bridge-xxx
# Error: unable to connect to database
```

**Diagnosis:**
```bash
# Check ExternalSecret path
kubectl get externalsecret bridge-external-secrets -n cdk-agglayer -o yaml | grep -A 3 postgres

# Verify actual AWS secret path
aws secretsmanager list-secrets --region ap-south-1 | grep postgres
```

**Solutions:**

1. **Verify bridge uses component-specific path:**
```bash
# Should be: /dev/cdk-agglayer/bridge/postgres
# NOT: /dev/cdk-agglayer/postgres/credentials

# Update if incorrect
helm upgrade bridge ./charts/bridge \
  -n cdk-agglayer \
  -f ./charts/bridge/values.dev.yaml \
  --reuse-values
```

2. **Ensure AWS secret exists at correct path:**
```bash
aws secretsmanager describe-secret \
  --secret-id /dev/cdk-agglayer/bridge/postgres \
  --region ap-south-1

# If not found, create it:
aws secretsmanager create-secret \
  --name /dev/cdk-agglayer/bridge/postgres \
  --secret-string '{"user":"bridge_user","password":"...","name":"bridge_db","host":"postgres.cdk-agglayer.svc.cluster.local","port":"5432"}' \
  --region ap-south-1
```

## Secret Paths Reference

### Aggkit

| AWS Secret Path | Properties | Kubernetes Secret | Purpose |
|----------------|------------|-------------------|---------|
| `/<env>/cdk-agglayer/aggkit/aggsender-keystore` | `aggsenderKeystore` (base64) | `aggkit-keystore-secrets` | Aggsender private key file |
| `/<env>/cdk-agglayer/aggkit/trusted-sequencer-keystore` | `trustedSequencerKeystore` (base64) | `aggkit-keystore-secrets` | Sequencer private key file |
| `/<env>/cdk-agglayer/aggkit/keystore-passwords` | `aggsenderKeystorePassword`, `trustedSequencerKeystorePassword` | `aggkit-secrets` | Keystore passwords |
| `/<env>/cdk-agglayer/aggkit/l1-rpc` | `l1RpcUrl` | `aggkit-secrets` | L1 RPC endpoint |

### Aggkit-Prover

| AWS Secret Path | Properties | Kubernetes Secret | Purpose |
|----------------|------------|-------------------|---------|
| `/<env>/cdk-agglayer/aggkit-prover/network-private-key` | `networkPrivateKey` | `aggkit-prover-secrets` | Network private key (hex) |
| `/<env>/cdk-agglayer/aggkit-prover/l1-rpc` | `l1RpcUrl` | `aggkit-prover-secrets` | L1 RPC endpoint |
| `/<env>/cdk-agglayer/aggkit-prover/sp1-network-rpc` | `sp1NetworkRpcUrl` | `aggkit-prover-secrets` | SP1 Network RPC endpoint |

### Agglayer

| AWS Secret Path | Properties | Kubernetes Secret | Purpose |
|----------------|------------|-------------------|---------|
| `/<env>/cdk-agglayer/agglayer/l1-keystore` | `l1Keystore` (base64) | `agglayer-keystore-secrets` | L1 private key file |
| `/<env>/cdk-agglayer/agglayer/agglayer-keystore` | `agglayerKeystore` (base64) | `agglayer-keystore-secrets` | Agglayer private key file |
| `/<env>/cdk-agglayer/agglayer/keystore-passwords` | `l1KeystorePassword`, `agglayerKeystorePassword` | `agglayer-secrets` | Keystore passwords |
| `/<env>/cdk-agglayer/agglayer/l1-rpc` | `l1RpcUrl` | `agglayer-secrets` | L1 RPC endpoint |

### Agglayer-Prover

| AWS Secret Path | Properties | Kubernetes Secret | Purpose |
|----------------|------------|-------------------|---------|
| `/<env>/cdk-agglayer/agglayer-prover/network-private-key` | `networkPrivateKey` | `agglayer-prover-secrets` | Network private key (hex) |

### Bridge

| AWS Secret Path | Properties | Kubernetes Secret | Purpose |
|----------------|------------|-------------------|---------|
| `/<env>/cdk-agglayer/bridge/postgres` | `user`, `password`, `name`, `host`, `port` | `bridge-secrets` | Database credentials |
| `/<env>/cdk-agglayer/bridge/l1-rpc` | `l1RpcUrl` | `bridge-secrets` | L1 RPC endpoint |
| `/<env>/cdk-agglayer/bridge/l2-rpc` | `l2RpcUrl` | `bridge-secrets` | L2 RPC endpoint |
| `/<env>/cdk-agglayer/bridge/claim-tx-private-key` | `claimTxPrivateKeyPassword` | `bridge-secrets` | Claim transaction key password |

### Bridge-UI

| AWS Secret Path | Properties | Kubernetes Secret | Purpose |
|----------------|------------|-------------------|---------|
| `/<env>/cdk-agglayer/bridge-ui/l1-rpc` | `l1RpcUrl` | `bridge-ui-secrets` | L1 RPC endpoint |

**Note:** Replace `<env>` with your environment: `dev`, `uat`, `test`, or `main`

## Common Operations

### Secret Rotation Workflow

1. **Update secret in AWS:**
```bash
aws secretsmanager update-secret \
  --secret-id /dev/cdk-agglayer/aggkit/keystore-passwords \
  --secret-string '{"aggsenderKeystorePassword":"NEW_PASSWORD","trustedSequencerKeystorePassword":"NEW_PASSWORD2"}' \
  --region ap-south-1
```

2. **Force ExternalSecret sync:**
```bash
kubectl annotate externalsecret aggkit-external-secrets \
  -n cdk-agglayer \
  force-sync=$(date +%s) \
  --overwrite
```

3. **Verify update:**
```bash
kubectl get secret aggkit-secrets -n cdk-agglayer \
  -o jsonpath='{.data.aggsenderKeystorePassword}' | base64 -d
```

4. **Restart affected pods:**
```bash
kubectl rollout restart statefulset aggkit -n cdk-agglayer
kubectl rollout status statefulset aggkit -n cdk-agglayer
```

### Environment Migration (dev → uat → main)

1. **Export secrets from source env:**
```bash
# List all dev secrets
aws secretsmanager list-secrets \
  --filters Key=name,Values=/dev/cdk-agglayer \
  --region ap-south-1 \
  --query 'SecretList[].Name' \
  --output text

# Export specific secret
aws secretsmanager get-secret-value \
  --secret-id /dev/cdk-agglayer/aggkit/l1-rpc \
  --region ap-south-1 \
  --query SecretString \
  --output text > /tmp/aggkit-l1-rpc.json
```

2. **Create in target env:**
```bash
# Create in uat with updated values
aws secretsmanager create-secret \
  --name /uat/cdk-agglayer/aggkit/l1-rpc \
  --secret-string file:///tmp/aggkit-l1-rpc.json \
  --region ap-south-1
```

3. **Deploy with new env:**
```bash
helm upgrade aggkit ./charts/aggkit \
  -n cdk-agglayer \
  --set externalSecrets.enabled=true \
  --set externalSecrets.env=uat \
  -f ./charts/aggkit/values.yaml
```

### Health Check Script

```bash
#!/bin/bash
# save as: check-secrets-health.sh

NAMESPACE="cdk-agglayer"
COMPONENTS=("aggkit" "aggkit-prover" "agglayer" "agglayer-prover" "bridge" "bridge-ui")

echo "=== ClusterSecretStore Status ==="
kubectl get clustersecretstore aws-secretsmanager
echo ""

echo "=== ExternalSecrets Status ==="
for component in "${COMPONENTS[@]}"; do
  echo "--- ${component} ---"
  kubectl get externalsecret -n ${NAMESPACE} -l app.kubernetes.io/name=${component} 2>/dev/null || echo "No ExternalSecret found"
done
echo ""

echo "=== Kubernetes Secrets ==="
kubectl get secrets -n ${NAMESPACE} | grep -E "aggkit|agglayer|bridge"
echo ""

echo "=== Pod Status ==="
kubectl get pods -n ${NAMESPACE}
echo ""

echo "=== Recent Events ==="
kubectl get events -n ${NAMESPACE} --sort-by='.lastTimestamp' | tail -20
```

Run with:
```bash
chmod +x check-secrets-health.sh
./check-secrets-health.sh
```

## Security Best Practices

1. **Use different AWS secrets per environment** - Never share dev/uat/prod secrets
2. **Rotate secrets regularly** - Set up automated rotation for passwords and keys
3. **Limit IAM permissions** - Grant minimum required access (GetSecretValue, DescribeSecret)
4. **Use separate keystores** - Don't reuse the same keystore file across components
5. **Enable audit logging** - Use AWS CloudTrail for Secrets Manager access logs
6. **Encrypt etcd** - Ensure Kubernetes etcd encryption is enabled for secret data at rest
7. **Use RBAC** - Limit which ServiceAccounts can access which Secrets in Kubernetes
8. **Monitor secret access** - Set up alerts for unusual secret access patterns

## Reference Documentation

- **Installation Guide**: `charts/*/README.md` - Component-specific installation
- **Migration Summary**: `ESO_MIGRATION_SUMMARY.md` - Implementation details
- **Security Guide**: `SECURITY.md` - Security best practices
- **External Secrets Docs**: https://external-secrets.io/latest/
- **AWS Secrets Manager**: https://docs.aws.amazon.com/secretsmanager/

## Quick Reference Commands

```bash
# List all secrets in AWS
aws secretsmanager list-secrets --region ap-south-1 | grep cdk-agglayer

# Get all ExternalSecrets in namespace
kubectl get externalsecret -n cdk-agglayer -o wide

# Get all generated K8s secrets
kubectl get secrets -n cdk-agglayer | grep -E "secrets|keystore"

# Check pods health
kubectl get pods -n cdk-agglayer -o wide

# Stream logs from all aggkit pods
kubectl logs -n cdk-agglayer -l app.kubernetes.io/name=aggkit -f

# Exec into pod for debugging
kubectl exec -it -n cdk-agglayer aggkit-0 -- /bin/bash

# Force refresh all ExternalSecrets
for es in $(kubectl get externalsecret -n cdk-agglayer -o name); do
  kubectl annotate ${es} -n cdk-agglayer force-sync=$(date +%s) --overwrite
done
```
