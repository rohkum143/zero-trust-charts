# Helm Charts Repository

This repository contains Helm charts for Kubernetes security policies including Istio Authorization Policies and Kubernetes Network Policies.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Charts](#charts)
  - [Authorization Policy Chart](#authorization-policy-chart)
  - [Network Policy Chart](#network-policy-chart)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)

## Overview

This repository provides two main Helm charts:

1. **authorization-policy**: Implements Istio Authorization Policies with a default deny-all configuration for the istio-system namespace
2. **network-policy**: Implements Kubernetes Network Policies that can be deployed to any namespace with customizable ingress/egress rules

## Prerequisites

- Kubernetes cluster (v1.19+)
- Helm 3.x installed
- For authorization-policy chart: Istio service mesh installed and configured
- kubectl configured to access your cluster

## Charts

### Authorization Policy Chart

**Location**: `charts/authorization-policy`

**Description**: This chart deploys an Istio Authorization Policy that implements a default deny-all policy in the istio-system namespace. This ensures that all traffic is denied by default unless explicitly allowed.

**Key Features**:
- Default deny-all policy for enhanced security
- Configurable rules to allow specific traffic
- Support for workload selectors
- Customizable labels and annotations

### Network Policy Chart

**Location**: `charts/network-policy`

**Description**: This chart deploys Kubernetes Network Policies to control traffic flow at the IP address or port level (OSI layer 3 or 4).

**Key Features**:
- Configurable ingress and egress rules
- Default DNS egress rules included
- Support for namespace and pod selectors
- Flexible policy types configuration

## Installation

### Adding the Helm Repository

```bash
# Clone the repository
git clone <your-repo-url>
cd charts
```

### Installing Authorization Policy Chart

```bash
# Install with default values (default deny-all in istio-system)
helm install authorization-policy ./authorization-policy

# Install with custom values
helm install authorization-policy ./authorization-policy -f custom-values.yaml

# Install in a specific namespace (note: the policy itself applies to istio-system)
helm install authorization-policy ./authorization-policy -n security-policies --create-namespace
```

### Installing Network Policy Chart

```bash
# Install with default values in default namespace
helm install network-policy ./network-policy

# Install in a specific namespace
helm install network-policy ./network-policy -n my-app --create-namespace

# Install with custom values
helm install network-policy ./network-policy -f custom-values.yaml -n my-app
```

## Configuration

### Authorization Policy Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `authorizationPolicy.name` | Name of the authorization policy | `default-deny-all` |
| `authorizationPolicy.namespace` | Namespace where the policy will be applied | `istio-system` |
| `authorizationPolicy.enabled` | Enable or disable the authorization policy | `true` |
| `authorizationPolicy.labels` | Labels to apply to the authorization policy | `{app: istio-security, policy-type: default-deny}` |
| `authorizationPolicy.annotations` | Annotations to apply to the authorization policy | `{}` |
| `authorizationPolicy.selector` | Selector for workloads to apply the policy | `{}` (applies to all) |
| `authorizationPolicy.action` | Action to take (ALLOW, DENY, AUDIT, CUSTOM) | `DENY` |
| `authorizationPolicy.rules` | Rules for the authorization policy | `[]` (deny all) |

### Network Policy Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `networkPolicy.name` | Name of the network policy | `default-network-policy` |
| `networkPolicy.enabled` | Enable or disable the network policy | `true` |
| `networkPolicy.labels` | Labels to apply to the network policy | `{app: network-security, policy-type: default}` |
| `networkPolicy.annotations` | Annotations to apply to the network policy | `{}` |
| `networkPolicy.podSelector` | Pod selector for the network policy | `{}` (applies to all pods) |
| `networkPolicy.policyTypes` | Policy types (Ingress, Egress, or both) | `[Ingress, Egress]` |
| `networkPolicy.ingress.rules` | Ingress rules | `[]` (deny all ingress) |
| `networkPolicy.egress.rules` | Egress rules | DNS rules to kube-system |
| `global.namespace` | Namespace where the network policy will be deployed | `default` |

## Usage Examples

### Example 1: Authorization Policy with Allowed Paths

Create a `values-auth-custom.yaml`:

```yaml
authorizationPolicy:
  name: allow-health-checks
  namespace: istio-system
  enabled: true
  action: DENY
  rules:
    - to:
      - operation:
          methods: ["GET"]
          paths: ["/health", "/ready", "/metrics"]
```

Install:
```bash
helm install auth-policy ./authorization-policy -f values-auth-custom.yaml
```

### Example 2: Network Policy for Microservice

Create a `values-netpol-custom.yaml`:

```yaml
networkPolicy:
  name: frontend-network-policy
  enabled: true
  podSelector:
    matchLabels:
      app: frontend
  ingress:
    rules:
      - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        ports:
        - protocol: TCP
          port: 8080
  egress:
    rules:
      # Allow DNS
      - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
        ports:
        - protocol: UDP
          port: 53
      # Allow backend service
      - to:
        - podSelector:
            matchLabels:
              app: backend
        ports:
        - protocol: TCP
          port: 9090

global:
  namespace: production
```

Install:
```bash
helm install frontend-netpol ./network-policy -f values-netpol-custom.yaml -n production
```

### Example 3: Allow Traffic from Specific Service Accounts

Create a `values-auth-sa.yaml`:

```yaml
authorizationPolicy:
  name: allow-service-accounts
  namespace: istio-system
  enabled: true
  action: ALLOW
  rules:
    - from:
      - source:
          principals: 
            - "cluster.local/ns/default/sa/frontend-sa"
            - "cluster.local/ns/production/sa/backend-sa"
    - when:
      - key: source.namespace
        values: ["default", "production"]
```

## Validation

### Validate Helm Templates

#### Authorization Policy Validation

```bash
# Validate with default values
helm template authorization-policy ./authorization-policy

# Validate with custom values
helm template authorization-policy ./authorization-policy -f custom-values.yaml

# Dry run installation
helm install authorization-policy ./authorization-policy --dry-run --debug
```

#### Network Policy Validation

```bash
# Validate with default values
helm template network-policy ./network-policy

# Validate with custom namespace
helm template network-policy ./network-policy --namespace my-app

# Validate with custom values
helm template network-policy ./network-policy -f custom-values.yaml

# Dry run installation
helm install network-policy ./network-policy --dry-run --debug -n my-app
```

### Post-Installation Validation

#### Verify Authorization Policy

```bash
# Check if the authorization policy is created
kubectl get authorizationpolicy -n istio-system

# Describe the policy
kubectl describe authorizationpolicy default-deny-all -n istio-system

# Test the policy (should be denied)
kubectl exec -it <pod-name> -n <namespace> -- curl http://service.istio-system.svc.cluster.local
```

#### Verify Network Policy

```bash
# Check if the network policy is created
kubectl get networkpolicy -n <namespace>

# Describe the policy
kubectl describe networkpolicy default-network-policy -n <namespace>

# Test connectivity (use a test pod)
kubectl run test-pod --image=busybox --rm -it -- /bin/sh
# Inside the pod, test connectivity:
# wget -qO- --timeout=2 http://service-name:port
```

## Troubleshooting

### Common Issues

#### Authorization Policy Not Working

1. **Check Istio Installation**:
   ```bash
   kubectl get pods -n istio-system
   istioctl version
   ```

2. **Verify Sidecar Injection**:
   ```bash
   kubectl get pods -n <your-namespace> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
   ```

3. **Check Policy Status**:
   ```bash
   istioctl analyze -n istio-system
   ```

#### Network Policy Not Blocking Traffic

1. **Verify CNI Plugin Support**:
   - Ensure your CNI plugin supports Network Policies (e.g., Calico, Cilium, Weave Net)
   
2. **Check Pod Labels**:
   ```bash
   kubectl get pods -n <namespace> --show-labels
   ```

3. **Test with netshoot**:
   ```bash
   kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
   ```

### Debug Commands

```bash
# List all policies
kubectl get authorizationpolicy -A
kubectl get networkpolicy -A

# Check Helm releases
helm list -A

# Get Helm values
helm get values <release-name> -n <namespace>

# Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Development Guidelines

- Update chart version in `Chart.yaml` for any changes
- Update README.md with new configuration options
- Test charts with `helm lint` before committing
- Include examples for new features

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues, questions, or contributions, please open an issue in the GitHub repository.