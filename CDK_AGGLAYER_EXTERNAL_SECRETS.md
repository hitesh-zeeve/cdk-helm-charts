# External Secrets (AWS Secrets Manager) Setup — CDK Agglayer Helm Charts

This guide provides the **exact AWS Secrets Manager secrets** (paths + JSON fields) required by these charts:

- `aggkit`
- `aggkit-prover`
- `agglayer`
- `agglayer-prover`
- `bridge` (bridge-service)
- `bridge-ui`

It is written in the same spirit as `op-succinct/EXTERNALSECRETS.md`: create AWS secrets following a convention, then ESO (`ExternalSecret`) extracts those JSON fields and generates Kubernetes Secrets.

## Overview

Each chart has one or more `ExternalSecret` resources (see `charts/*/templates/externalsecret.yaml`). Most of them use:

- `spec.dataFrom.extract.key: <aws-secret-path>` (extract JSON)
- `spec.target.template.data: ...` (select/rename fields)
- For binary files (keystores/certs), fields are stored in AWS as **base64**, then decoded in ESO using `| b64dec`.

## Naming Convention

All AWS secrets must follow this pattern:

```
/<environment>/cdk-agglayer/<component>/<secret>
```

Examples:
- `/dev/cdk-agglayer/aggkit/credentials`
- `/uat/cdk-agglayer/bridge/postgres`

## Prerequisites

- External Secrets Operator installed
- A `ClusterSecretStore` (or `SecretStore`) named `aws-secretsmanager` configured to read from AWS Secrets Manager
- You know which AWS region your `ClusterSecretStore` uses (create secrets in that region)

## Set Variables (recommended)

```bash
export ENV="dev"              # dev | uat | test | main
export AWS_REGION="ap-south-1" # must match ClusterSecretStore region
export PREFIX="/${ENV}/cdk-agglayer"
```

---

## Shared Secret (used by multiple charts)

### 1) Shared L1 RPC URL

**Path:** `${PREFIX}/shared/l1-rpc-url`

**Required JSON fields:**
- `l1RpcURL`

```bash
cat > shared-l1-rpc.json <<'EOF'
{
  "l1RpcURL": "https://YOUR_L1_RPC_URL"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/shared/l1-rpc-url" \
  --description "CDK Agglayer shared L1 RPC URL" \
  --secret-string file://shared-l1-rpc.json \
  --region "${AWS_REGION}"
```

Used by:
- `aggkit`
- `aggkit-prover`
- `bridge`
- `bridge-ui`

---

## aggkit

### 2) aggkit credentials (keystore passwords)

**Path:** `${PREFIX}/aggkit/credentials`

**Required JSON fields:**
- `aggsenderKeystorePassword`
- `aggsenderValidatorKeystorePassword`
- `aggoracleKeystorePassword`

```bash
cat > aggkit-credentials.json <<'EOF'
{
  "aggsenderKeystorePassword": "CHANGE_ME",
  "aggsenderValidatorKeystorePassword": "CHANGE_ME",
  "aggoracleKeystorePassword": "CHANGE_ME"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/aggkit/credentials" \
  --description "aggkit keystore passwords" \
  --secret-string file://aggkit-credentials.json \
  --region "${AWS_REGION}"
```

### 3) aggkit keystores (binary files; base64 in AWS)

**Path:** `${PREFIX}/aggkit/keystores`

**Required JSON fields (base64):**
- `aggsenderKeystore`
- `aggsenderValidatorKeystore`
- `aggoracleKeystore`

**Encode keystore files:**

```bash
# base64 with no wrapping
AGGSENDER_B64=$(base64 -w 0 /path/to/aggsender.keystore)
VALIDATOR_B64=$(base64 -w 0 /path/to/aggsenderValidator.keystore)
ORACLE_B64=$(base64 -w 0 /path/to/aggoracle.keystore)

cat > aggkit-keystores.json <<EOF
{
  "aggsenderKeystore": "${AGGSENDER_B64}",
  "aggsenderValidatorKeystore": "${VALIDATOR_B64}",
  "aggoracleKeystore": "${ORACLE_B64}"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/aggkit/keystores" \
  --description "aggkit keystore files (base64)" \
  --secret-string file://aggkit-keystores.json \
  --region "${AWS_REGION}"
```

ESO will decode these into Kubernetes Secret keys:
- `aggsender.keystore`
- `aggsenderValidator.keystore`
- `aggoracle.keystore`

---

## aggkit-prover

### 4) aggkit-prover credentials

**Path:** `${PREFIX}/aggkit-prover/credentials`

**Required JSON fields:**
- `sp1NetworkPrivateKey`
- `sp1NetworkRpcURL`

```bash
cat > aggkit-prover-credentials.json <<'EOF'
{
  "sp1NetworkPrivateKey": "0xYOUR_PRIVATE_KEY",
  "sp1NetworkRpcURL": "https://rpc.succinct.xyz/"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/aggkit-prover/credentials" \
  --description "aggkit-prover credentials" \
  --secret-string file://aggkit-prover-credentials.json \
  --region "${AWS_REGION}"
```

