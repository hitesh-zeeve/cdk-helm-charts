# Bridge UI

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 0.1.0](https://img.shields.io/badge/AppVersion-0.1.0-informational?style=flat-square)

Installs and configures the bridge UI service on Kubernetes using Helm. This repo is used to create & manage the helm chart.

## Installation

### Quick Start - Local Development

```bash
# Install with local configuration
helm install bridge-ui ./charts/bridge-ui -f charts/bridge-ui/values.local.yaml

# Access via port-forward (no ingress/NodePort needed)
kubectl port-forward svc/bridge-ui 8080:80

# Access at http://localhost:8080
```

### Ternoa Production Configuration

```bash
# Install with Ternoa-specific configuration
helm install bridge-ui ./charts/bridge-ui -f charts/bridge-ui/values.ternoa.yaml

# If using NodePort, access at http://<node-ip>:30088
```

### Custom Configuration

```bash
# Install with custom values
helm install bridge-ui ./charts/bridge-ui -f your-custom-values.yaml
```

## Configuration

### Environment Variables

The chart supports all environment variables from the bridge-ui Docker image:

#### Ethereum (L1) Configuration
- `ETHEREUM_RPC_URL` - Ethereum L1 RPC endpoint
- `ETHEREUM_EXPLORER_URL` - Ethereum block explorer URL
- `ETHEREUM_BRIDGE_CONTRACT_ADDRESS` - Bridge contract on L1
- `ETHEREUM_FORCE_UPDATE_GLOBAL_EXIT_ROOT` - Force update flag
- `ETHEREUM_PROOF_OF_EFFICIENCY_CONTRACT_ADDRESS` - PoE contract address
- `ETHEREUM_ROLLUP_MANAGER_ADDRESS` - Rollup manager contract address

#### Polygon zkEVM (L2) Configuration
- `POLYGON_ZK_EVM_RPC_URL` - zkEVM RPC endpoint
- `POLYGON_ZK_EVM_EXPLORER_URL` - zkEVM block explorer URL
- `POLYGON_ZK_EVM_BRIDGE_CONTRACT_ADDRESS` - Bridge contract on L2
- `POLYGON_ZK_EVM_NETWORK_ID` - Network ID
- `POLYGON_ZK_EVM_NATIVE_GAS_TOKEN` - Use native gas token (optional)
- `POLYGON_ZK_EVM_NATIVE_GAS_TOKEN_ADDRESS` - Native gas token address (optional)
- `POLYGON_ZK_EVM_WETH_TOKEN_ADDRESS` - WETH token address (optional)

#### Bridge API
- `BRIDGE_API_URL` - Bridge service API endpoint

#### Feature Flags
- `ENABLE_FIAT_EXCHANGE_RATES` - Enable fiat exchange rates
- `ENABLE_OUTDATED_NETWORK_MODAL` - Show outdated network modal
- `ENABLE_DEPOSIT_WARNING` - Show deposit warnings
- `ENABLE_REPORT_FORM` - Enable report form

#### Customization (Vite Build Variables)
- `VITE_RESOLVE_RELATIVE_URLS` - Resolve relative URLs
- `VITE_CUSTOMIZATION` - JSON customization object
- `VITE_NAME` - Brand name
- `VITE_APP_FAVICON` - Custom favicon URL
- `VITE_APP_TITLE` - Custom app title
- `VITE_TOKEN_NAME` - Token name
- `VITE_TOKEN_SYMBOL` - Token symbol
- `VITE_TOKEN_DECIMAL` - Token decimals

### Service Configuration

```yaml
service:
  type: ClusterIP  # Options: ClusterIP, NodePort, LoadBalancer
  nodePort: 30080  # Only when type is NodePort
```

### Ingress Configuration

```yaml
ingress:
  enabled: true
  hostname: "bridge.example.com"
  annotations:
    kubernetes.io/ingress.class: nginx
  tls:
    enabled: true  # Enable for production with certificates
```

## Docker Compose Compatibility

This chart is fully compatible with docker-compose configurations. Simply map your `.env` file variables to the `container.env` section in values.yaml:

**Docker Compose .env:**
```env
ETHEREUM_RPC_URL=https://eth-mainnet.example.com
POLYGON_ZK_EVM_RPC_URL=https://rpc.example.com
BRIDGE_API_URL=https://bridge-api.example.com
```

**Helm values.yaml:**
```yaml
container:
  env:
    ethereumRpcUrl: "https://eth-mainnet.example.com"
    polygonZkEvmRpcUrl: "https://rpc.example.com"
    bridgeApiUrl: "https://bridge-api.example.com"
```

## Access Options

### Option 1: Port Forward (Recommended for Local)
```bash
kubectl port-forward svc/bridge-ui 8080:80
# Access at http://localhost:8080
```

### Option 2: NodePort
```yaml
# values.yaml
service:
  type: NodePort
  nodePort: 30088
```
Access at `http://<node-ip>:30088`

### Option 3: Ingress (Requires Ingress Controller)
```yaml
# values.yaml
ingress:
  enabled: true
  hostname: "bridge.example.com"
  annotations:
    kubernetes.io/ingress.class: nginx
```

Install nginx ingress controller:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

