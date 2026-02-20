# External Secrets Operator (ESO) Migration Inventory

## Executive Summary

Analysis completed for 6 Helm charts. All charts currently reference a single Kubernetes Secret named `op-secrets` (with some variations). Migration will replace this pattern with AWS Secrets Manager via External Secrets Operator.

**Namespace**: `cdk-agglayer`  
**Environment Variables**: dev, uat, test, main  
**AWS Secrets Manager Naming**: `/<env>/cdk-agglayer/<component>/<secret>`  
**SecretStore**: ClusterSecretStore named `aws-secretsmanager` (default)

---

## Detailed Inventory by Chart

### 1. aggkit

**Current Secrets Usage:**
- **Secret Reference**: `op-secrets` (via `values.existingSecret`)
- **Mount Method**: Projected volume combining ConfigMap + Secret
- **Secret Keys Used**:
  - `l1RpcURL` - L1 blockchain RPC endpoint URL
  - `aggsenderKeystorePassword` - Password for aggsender.keystore file
  - `aggsenderValidatorKeystorePassword` - Password for aggsenderValidator.keystore file
  - `aggoracleKeystorePassword` - Password for aggoracle.keystore file
  - Keystore binary files: `aggsender.keystore`, `aggsenderValidator.keystore`, `aggoracle.keystore`
- **Workload References**: 
  - StatefulSet mounts secret as part of projected volume at `/etc/aggkit`
  - ConfigMap template reads passwords from secret using `lookup` function
- **Additional**: Supports OnePassword integration via `onePasswordSecrets` values (will be removed)

**Secret Material Breakdown**:
- **RPC URLs**: 1 (L1)
- **Passwords**: 3 (keystore passwords)
- **Binary Files**: 3 (keystore files)

---

### 2. aggkit-prover

**Current Secrets Usage:**
- **Secret Reference**: `op-secrets` (via `values.existingSecret`)
- **Mount Method**: 
  - Environment variable injection (NETWORK_PRIVATE_KEY)
  - ConfigMap reads secrets using `lookup` function
- **Secret Keys Used**:
  - `sp1NetworkPrivateKey` - SP1 network private key (injected as env `NETWORK_PRIVATE_KEY`)
  - `l1RpcURL` - L1 RPC endpoint URL (in configmap)
  - `sp1NetworkRpcURL` - SP1 network RPC URL (in configmap)
- **Workload References**:
  - StatefulSet uses `lookup` to inject `sp1NetworkPrivateKey` as environment variable
  - ConfigMap template reads RPC URLs from secret

**Secret Material Breakdown**:
- **RPC URLs**: 2 (L1, SP1 Network)
- **Private Keys**: 1 (SP1 network key)

---

### 3. agglayer

**Current Secrets Usage:**
- **Secret Reference**: `op-secrets` (via `values.existingSecret`)
- **Explicit Secrets Created**:
  1. **certificate.yaml** (kind: Secret) - TLS certificate
     - Type: `kubernetes.io/tls`
     - Keys: `tls.crt`, `tls.key`
     - Conditional: Only created if `ingress.enabled=true`, `tls.createSecret=true`, and no `tls.existingSecretName`
     - Values source: `values.tls.base64Cert`, `values.tls.base64Key`
  
  2. **secret.yaml** (kind: Secret) - OAuth credentials
     - Type: Opaque
     - Keys: `client_id`, `client_secret`
     - Conditional: Only created if `gke.enabled=true` and `gke.backendConfig.enabled=true`
     - Values fetched from `op-secrets` using `lookup`: `admin_api_oauth_client_id`, `admin_api_oauth_client_secret`

- **Mount Method**: Projected volume combining ConfigMap + Secret
- **Secret Keys Used from op-secrets**:
  - `aggregatorKeystorePassword` - Password for aggregator keystore
  - Keystore file: `aggregator.keystore`
  - `admin_api_oauth_client_id` - OAuth client ID (for GKE BackendConfig)
  - `admin_api_oauth_client_secret` - OAuth client secret (for GKE BackendConfig)
  - `DATA_NODE_RPC_HOST` - Data node RPC host
  - `DATA_NODE_RPC_PORT` - Data node RPC port
  - `DATA_NODE_L1_CHAINID` - L1 chain ID
  - `DATA_NODE_L1_NODEURL` - L1 node URL
  - `DATA_NODE_L1_NODEURL_WS` - L1 node WebSocket URL
  - `DATA_NODE_L1_ROLLUPMANAGERCONTRACT` - L1 rollup manager contract address
  - `DATA_NODE_L1_GLOBAL_EXIT_ROOT_CONTRACT` - L1 global exit root contract
  - `datadog_api_key` - Datadog API key
  - `DATA_NODE_KMSKEYNAME` - GCP KMS key name
  - `DATA_NODE_PROJECTID` - GCP project ID
  - `DATA_NODE_LOCATION` - GCP location
  - `DATA_NODE_KEYRING` - GCP keyring
  - `DATA_NODE_KEYNAME` - GCP key name
  - `DATA_NODE_KEYVERSION` - GCP key version
  - `DATA_NODE_PROOFSIGNERS` - Proof signers configuration