---

## agglayer

### 5) agglayer credentials (keystore password)

**Path:** `${PREFIX}/agglayer/credentials`

**Required JSON fields:**
- `aggregatorKeystorePassword`

```bash
cat > agglayer-credentials.json <<'EOF'
{
  "aggregatorKeystorePassword": "CHANGE_ME"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/agglayer/credentials" \
  --description "agglayer keystore password" \
  --secret-string file://agglayer-credentials.json \
  --region "${AWS_REGION}"
```

### 6) agglayer keystores (binary files; base64 in AWS)

**Path:** `${PREFIX}/agglayer/keystores`

**Required JSON fields (base64):**
- `aggregatorKeystore`

```bash
AGGREGATOR_B64=$(base64 -w 0 /path/to/aggregator.keystore)

cat > agglayer-keystores.json <<EOF
{
  "aggregatorKeystore": "${AGGREGATOR_B64}"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/agglayer/keystores" \
  --description "agglayer keystore file (base64)" \
  --secret-string file://agglayer-keystores.json \
  --region "${AWS_REGION}"
```

ESO will decode this into a Kubernetes Secret key:
- `aggregator.keystore`

### 7) (Optional) agglayer TLS secret for Ingress

Created only when:
- `ingress.enabled=true` AND
- `tls.existingSecretName` is empty

**Path:** `${PREFIX}/agglayer/tls`

**Required JSON fields (base64):**
- `tlsCrt`
- `tlsKey`

```bash
TLS_CRT_B64=$(base64 -w 0 /path/to/tls.crt)
TLS_KEY_B64=$(base64 -w 0 /path/to/tls.key)

cat > agglayer-tls.json <<EOF
{
  "tlsCrt": "${TLS_CRT_B64}",
  "tlsKey": "${TLS_KEY_B64}"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/agglayer/tls" \
  --description "agglayer ingress TLS cert (base64)" \
  --secret-string file://agglayer-tls.json \
  --region "${AWS_REGION}"
```

### 8) (Optional) agglayer OAuth secret (GKE BackendConfig)

Created only when:
- `gke.enabled=true` AND
- `gke.backendConfig.enabled=true`

**Path:** `${PREFIX}/agglayer/oauth`

**Required JSON fields:**
- `clientId`
- `clientSecret`

```bash
cat > agglayer-oauth.json <<'EOF'
{
  "clientId": "YOUR_OAUTH_CLIENT_ID",
  "clientSecret": "YOUR_OAUTH_CLIENT_SECRET"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/agglayer/oauth" \
  --description "agglayer admin oauth credentials" \
  --secret-string file://agglayer-oauth.json \
  --region "${AWS_REGION}"
```

### 9) (Optional) agglayer Datadog API key

Created only when:
- `datadog.enabled=true`

**Path:** `${PREFIX}/agglayer/datadog`

**Required JSON fields:**
- `apiKey`

```bash
cat > agglayer-datadog.json <<'EOF'
{
  "apiKey": "YOUR_DATADOG_API_KEY"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/agglayer/datadog" \
  --description "agglayer datadog api key" \
  --secret-string file://agglayer-datadog.json \
  --region "${AWS_REGION}"
```

### 10) (Recommended) agglayer config secret (DATA_NODE_* etc)

This ExternalSecret is always created when `externalSecrets.enabled=true`.

**Path:** `${PREFIX}/agglayer/config`

**Expected JSON fields:**

The template reads these keys (missing keys default to empty string):
- `DATA_NODE_RPC_HOST`
- `DATA_NODE_RPC_PORT`
- `DATA_NODE_L1_CHAINID`
- `DATA_NODE_L1_NODEURL`
- `DATA_NODE_L1_NODEURL_WS`
- `DATA_NODE_L1_ROLLUPMANAGERCONTRACT`
- `DATA_NODE_L1_GLOBAL_EXIT_ROOT_CONTRACT`
- `DATA_NODE_KMSKEYNAME`
- `DATA_NODE_PROJECTID`
- `DATA_NODE_LOCATION`
- `DATA_NODE_KEYRING`
- `DATA_NODE_KEYNAME`
- `DATA_NODE_KEYVERSION`
- `DATA_NODE_PROOFSIGNERS`

Minimal example (fill what you use):

```bash
cat > agglayer-config.json <<'EOF'
{
  "DATA_NODE_RPC_HOST": "",
  "DATA_NODE_RPC_PORT": "",
  "DATA_NODE_L1_CHAINID": "",
  "DATA_NODE_L1_NODEURL": "",
  "DATA_NODE_L1_NODEURL_WS": "",
  "DATA_NODE_L1_ROLLUPMANAGERCONTRACT": "",
  "DATA_NODE_L1_GLOBAL_EXIT_ROOT_CONTRACT": "",
  "DATA_NODE_KMSKEYNAME": "",
  "DATA_NODE_PROJECTID": "",
  "DATA_NODE_LOCATION": "",
  "DATA_NODE_KEYRING": "",
  "DATA_NODE_KEYNAME": "",
  "DATA_NODE_KEYVERSION": "",
  "DATA_NODE_PROOFSIGNERS": ""
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/agglayer/config" \
  --description "agglayer config secret (DATA_NODE_*)" \
  --secret-string file://agglayer-config.json \
  --region "${AWS_REGION}"
```

