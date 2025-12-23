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

**Location**: `gitclone/charts/authorization-policy`

**Description**: This chart deploys an Istio Authorization Policy that implements a default deny-all policy in the istio-system namespace. This ensures that all traffic is denied by default unless explicitly allowed.

**Key Features**:
- Default deny-all policy for enhanced security
- Configurable rules to allow specific traffic
- Support for workload selectors
- Customizable labels and annotations

### Network Policy Chart

**Location**: `gitclone/charts/network-policy`

**Description**: This chart deploys Kubernetes Network Policies to control traffic flow at the IP address or port level (OSI layer 3 or 4).

**Key Features**:
- Configurable ingress and egress rules
- Default DNS egress rules included
- Support for namespace and pod selectors
- Flexible policy types configuration

## Installation

### Adding the Helm Repository

```bash
# You're already at the root directory
# No need to clone or cd into any directory
```

### Installing Authorization Policy Chart

```bash
# Install with default values (default deny-all in istio-system)
helm install authorization-policy ./gitclone/charts/authorization-policy

# Install with custom values
helm install authorization-policy ./gitclone/charts/authorization-policy -f custom-values.yaml

# Install in a specific namespace (note: the policy itself applies to istio-system)
helm install authorization-policy ./gitclone/charts/authorization-policy -n security-policies --create-namespace
```

### Installing Network Policy Chart

```bash
# Install with default values in default namespace
helm install network-policy ./gitclone/charts/network-policy

# Install in a specific namespace
helm install network-policy ./gitclone/charts/network-policy -n my-app --create-namespace

# Install with custom values
helm install network-policy ./gitclone/charts/network-policy -f custom-values.yaml -n my-app
```

## Configuration

### Authorization Policy Configuration

| Parameter | Description | Default |
|-----------|-------------|---------||
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
|-----------|-------------|---------||
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

Create a `values-auth-custom.yaml` at the root:

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

Install from root:
```bash
helm install auth-policy ./gitclone/charts/authorization-policy -f values-auth-custom.yaml
```

### Example 2: Network Policy for Microservice

Create a `values-netpol-custom.yaml` at the root:

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

Install from root:
```bash
helm install frontend-netpol ./gitclone/charts/network-policy -f values-netpol-custom.yaml -n production
```

### Example 3: Allow Traffic from Specific Service Accounts

Create a `values-auth-sa.yaml` at the root:

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

Install from root:
```bash
helm install sa-policy ./gitclone/charts/authorization-policy -f values-auth-sa.yaml
```

## Validation

### Validate Helm Templates

#### Authorization Policy Validation

```bash
# Validate with default values from root
helm template authorization-policy ./gitclone/charts/authorization-policy

# Validate with custom values from root
helm template authorization-policy ./gitclone/charts/authorization-policy -f custom-values.yaml

# Dry run installation from root
helm install authorization-policy ./gitclone/charts/authorization-policy --dry-run --debug
```

#### Network Policy Validation

```bash
# Validate with default values from root
helm template network-policy ./gitclone/charts/network-policy

# Validate with custom namespace from root
helm template network-policy ./gitclone/charts/network-policy --namespace my-app

# Validate with custom values from root
helm template network-policy ./gitclone/charts/network-policy -f custom-values.yaml

# Dry run installation from root
helm install network-policy ./gitclone/charts/network-policy --dry-run --debug -n my-app
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

## Quick Start Commands

Here are the most common commands you'll run from the root directory:

```bash
# Install both charts with default settings
helm install authorization-policy ./gitclone/charts/authorization-policy
helm install network-policy ./gitclone/charts/network-policy

# Validate templates before installation
helm template authorization-policy ./gitclone/charts/authorization-policy
helm template network-policy ./gitclone/charts/network-policy

# Check installed releases
helm list -A

# Upgrade existing releases
helm upgrade authorization-policy ./gitclone/charts/authorization-policy
helm upgrade network-policy ./gitclone/charts/network-policy

# Uninstall charts
helm uninstall authorization-policy
helm uninstall network-policy
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