- **Workload References**:
  - StatefulSet mounts secret if `aggregator.keyName` is empty (i.e., not using GCP KMS)
  - Config template uses `lookup` to read secret values

**Secret Material Breakdown**:
- **TLS Certificates**: 1 set (cert + key)
- **OAuth Credentials**: 1 set (client_id + client_secret)
- **Passwords**: 1 (keystore password)
- **Binary Files**: 1 (keystore file)
- **RPC/API URLs**: 3 (host, URL, WS URL)
- **Contract Addresses**: 2
- **API Keys**: 1 (Datadog)
- **GCP KMS Config**: 6 fields
- **Misc Config**: Multiple DATA_NODE_* fields

---

### 4. agglayer-prover

**Current Secrets Usage:**
- **Secret Reference**: None active (code commented out)
- **Commented Secret Usage**:
  - `NETWORK_PRIVATE_KEY` from `op-secrets` (currently commented)
- **Mount Method**: N/A
- **Secret Keys Used**: None (no active secret consumption)
- **Workload References**: None

**Secret Material Breakdown**:
- **None active** (deployment has placeholder for future private key)

---

### 5. bridge

**Current Secrets Usage:**
- **Secret Reference**: `op-secrets` (via `values.existingSecret`)
- **Mount Method**: Projected volume combining ConfigMap + Secret file
- **Secret Keys Used**:
  - `claimTxPrivateKeyPassword` - Password for claim transaction keystore
  - `l1RpcURL` - L1 RPC URL
  - `bridgeDbName` - PostgreSQL database name
  - `dbUser` - PostgreSQL username
  - `dbUserPassword` - PostgreSQL user password
  - `dbHost` - PostgreSQL host
  - `dbPort` - PostgreSQL port
  - Keystore file: `claimtx.keystore` (referenced via `values.secretItem.key`)
- **Workload References**:
  - Deployment mounts ConfigMap + Secret as projected volume at `/etc/bridge/`
  - Secret file `claimtx.keystore` is mounted alongside config
  - ConfigMap template reads passwords and DB credentials from secret using `lookup`

**Secret Material Breakdown**:
- **Database Credentials**: 5 fields (name, user, password, host, port)
- **RPC URLs**: 1 (L1)
- **Passwords**: 1 (keystore password)
- **Binary Files**: 1 (keystore file)

---

### 6. bridge-ui

**Current Secrets Usage:**
- **Secret Reference**: `op-secrets` (hardcoded in deployment template)
- **Mount Method**: Environment variable injection only
- **Secret Keys Used**:
  - `l1RpcURL` - L1 RPC URL (fallback if `values.container.env.ethereumRpcUrl` not set)
- **Workload References**:
  - Deployment uses `lookup` to read `l1RpcURL` and injects as `ETHEREUM_RPC_URL` env var
  - No volume mounts of secrets
- **Notes**: All other configuration via values.yaml (no other secret material)

**Secret Material Breakdown**:
- **RPC URLs**: 1 (L1)

---

## Summary Statistics

| Chart | Secret Refs | Keystore Files | Passwords | RPC URLs | DB Creds | TLS Certs | API Keys | Other |
|-------|-------------|----------------|-----------|----------|----------|-----------|----------|-------|
| aggkit | 1 | 3 | 3 | 1 | 0 | 0 | 0 | 0 |
| aggkit-prover | 1 | 0 | 0 | 2 | 0 | 0 | 0 | 1 private key |
| agglayer | 1 (+2 explicit) | 1 | 1 | 3 | 0 | 1 | 1 | 20+ config fields |
| agglayer-prover | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 (inactive) |
| bridge | 1 | 1 | 1 | 1 | 5 | 0 | 0 | 0 |
| bridge-ui | 1 | 0 | 0 | 1 | 0 | 0 | 0 | 0 |

---

## Current Problems & Anti-Patterns

1. **Plaintext in Values**: Some charts may have plaintext secrets in values.yaml (especially TLS certs in agglayer)
2. **Lookup Function Usage**: Heavy reliance on `lookup` function which:
   - Doesn't work during initial install (secret doesn't exist yet)
   - Creates tight coupling with existing secret
   - Makes templates non-declarative
3. **Single Monolithic Secret**: All components share `op-secrets`, violating least-privilege
4. **No Separation by Environment**: Same secret name across all environments
5. **Hardcoded Secret Names**: `op-secrets` hardcoded in templates and values
6. **Binary Files in Secrets**: Keystore files stored directly in secrets (acceptable but need careful migration)

