# External Secrets Operator Helm Chart

This chart installs the External Secrets Operator (ESO) with AWS Secrets Manager integration for EKS clusters using IRSA (IAM Roles for Service Accounts).

## Architecture

- Each environment (dev/stage/prod) has its own EKS cluster
- Terraform creates IAM policy and IRSA role (external to this chart)
- This Helm chart:
  - Installs External Secrets Operator
  - Creates `external-secrets` namespace
  - Creates ServiceAccount with IRSA annotation
  - Deploys ClusterSecretStore for AWS Secrets Manager

## Prerequisites

1. **EKS Cluster** with OIDC provider configured
2. **IAM Role** created by Terraform with:
   - Trust policy for the ServiceAccount
   - Read-only access to AWS Secrets Manager
3. **Helm 3.x** installed

## Installation

### 1. Update Dependencies

```bash
helm dependency update
```

### 2. Install the Chart

```bash
helm install external-secrets-operator . \
  --namespace external-secrets \
  --create-namespace \
  --set aws.irsaRoleArn="arn:aws:iam::123456789012:role/eso-irsa-prod" \
  --set aws.region="ap-south-1"
```

### 3. Environment-Specific Values

Create environment-specific values files:

**values-dev.yaml**
```yaml
aws:
  irsaRoleArn: "arn:aws:iam::123456789012:role/eso-irsa-dev"
  region: "ap-south-1"
```

**values-prod.yaml**
```yaml
aws:
  irsaRoleArn: "arn:aws:iam::987654321098:role/eso-irsa-prod"
  region: "ap-south-1"
```

Install with environment-specific values:

```bash
helm install external-secrets-operator . \
  --namespace external-secrets \
  --create-namespace \
  -f values-prod.yaml
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `aws.irsaRoleArn` | IAM role ARN for IRSA | `""` |
| `aws.region` | AWS region for Secrets Manager | `"ap-south-1"` |
| `external-secrets.installCRDs` | Install External Secrets CRDs | `true` |
| `external-secrets.serviceAccount.name` | ServiceAccount name | `"external-secrets-sa"` |

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
helm uninstall external-secrets-operator --namespace external-secrets
kubectl delete namespace external-secrets
```

## Next Steps

After installing ESO, configure ExternalSecret resources in your application charts. See the main README for details.
