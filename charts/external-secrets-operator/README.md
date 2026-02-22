# External Secrets Operator Helm Chart

This chart installs the External Secrets Operator (ESO) with AWS Secrets Manager integration for EKS clusters using IRSA (IAM Roles for Service Accounts).

## Architecture

- Each environment (dev/stage/prod) has its own EKS cluster
- Terraform creates IAM policy and IRSA role (external to this chart)
- External Secrets Operator is installed directly from the official chart
- This Helm chart:
  - Creates ClusterSecretStore for AWS Secrets Manager
  - References the ServiceAccount created by the ESO installation

## Prerequisites

1. **EKS Cluster** with OIDC provider configured
2. **IAM Role** created by Terraform with:
   - Trust policy for the ServiceAccount
   - Read-only access to AWS Secrets Manager
3. **Helm 3.x** installed

## Installation

### Step 1: Install External Secrets Operator

First, install the External Secrets Operator from the official repository:

```bash
# Add the External Secrets Helm repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Install the External Secrets Operator (CRDs included)
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  -f eso-values-prod.yaml
```

Edit `eso-values-prod.yaml` and set `serviceAccount.annotations.eks.amazonaws.com/role-arn` to your IRSA role ARN.

### Step 2: Install ClusterSecretStore Configuration

Then install this chart to create the ClusterSecretStore:

```bash
cd external-secrets-operator
helm install external-secrets-operator . \
  -n external-secrets \
  -f values-prod.yaml
```

### Environment-Specific Installation

**For Development:**
```bash
# Step 1: Install ESO (dev)
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  -f eso-values-dev.yaml

# Step 2: Install ClusterSecretStore
helm install external-secrets-operator . \
  -n external-secrets \
  -f values-dev.yaml
```

**For Production:**
```bash
# Step 1: Install ESO (prod)
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  -f eso-values-prod.yaml

# Step 2: Install ClusterSecretStore
helm install external-secrets-operator . \
  -n external-secrets \
  -f values-prod.yaml
```

## Why Two Separate Installations?

Helm 3 has a limitation where CRDs from subchart dependencies are not installed automatically. By installing the External Secrets Operator directly from the official repository, we ensure:
- CRDs are properly installed with Helm ownership metadata
- The operator runs with correct IRSA configuration
- Future upgrades are handled cleanly

## Configuration

### This Chart's Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `aws.region` | AWS region for Secrets Manager | `"us-east-1"` |
| `external-secrets.serviceAccount.name` | ServiceAccount name (must match ESO installation) | `"external-secrets-sa"` |

### External Secrets Operator Values

When installing the ESO chart directly, you can configure:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `installCRDs` | Install External Secrets CRDs | `true` |
| `serviceAccount.create` | Create ServiceAccount | `true` |
| `serviceAccount.name` | ServiceAccount name | `"external-secrets-sa"` |
| `serviceAccount.annotations` | ServiceAccount annotations (for IRSA) | `{}` |

For full configuration options, see: https://github.com/external-secrets/external-secrets/tree/main/deploy/charts/external-secrets

## AWS Secrets Naming Convention

Secrets in AWS Secrets Manager should follow this pattern:

```
/<environment>/<namespace>/<component>/<secret>
```

Examples:
- `/prod/payments/api/db`
- `/dev/auth/service/jwt`
- `/stage/op-succinct/batcher/private-key`

## Verification

### 1. Check ESO Installation

```bash
kubectl get pods -n external-secrets
```

### 2. Verify ServiceAccount

```bash
kubectl get sa external-secrets-sa -n external-secrets -o yaml
```

Verify the annotation:
```yaml
annotations:
  eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eso-irsa-prod
```

### 3. Check ClusterSecretStore

```bash
kubectl get clustersecretstore aws-secretsmanager
kubectl describe clustersecretstore aws-secretsmanager
```

The status should show `Valid: True`.

## Troubleshooting

### ClusterSecretStore Not Ready

Check ESO logs:
```bash
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets
```

### IRSA Issues

Verify the ServiceAccount annotation:
```bash
kubectl get sa external-secrets-sa -n external-secrets -o jsonpath='{.metadata.annotations}'
```

Verify IAM role trust policy includes the ServiceAccount.

### Secrets Manager Access

Test from a pod using the ServiceAccount:
```bash
kubectl run aws-cli --rm -it --image=amazon/aws-cli --serviceaccount=external-secrets-sa -n external-secrets -- secretsmanager list-secrets --region ap-south-1
```

## Uninstall

```bash
# Uninstall ClusterSecretStore configuration
helm uninstall external-secrets-operator -n external-secrets

# Uninstall External Secrets Operator
helm uninstall external-secrets -n external-secrets

# Delete namespace
kubectl delete namespace external-secrets
```

**Note:** Uninstalling the ESO chart will also remove the CRDs, which will delete all ExternalSecret resources across all namespaces.

## Next Steps

After installing ESO, configure ExternalSecret resources in your application charts. See the main README for details.
