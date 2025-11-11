
# Automate OpenShift Cluster Configuration After Provisioning

Part of the blog series at [eduard-marbach.de/post](https://eduard-marbach.de/post)

## Overview

OpenShift clusters can be provisioned in three main ways:
- **Installer-Provisioned Infrastructure (IPI)**: Automated deployment where the installer manages infrastructure
- **User-Provisioned Infrastructure (UPI)**: Manual infrastructure setup with OpenShift installation on existing resources
- **Managed Services**: Cloud provider managed solutions (e.g., Red Hat OpenShift on AWS, Azure Red Hat OpenShift)

### The Challenge

After deployment, OpenShift clusters come with default configurations requiring manual setup through the web console:
- Custom certificates and TLS configuration
- OAuth/SSO integration
- Proxy settings
- Operator subscriptions (LVMS, KubeVirt, OADP, etc.)

This manual approach is time-consuming, error-prone, and difficult to replicate across environments.

### The Solution: Configuration as Code

This repository demonstrates automating post-deployment configuration using **Kustomize** and **SOPS**, enabling:
- ✅ Declarative, version-controlled configuration
- ✅ Secure secret management with encryption
- ✅ Repeatable deployments across clusters
- ✅ Easy environment-specific customization via overlays

## Structure

```
kustomize/
├── base/                      # Base cluster configurations
│   ├── proxy-cluster.yaml
│   └── ingresscontroller-default.yaml
└── overlays/
    └── my_site/              # Site-specific overlay
        ├── apiserver-cluster.yaml
        ├── oauth.yaml
        ├── cert-*.enc.yaml   # SOPS encrypted secrets
        └── subscriptions/    # Operator subscriptions
```

## Prerequisites

- `kubectl` or `oc` CLI
- `kustomize` (v5.0+)
- `sops` for secret encryption
- `ksops` kustomize plugin

Install tools:
```bash
# Install kustomize
brew install kustomize

# Install sops
brew install sops

# Install ksops
brew install ksops
```

## Usage

### 1. Encrypt Secrets with SOPS

Create a `.sops.yaml` file in the repository root:

```yaml
creation_rules:
  - path_regex: \.enc\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    # Or use GPG:
    # pgp: YOUR_PGP_FINGERPRINT
```

Encrypt sensitive files:
```bash
cd kustomize/overlays/my_site

# Encrypt each secret file
sops -e -i cert-api-secret.enc.yaml
sops -e -i cert-wildcard-secret.enc.yaml
sops -e -i custom-ca-configmap.enc.yaml
sops -e -i oauth-secret.enc.yaml
```

### 2. Apply Configuration

```bash
# Build and preview configuration
kustomize build --enable-alpha-plugins kustomize/overlays/my_site

# Apply to cluster
kustomize build --enable-alpha-plugins kustomize/overlays/my_site | oc apply -f -
```

### 3. Approve Operator Subscriptions

**Important**: Initial application will partially fail because operator CRDs don't exist yet. This is expected behavior.

After the first apply:
1. Navigate to **Operators → Installed Operators** in the OpenShift console
2. Find pending operator installations
3. **Manually approve** each InstallPlan (security best practice)
4. Wait for operators to install their CRDs
5. Re-run the kustomize command:
   ```bash
   kustomize build --enable-alpha-plugins kustomize/overlays/my_site | oc apply -f -
   ```

## Key Components

- **Base Layer**: Shared cluster-wide configurations (proxy, ingress)
- **Overlay Layer**: Site-specific customizations (certificates, OAuth, subscriptions)
- **SOPS Integration**: Encrypted secrets stored safely in Git
- **Operator Subscriptions**: Automated installation of OADP, LVMS, NMState, KubeVirt

## Security Notes

- All `.enc.yaml` files should be encrypted with SOPS before committing
- Never commit unencrypted secrets to version control
- Use Age or GPG keys for SOPS encryption
- Operator approvals remain manual to maintain security control

## Customization

To adapt for your environment:
1. Copy `my_site` overlay to a new directory
2. Update certificates, OAuth settings, and subscriptions
3. Encrypt secrets with your SOPS key
4. Apply to your cluster

---

*For more details, visit the full blog post at [eduard-marbach.de/post](https://eduard-marbach.de/post)*