---

## AWS Secrets Manager Structure Proposal

### Naming Convention
```
/<env>/cdk-agglayer/<component>/<secret-category>
```

### Secret Organization Strategy

**Option A: One JSON secret per component** (RECOMMENDED)
```
/dev/cdk-agglayer/aggkit/credentials
{
  "l1RpcURL": "https://...",
  "aggsenderKeystorePassword": "...",
  "aggsenderValidatorKeystorePassword": "...",
  "aggoracleKeystorePassword": "...",
  "aggsender.keystore": "<base64>",
  "aggsenderValidator.keystore": "<base64>",
  "aggoracle.keystore": "<base64>"
}
```

**Option B: Separate secrets by type**
```
/dev/cdk-agglayer/aggkit/rpc-urls
/dev/cdk-agglayer/aggkit/keystore-passwords
/dev/cdk-agglayer/aggkit/keystores
```

**RECOMMENDATION**: Use Option A for simplicity and atomic updates, except for:
- Shared secrets (L1 RPC URL) → `/dev/cdk-agglayer/shared/l1-rpc-url`
- Database credentials → `/dev/cdk-agglayer/bridge/postgres`
- Large binary files (keystores) → May need separate storage or base64 encoding

### Proposed Secret Mapping

#### Shared Secrets (used by multiple components)
```
/dev/cdk-agglayer/shared/l1-rpc-url
{
  "url": "https://eth-sepolia.g.alchemy.com/v2/..."
}
```

#### aggkit
```
/dev/cdk-agglayer/aggkit/credentials
{
  "aggsenderKeystorePassword": "...",
  "aggsenderValidatorKeystorePassword": "...",
  "aggoracleKeystorePassword": "..."
}

/dev/cdk-agglayer/aggkit/keystores
{
  "aggsender.keystore": "<base64>",
  "aggsenderValidator.keystore": "<base64>",
  "aggoracle.keystore": "<base64>"
}
```

#### aggkit-prover
```
/dev/cdk-agglayer/aggkit-prover/credentials
{
  "sp1NetworkPrivateKey": "0x...",
  "sp1NetworkRpcURL": "https://..."
}
```

#### agglayer
```
/dev/cdk-agglayer/agglayer/credentials
{
  "aggregatorKeystorePassword": "..."
}

/dev/cdk-agglayer/agglayer/keystores
{
  "aggregator.keystore": "<base64>"
}

/dev/cdk-agglayer/agglayer/tls
{
  "tls.crt": "<base64>",
  "tls.key": "<base64>"
}

/dev/cdk-agglayer/agglayer/oauth
{
  "client_id": "...",
  "client_secret": "..."
}

/dev/cdk-agglayer/agglayer/datadog
{
  "api_key": "..."
}

/dev/cdk-agglayer/agglayer/config
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
```
# No active secrets currently
# Future:
/dev/cdk-agglayer/agglayer-prover/credentials
{
  "networkPrivateKey": "0x..."
}
```

#### bridge
```
/dev/cdk-agglayer/bridge/credentials
{
  "claimTxPrivateKeyPassword": "..."
}

/dev/cdk-agglayer/bridge/keystores
{
  "claimtx.keystore": "<base64>"
}

/dev/cdk-agglayer/bridge/postgres
{
  "bridgeDbName": "bridge",
  "dbUser": "bridge_user",
  "dbUserPassword": "...",
  "dbHost": "postgres.cdk-agglayer.svc.cluster.local",
  "dbPort": "5432"
}
```

#### bridge-ui
```
# Uses shared L1 RPC URL only
```

---

## Migration Notes

### Binary Files (Keystores)
Keystore files need special handling:
- **Store as base64** in AWS Secrets Manager JSON
- **Decode in ExternalSecret** using `decodeStrategy: Base64`
- **Alternative**: Store keystores in S3 and reference via IAM (more complex)

### Lookup Function Replacement
All `lookup` calls in templates must be removed:
- ConfigMaps should reference values or environment variables
- Passwords/secrets should be mounted as files or env vars from ESO-generated secrets
- No runtime secret lookups

### Testing Strategy
1. **Parallel Run**: Create ExternalSecrets alongside existing `op-secrets`
2. **Validation**: Verify ESO-generated secrets match `op-secrets` structure
3. **Cutover**: Update workload references from `op-secrets` to new secret names
4. **Cleanup**: Remove `op-secrets` and OnePassword integrations

---

## Next Steps

1. ✅ Inventory Complete
2. 🔄 Design ExternalSecret templates per chart
3. ⏳ Update values.yaml with ESO configuration
4. ⏳ Remove/replace Secret manifests
5. ⏳ Update ConfigMap templates to remove `lookup` calls
6. ⏳ Update workload templates to reference new secrets
7. ⏳ Create migration runbook
8. ⏳ Document secret name changes