---

## agglayer-prover

### 11) agglayer-prover credentials (network private key)

**Path:** `${PREFIX}/agglayer-prover/credentials`

**Required JSON fields:**
- `networkPrivateKey`

```bash
cat > agglayer-prover-credentials.json <<'EOF'
{
  "networkPrivateKey": "0xYOUR_PRIVATE_KEY"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/agglayer-prover/credentials" \
  --description "agglayer-prover network private key" \
  --secret-string file://agglayer-prover-credentials.json \
  --region "${AWS_REGION}"
```

Note: the chart currently has an ExternalSecret template for this, but its `values.yaml` doesn’t define `externalSecrets.*` yet. If you plan to enable ESO for `agglayer-prover`, align its values with the other charts (same `externalSecrets.enabled/refreshInterval/storeRef/awsSecrets.credentials`).

---

## bridge (bridge-service)

### 12) bridge postgres connection info

**Path:** `${PREFIX}/bridge/postgres`

**Required JSON fields:**
- `bridgeDbName`
- `dbUser`
- `dbUserPassword`
- `dbHost`
- `dbPort`

```bash
cat > bridge-postgres.json <<'EOF'
{
  "bridgeDbName": "bridge_db",
  "dbUser": "bridge_user",
  "dbUserPassword": "CHANGE_ME",
  "dbHost": "postgres.cdk-agglayer.svc.cluster.local",
  "dbPort": "5432"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/bridge/postgres" \
  --description "bridge postgres connection" \
  --secret-string file://bridge-postgres.json \
  --region "${AWS_REGION}"
```

### 13) bridge credentials (claim tx keystore password)

**Path:** `${PREFIX}/bridge/credentials`

**Required JSON fields:**
- `claimTxPrivateKeyPassword`

```bash
cat > bridge-credentials.json <<'EOF'
{
  "claimTxPrivateKeyPassword": "CHANGE_ME"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/bridge/credentials" \
  --description "bridge claim tx keystore password" \
  --secret-string file://bridge-credentials.json \
  --region "${AWS_REGION}"
```

### 14) bridge keystores (binary file; base64 in AWS)

**Path:** `${PREFIX}/bridge/keystores`

**Required JSON fields (base64):**
- `claimtxKeystore`

```bash
CLAIMTX_B64=$(base64 -w 0 /path/to/claimtx.keystore)

cat > bridge-keystores.json <<EOF
{
  "claimtxKeystore": "${CLAIMTX_B64}"
}
EOF

aws secretsmanager create-secret \
  --name "${PREFIX}/bridge/keystores" \
  --description "bridge claimtx keystore file (base64)" \
  --secret-string file://bridge-keystores.json \
  --region "${AWS_REGION}"
```

ESO will decode this into a Kubernetes Secret key:
- `claimtx.keystore`

---

## bridge-ui

### 15) bridge-ui secrets

`bridge-ui` only consumes the **shared L1 RPC** secret (`${PREFIX}/shared/l1-rpc-url`) via its own `ExternalSecret`.

No additional component-specific AWS secrets are required by `charts/bridge-ui/templates/externalsecret.yaml`.

---

## Idempotent Updates (when secrets already exist)

If `create-secret` fails because the secret exists, use `update-secret`:

```bash
aws secretsmanager update-secret \
  --secret-id "${PREFIX}/aggkit/credentials" \
  --secret-string file://aggkit-credentials.json \
  --region "${AWS_REGION}"
```

One-command update (no file) example (useful for quick fixes like `dbHost` changes):

```bash
aws secretsmanager update-secret \
  --secret-id "/dev/cdk-agglayer/bridge/postgres" \
  --secret-string '{"bridgeDbName":"postgres","dbUser":"bridge_user","dbUserPassword":"bridge_password","dbHost":"my-postgres","dbPort":"5432"}' \
  --region "us-east-2"
```

## Verification (quick)

```bash
# list secrets under your prefix
aws secretsmanager list-secrets \
  --region "${AWS_REGION}" \
  --query 'SecretList[].Name' \
  --output text | tr '\t' '\n' | grep "${PREFIX}/" | sort

# get one secret value (careful: prints secret)
aws secretsmanager get-secret-value \
  --secret-id "${PREFIX}/shared/l1-rpc-url" \
  --region "${AWS_REGION}"
```

## Reference

- Chart templates: `charts/*/templates/externalsecret.yaml`
- External Secrets Operator docs: https://external-secrets.io/
- AWS Secrets Manager docs: https://docs.aws.amazon.com/secretsmanager/
