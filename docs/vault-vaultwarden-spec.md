# Vault + Vaultwarden Deployment Specification

**Date:** 2026-03-07
**Status:** Approved — ready for implementation
**Repo:** fleet-homelab (cluster-side) + infra (Vault VM provisioning)

---

## 1. Problem Statement and Goals

This document specifies the full deployment of HashiCorp Vault as an external secrets management backend for the homelab Kubernetes cluster, the Secrets Store CSI Driver to bridge Vault secrets into pods as Kubernetes Secrets, Vaultwarden (a self-hosted Bitwarden-compatible password manager) exposed publicly via Cloudflare Tunnel, and Cloudflared as its own isolated workload.

The primary goals are:

- Vault runs on a dedicated external VM, fully isolated from the Kubernetes cluster. If the cluster is compromised, Vault is not automatically compromised. This is the central architectural motivation and must not be compromised in implementation.
- A single source of truth for all application secrets in the cluster, served by Vault with per-app least-privilege policies.
- Vaultwarden accessible at `vault.khoahuynh.dev` via Cloudflare Tunnel with no direct ingress exposure.
- Full GitOps management of all Kubernetes resources via FluxCD, with a documented manual bootstrap sequence for the parts that cannot be safely automated.
- A scalable policy structure in Vault that can accommodate many future applications without redesign.

---

## 2. Architecture Overview

### 2.1 Cluster Context

- 3-node kubeadm cluster: 1 control-plane, 2 workers, Fedora 42
- Cilium CNI with Gateway API enabled
- NFS StorageClass `nfs-client` (default), NFS server at `192.168.122.53`
- cert-manager with self-signed ClusterIssuer
- kube-prometheus-stack for monitoring
- FluxCD watching `ssh://git@github.com/khoahuynhdev/fleet-homelab`, branch `main`, path `./clusters/homelab`
- Vault runs OUTSIDE the cluster on a dedicated VM at `192.168.122.55`

### 2.2 Vault VM

Vault is provisioned as a standalone VM on the libvirt/KVM hypervisor via Terraform in the infra repo, using the existing `modules/libvirt-domain` module. It is not a Kubernetes workload.

| Property        | Value                                                   |
|-----------------|---------------------------------------------------------|
| OS              | Fedora 42 (matching cluster nodes)                      |
| IP              | `192.168.122.55` (static, via cloud-init)               |
| Network         | `192.168.122.0/24` (same libvirt network)               |
| Vault install   | HashiCorp RPM package, systemd service                  |
| TLS CA          | step-ca, co-located on the Vault VM                     |
| Vault address   | `https://192.168.122.55:8200`                           |
| Data directory  | `/opt/vault/data`                                       |
| Unseal          | systemd timer, keys in `/etc/vault.d/unseal-keys`       |
| Backup target   | NFS server at `192.168.122.53`                          |

The IP `192.168.122.55` is chosen to avoid all existing cluster nodes (.34, .119, .132), the NFS server (.53), and the MetalLB pool (.128–.191).

### 2.3 Namespace Layout (cluster-side)

| Namespace      | Workloads                                      |
|----------------|------------------------------------------------|
| `vaultwarden`  | Vaultwarden Deployment                         |
| `cloudflared`  | Cloudflared Deployment                         |
| `csi-secrets`  | Secrets Store CSI Driver, Vault CSI Provider   |

There is no `vault` namespace. Vault does not run inside the cluster.

### 2.4 Fleet-Homelab Directory Structure

```
clusters/homelab/
├── flux-system/                    # Managed by flux bootstrap — do not edit
├── sources/                        # HelmRepository objects
│   ├── jetstack.yaml
│   ├── metrics-server.yaml
│   ├── hashicorp.yaml              # NEW: HashiCorp Helm repo (for CSI provider only)
│   └── secrets-store-csi.yaml     # NEW: Secrets Store CSI Helm repo
├── releases/                       # Infrastructure-level HelmReleases
│   ├── cert-manager.yaml
│   ├── metrics-server.yaml
│   ├── secrets-store-csi-driver.yaml  # NEW
│   └── vault-csi-provider.yaml        # NEW
├── apps/                           # User-facing applications
│   ├── vaultwarden/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── pvc.yaml
│   │   ├── service-account.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── secret-provider-class.yaml
│   └── cloudflared/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── service-account.yaml
│       ├── deployment.yaml
│       └── secret-provider-class.yaml
├── releases-kustomization.yaml     # NEW
└── apps-kustomization.yaml         # MODIFIED: dependsOn releases
```

Note: there is no `services/` directory yet. It should be introduced when a second backend service (beyond Vault, which is external) is added to the cluster. For now the FluxCD dependency chain is two layers: releases -> apps.

### 2.5 FluxCD Dependency Chain

```
flux-system Kustomization
    └── releases-kustomization.yaml   (CSI Driver, Vault CSI Provider HelmReleases)
            └── apps-kustomization.yaml   (Vaultwarden, Cloudflared)
```

Apps depend on releases because the CSI Driver CRDs must exist before SecretProviderClass objects and CSI volumes can be used by Vaultwarden and Cloudflared pods.

---

## 3. FluxCD Kustomization Objects

### 3.1 releases-kustomization.yaml

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: releases
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./clusters/homelab/releases
  prune: true
```

### 3.2 apps-kustomization.yaml (modified from current)

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./clusters/homelab/apps
  prune: true
  dependsOn:
    - name: releases
```

---

## 4. Secrets Store CSI Driver

### 4.1 Deployment Method

HelmRelease in `releases/`, consistent with cert-manager and metrics-server patterns.

### 4.2 HelmRepository Source

```yaml
# sources/secrets-store-csi.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: secrets-store-csi
  namespace: flux-system
spec:
  interval: 12h
  url: https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
```

### 4.3 HelmRelease

```yaml
# releases/secrets-store-csi-driver.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: secrets-store-csi-driver
  namespace: csi-secrets
spec:
  interval: 30m
  chart:
    spec:
      chart: secrets-store-csi-driver
      version: "1.4.x"
      sourceRef:
        kind: HelmRepository
        name: secrets-store-csi
        namespace: flux-system
  install:
    createNamespace: true
  values:
    syncSecret:
      enabled: true     # Required: enables syncing CSI secrets to K8s Secrets
    enableSecretRotation: false
```

`syncSecret.enabled: true` is required because the manifests use `secretObjects` in SecretProviderClass to project Vault secrets into native K8s Secrets consumed by pods via `secretKeyRef`.

### 4.4 Vault CSI Provider

The Vault CSI Provider is a DaemonSet that the CSI Driver calls to fetch secrets from Vault. It is deployed via HelmRelease alongside the CSI Driver. It connects to the external Vault VM over TLS.

```yaml
# releases/vault-csi-provider.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: vault-csi-provider
  namespace: csi-secrets
spec:
  interval: 30m
  chart:
    spec:
      chart: vault
      version: "0.29.x"
      sourceRef:
        kind: HelmRepository
        name: hashicorp
        namespace: flux-system
  install:
    createNamespace: false   # namespace created by secrets-store-csi-driver release
  values:
    server:
      enabled: false
    injector:
      enabled: false
    csi:
      enabled: true
      agent:
        enabled: false
```

```yaml
# sources/hashicorp.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: hashicorp
  namespace: flux-system
spec:
  interval: 12h
  url: https://helm.releases.hashicorp.com
```

---

## 5. Vault VM Provisioning (infra repo)

### 5.1 Terraform Module

A new environment or module in the infra repo provisions the Vault VM using the existing `modules/libvirt-domain` module. VM sizing is left to the implementer's judgment using the standardized locals (`local.memory`, `local.cpus`, `local.disk_sizes`).

The cloud-init configuration for the Vault VM must at minimum:

- Set a static IP of `192.168.122.55` via netplan or NetworkManager
- Install the HashiCorp RPM repository and install `vault`
- Install `step-ca` and `step` CLI (for the private CA)
- Create the `vault` system user and required directories
- Write the initial `vault.hcl` configuration file
- Enable and start the `vault` systemd service
- Inject the Kubernetes API server URL (`https://192.168.122.119:6443`) and the cluster CA certificate as files on disk, so the bootstrap sequence can configure Vault's Kubernetes auth method without manual copying

The cluster CA cert can be obtained from the control-plane node at `/etc/kubernetes/pki/ca.crt`. It should be passed into the Terraform module as a variable and written to a file on the Vault VM (e.g., `/etc/vault.d/k8s-ca.crt`) via cloud-init.

### 5.2 Vault Configuration File

```hcl
# /etc/vault.d/vault.hcl
ui = true

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault.d/tls/vault.crt"
  tls_key_file  = "/etc/vault.d/tls/vault.key"
}

storage "file" {
  path = "/opt/vault/data"
}

telemetry {
  prometheus_retention_time = "30s"
  disable_hostname          = true
}
```

TLS is mandatory. The cert and key are issued by step-ca (see section 5.3).

### 5.3 TLS via step-ca

step-ca runs on the Vault VM itself as a systemd service, acting as a private CA scoped to issuing Vault's TLS certificate. It is not a general-purpose CA at this stage.

Bootstrap sequence for step-ca (run once on the VM after provisioning):

```bash
# Initialize step-ca
step ca init \
  --name="Homelab Vault CA" \
  --dns="192.168.122.55" \
  --address=":8443" \
  --provisioner="vault-vm"

# Issue the initial Vault TLS cert
step ca certificate "192.168.122.55" \
  /etc/vault.d/tls/vault.crt \
  /etc/vault.d/tls/vault.key \
  --san="192.168.122.55" \
  --not-after=8760h   # 1 year; renew via systemd timer

# Enable and start step-ca
systemctl enable --now step-ca
```

step-ca handles automatic renewal via its built-in renewal mechanism. A systemd timer using `step ca renew` should be configured to renew the Vault cert before expiry.

The step-ca root CA certificate must be distributed to the Kubernetes cluster so the CSI provider and any other in-cluster workload talking to Vault can verify the TLS connection. This is done as a ConfigMap in the relevant namespaces during the cluster-side bootstrap (see section 9, Part B).

### 5.4 Auto-Unseal via systemd Timer

Every time the Vault process starts — reboot, crash, or manual restart — it comes up sealed. The data on disk is encrypted with a master key that Vault does not persist in memory across restarts. It refuses to serve any secrets until enough unseal key shares are provided to reconstruct the master key in RAM. With 5 shares and a threshold of 3, any 3 of the 5 key shares are sufficient to unseal. Once unsealed, Vault is fully operational until the next restart.

The unseal keys are stored on the Vault VM at `/etc/vault.d/unseal-keys`, owned by root, mode 0400. This file is created manually during the bootstrap sequence (see section 9) and is never committed to Git or stored in Terraform state.

```bash
# /etc/vault.d/unseal-keys
KEY1=<unseal-key-1>
KEY2=<unseal-key-2>
KEY3=<unseal-key-3>
```

```bash
# /usr/local/bin/vault-unseal.sh
#!/bin/bash
set -e
source /etc/vault.d/unseal-keys
export VAULT_ADDR="https://192.168.122.55:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"

STATUS=$(vault status -format=json 2>/dev/null || true)
SEALED=$(echo "$STATUS" | grep '"sealed"' | grep -o 'true\|false')

if [ "$SEALED" = "true" ]; then
  echo "Vault is sealed. Unsealing..."
  vault operator unseal "$KEY1"
  vault operator unseal "$KEY2"
  vault operator unseal "$KEY3"
  echo "Unseal complete."
else
  echo "Vault is not sealed. Nothing to do."
fi
```

```ini
# /etc/systemd/system/vault-unseal.service
[Unit]
Description=Vault unseal check
After=vault.service
Wants=vault.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/vault-unseal.sh
```

```ini
# /etc/systemd/system/vault-unseal.timer
[Unit]
Description=Run vault-unseal check every minute

[Timer]
OnBootSec=30s
OnUnitActiveSec=1min

[Install]
WantedBy=timers.target
```

After any reboot, Vault is back in service within approximately 30–60 seconds without human intervention.

The unseal keys file on the VM is a deliberate trust trade-off: anyone with root access to the Vault VM can read the unseal keys. This is acceptable here because the VM is not shared and you control the hypervisor. It is strictly better than the in-cluster alternative where unseal keys in a Kubernetes Secret are accessible to anyone with sufficient RBAC permissions inside the cluster — which is precisely the blast radius you are trying to reduce.

### 5.5 Backup via systemd Timer

A systemd timer on the Vault VM runs a periodic backup to the NFS server at `192.168.122.53`. The NFS share is mounted on the Vault VM at `/mnt/nfs-backup`.

```bash
# /usr/local/bin/vault-backup.sh
#!/bin/bash
set -e
BACKUP_DIR="/mnt/nfs-backup/vault"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

mkdir -p "$BACKUP_DIR"

# Vault file backend: tar the data directory directly
tar -czf "$BACKUP_DIR/vault-data-$TIMESTAMP.tar.gz" /opt/vault/data

# Retain only the last 7 backups
ls -t "$BACKUP_DIR"/vault-data-*.tar.gz | tail -n +8 | xargs -r rm --

echo "Backup complete: vault-data-$TIMESTAMP.tar.gz"
```

```ini
# /etc/systemd/system/vault-backup.timer
[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

The NFS mount should be added to `/etc/fstab` on the Vault VM during or after provisioning. A full data loss on the VM (without a recent backup) requires complete re-bootstrapping of all secrets and re-configuration of all auth methods and policies.

### 5.6 Prometheus Metrics

Vault's telemetry block (included in vault.hcl) enables the `/v1/sys/metrics` endpoint in Prometheus format at `https://192.168.122.55:8200/v1/sys/metrics`. Because Vault is external, a ServiceMonitor cannot target it directly. Prometheus scraping of external Vault is a deferred item — see section 12.

---

## 6. Vault Policy Structure

### 6.1 Design Principle

One policy per application, with least-privilege path access. Each application gets a dedicated KV path (`secret/data/<appname>/*`), its own Vault policy, and its own Kubernetes auth role bound to its ServiceAccount in its namespace. This structure scales to many applications without modification.

### 6.2 KV Secrets Engine

Mounted at `secret/` using KV version 2. Enabled during bootstrap:

```bash
vault secrets enable -path=secret kv-v2
```

### 6.3 Policy Definitions

```hcl
# vaultwarden policy
path "secret/data/vaultwarden/*" {
  capabilities = ["read"]
}
path "secret/metadata/vaultwarden/*" {
  capabilities = ["read"]
}
```

```hcl
# cloudflared policy
path "secret/data/cloudflared/*" {
  capabilities = ["read"]
}
path "secret/metadata/cloudflared/*" {
  capabilities = ["read"]
}
```

For each new application added in the future, create a new policy following the same pattern: `secret/data/<appname>/*` read-only.

### 6.4 Kubernetes Auth Roles

```bash
# vaultwarden role
vault write auth/kubernetes/role/vaultwarden \
  bound_service_account_names=vaultwarden \
  bound_service_account_namespaces=vaultwarden \
  policies=vaultwarden \
  ttl=1h

# cloudflared role
vault write auth/kubernetes/role/cloudflared \
  bound_service_account_names=cloudflared \
  bound_service_account_namespaces=cloudflared \
  policies=cloudflared \
  ttl=1h
```

### 6.5 Secret Paths

| Application  | Vault Path                | Keys          |
|-------------|---------------------------|---------------|
| vaultwarden | `secret/data/vaultwarden` | `ADMIN_TOKEN` |
| cloudflared | `secret/data/cloudflared` | `token`       |

---

## 7. Vaultwarden Deployment

### 7.1 No changes to core design

Vaultwarden stays in the `vaultwarden` namespace. The main change from the original spec is the `vaultAddress` in its SecretProviderClass, which now points to the external Vault VM over TLS instead of an in-cluster service.

### 7.2 Deployment Spec

- Image: `vaultwarden/server:1.33.2` (pin to specific version)
- Replicas: 1
- Strategy: `Recreate` (correct for a single-instance stateful app on ReadWriteOnce PVC)
- PVC: 5Gi, `nfs-client` StorageClass
- Signups disabled: `SIGNUPS_ALLOWED: "false"`
- Domain: `https://vault.khoahuynh.dev`
- Admin token sourced from Vault via CSI: `secret/data/vaultwarden` -> `ADMIN_TOKEN`
- Resource requests: `cpu: 50m, memory: 128Mi`; limits: `memory: 256Mi`
- Liveness and readiness probes: `GET /alive`
- The `vault-ca-cert` ConfigMap must be mounted at `/etc/vault-ca/` in the pod (see section 9, Step B1)

### 7.3 SecretProviderClass

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vaultwarden-secrets
  namespace: vaultwarden
spec:
  provider: vault
  parameters:
    vaultAddress: "https://192.168.122.55:8200"
    vaultCACertPath: "/etc/vault-ca/ca.crt"
    roleName: "vaultwarden"
    objects: |
      - objectName: "admin-token"
        secretPath: "secret/data/vaultwarden"
        secretKey: "ADMIN_TOKEN"
  secretObjects:
    - secretName: vaultwarden-synced
      type: Opaque
      data:
        - objectName: admin-token
          key: admin-token
```

### 7.4 Service

ClusterIP service on port 80, targeting the vaultwarden pod.

---

## 8. Cloudflared Deployment

### 8.1 Isolation

Cloudflared is an independent workload with its own secret (the Cloudflare tunnel token), its own Vault policy, and its own trust boundary. It runs in the `cloudflared` namespace.

### 8.2 Directory: apps/cloudflared/

Files required:

- `namespace.yaml` — creates `cloudflared` namespace
- `service-account.yaml` — ServiceAccount named `cloudflared`
- `deployment.yaml` — Cloudflared Deployment
- `secret-provider-class.yaml` — SecretProviderClass for the tunnel token
- `kustomization.yaml` — lists all of the above

### 8.3 Deployment Spec

- Image: `cloudflare/cloudflared:2025.4.2` (pin to specific version)
- Replicas: 1
- Tunnel token sourced from Vault via CSI: `secret/data/cloudflared` -> `token`
- Resource requests: `cpu: 25m, memory: 64Mi`; limits: `memory: 128Mi`
- The tunnel is configured in Cloudflare Zero Trust dashboard to route `vault.khoahuynh.dev` -> `http://vaultwarden.vaultwarden.svc.cluster.local:80`
- The `vault-ca-cert` ConfigMap must be mounted at `/etc/vault-ca/` in the pod (see section 9, Step B1)

### 8.4 SecretProviderClass

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: cloudflared-secrets
  namespace: cloudflared
spec:
  provider: vault
  parameters:
    vaultAddress: "https://192.168.122.55:8200"
    vaultCACertPath: "/etc/vault-ca/ca.crt"
    roleName: "cloudflared"
    objects: |
      - objectName: "tunnel-token"
        secretPath: "secret/data/cloudflared"
        secretKey: "token"
  secretObjects:
    - secretName: cloudflared-synced
      type: Opaque
      data:
        - objectName: tunnel-token
          key: tunnel-token
```

### 8.5 Cloudflare Tunnel Configuration (manual, in Cloudflare dashboard)

- Tunnel name: `homelab` (or similar)
- Public hostname: `vault.khoahuynh.dev`
- Service: `http://vaultwarden.vaultwarden.svc.cluster.local:80`
- The tunnel token is copied from the dashboard and stored in Vault at `secret/data/cloudflared` with key `token` during the bootstrap sequence.

---

## 9. Bootstrap Sequence

The bootstrap sequence has two parts: provisioning and configuring the Vault VM (done on the VM via SSH), then preparing the cluster to trust and connect to it (done via kubectl).

### Part A: Vault VM Bootstrap (run on the VM via SSH)

#### Step A1: Verify step-ca and Vault are running

```bash
systemctl status step-ca
systemctl status vault
export VAULT_ADDR="https://192.168.122.55:8200"
export VAULT_CACERT="/etc/vault.d/tls/ca.crt"
vault status   # should show: Initialized: false, Sealed: true
```

#### Step A2: Initialize Vault

```bash
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > /root/vault-init.json
```

Copy `/root/vault-init.json` off the VM to a secure location (password manager, encrypted file). It contains all unseal keys and the initial root token. Do not leave it on the VM long-term and do not commit it to Git.

#### Step A3: Create the unseal keys file and start the unseal timer

```bash
KEY1=$(jq -r '.unseal_keys_b64[0]' /root/vault-init.json)
KEY2=$(jq -r '.unseal_keys_b64[1]' /root/vault-init.json)
KEY3=$(jq -r '.unseal_keys_b64[2]' /root/vault-init.json)

cat > /etc/vault.d/unseal-keys <<EOF
KEY1=$KEY1
KEY2=$KEY2
KEY3=$KEY3
EOF
chmod 0400 /etc/vault.d/unseal-keys

systemctl enable --now vault-unseal.timer
```

Wait up to 60 seconds for the timer to fire and unseal Vault, then verify:

```bash
vault status   # should show: Sealed: false
```

#### Step A4: Configure Vault

```bash
ROOT_TOKEN=$(jq -r '.root_token' /root/vault-init.json)
export VAULT_TOKEN="$ROOT_TOKEN"

# Enable KV v2
vault secrets enable -path=secret kv-v2

# Enable audit log to stdout (captured by journald)
vault audit enable file file_path=stdout

# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure Kubernetes auth using the injected cluster CA cert and API server URL
# Both files were written to the VM via cloud-init during Terraform provisioning
vault write auth/kubernetes/config \
  kubernetes_host="https://192.168.122.119:6443" \
  kubernetes_ca_cert=@/etc/vault.d/k8s-ca.crt

# Write policies
vault policy write vaultwarden - <<'EOF'
path "secret/data/vaultwarden/*" { capabilities = ["read"] }
path "secret/metadata/vaultwarden/*" { capabilities = ["read"] }
EOF

vault policy write cloudflared - <<'EOF'
path "secret/data/cloudflared/*" { capabilities = ["read"] }
path "secret/metadata/cloudflared/*" { capabilities = ["read"] }
EOF

# Create Kubernetes auth roles
vault write auth/kubernetes/role/vaultwarden \
  bound_service_account_names=vaultwarden \
  bound_service_account_namespaces=vaultwarden \
  policies=vaultwarden \
  ttl=1h

vault write auth/kubernetes/role/cloudflared \
  bound_service_account_names=cloudflared \
  bound_service_account_namespaces=cloudflared \
  policies=cloudflared \
  ttl=1h

# Populate application secrets
vault kv put secret/vaultwarden ADMIN_TOKEN="$(openssl rand -base64 48)"
vault kv put secret/cloudflared token="<paste cloudflare tunnel token here>"
```

#### Step A5: Configure NFS backup

```bash
# Add to /etc/fstab (adjust NFS export path as needed)
echo "192.168.122.53:/exports/vault-backup /mnt/nfs-backup nfs defaults 0 0" >> /etc/fstab
mkdir -p /mnt/nfs-backup
mount /mnt/nfs-backup

systemctl enable --now vault-backup.timer
```

### Part B: Cluster-Side Bootstrap (run from a machine with kubectl access)

#### Step B1: Distribute the step-ca root certificate to the cluster

Copy the step-ca root CA cert from the Vault VM:

```bash
scp user@192.168.122.55:/etc/vault.d/tls/ca.crt ./vault-ca.crt
```

Create a ConfigMap in every namespace that talks to Vault:

```bash
for ns in csi-secrets vaultwarden cloudflared; do
  kubectl create configmap vault-ca-cert \
    -n "$ns" \
    --from-file=ca.crt=./vault-ca.crt
done
```

These ConfigMaps are created manually and are not committed to Git. Each Deployment that uses CSI-mounted secrets from Vault must mount this ConfigMap as a volume at `/etc/vault-ca/`:

```yaml
volumes:
  - name: vault-ca
    configMap:
      name: vault-ca-cert
volumeMounts:
  - name: vault-ca
    mountPath: /etc/vault-ca
    readOnly: true
```

#### Step B2: Push GitOps manifests and trigger reconciliation

```bash
git push origin main
# FluxCD reconciles: releases (CSI driver + Vault CSI provider) -> apps (Vaultwarden + Cloudflared)
```

Verify the CSI driver and provider are running:

```bash
kubectl get pods -n csi-secrets
```

Verify Vaultwarden and Cloudflared pods reach Running state:

```bash
kubectl get pods -n vaultwarden
kubectl get pods -n cloudflared
```

---

## 10. Network Considerations

### 10.1 VM-to-Cluster and Cluster-to-VM Traffic

All VMs are on `192.168.122.0/24`. The cluster nodes can reach the Vault VM at `192.168.122.55:8200` without any additional routing. The Vault VM can reach the Kubernetes API server at `192.168.122.119:6443` for Kubernetes auth token verification — this is required for every CSI secret mount request.

If `firewalld` is active on the Vault VM, port 8200 must be opened:

```bash
firewall-cmd --permanent --add-port=8200/tcp
firewall-cmd --reload
```

### 10.2 Cross-Namespace Traffic (cluster-internal)

Cloudflared (in `cloudflared` namespace) needs to reach Vaultwarden (in `vaultwarden` namespace) at `vaultwarden.vaultwarden.svc.cluster.local:80`. With Cilium as the CNI, cross-namespace traffic is permitted by default unless NetworkPolicies restrict it.

No NetworkPolicies are defined in the initial deployment. If NetworkPolicies are added in a future phase, the required cross-namespace and egress rules are:

- `cloudflared` namespace -> `vaultwarden` namespace on port 80
- `vaultwarden` namespace -> egress to `192.168.122.55:8200`
- `cloudflared` namespace -> egress to `192.168.122.55:8200`
- `csi-secrets` namespace (CSI driver DaemonSet) -> egress to `192.168.122.55:8200`

### 10.3 No In-Cluster Vault Service

There is no `vault.vault.svc:8200` ClusterIP service. All references to `vaultAddress` in SecretProviderClass objects use `https://192.168.122.55:8200`. This is the single address used by the Vault CSI Provider for all secret fetch requests.

---

## 11. Manifest Changes Required to Current Draft

The following specific changes are needed relative to any previously drafted manifests.

### Remove entirely
- `services/vault/` directory and all contents (HelmRelease, unseal CronJob, bootstrap Job, ServiceMonitor)
- `services-kustomization.yaml` (the services layer does not exist yet)
- Any reference to `vault.vault.svc:8200` anywhere — replace with `https://192.168.122.55:8200`
- The `vault` namespace

### apps/vaultwarden/secret-provider-class.yaml
Remove the `cloudflared-secrets` SecretProviderClass object. Update `vaultAddress` to `https://192.168.122.55:8200` and add `vaultCACertPath: "/etc/vault-ca/ca.crt"`.

### apps/vaultwarden/deployment.yaml
Add the `vault-ca` volume (from `vault-ca-cert` ConfigMap) and mount it at `/etc/vault-ca`.

### apps/vaultwarden/cloudflared.yaml
Delete this file entirely.

### apps/vaultwarden/kustomization.yaml
Remove `cloudflared.yaml` from the resources list.

### apps-kustomization.yaml
Change `dependsOn` to reference `releases` (not `services`, which no longer exists).

### New files to create (fleet-homelab repo)
`apps/cloudflared/` directory with all manifests, `releases/secrets-store-csi-driver.yaml`, `releases/vault-csi-provider.yaml`, `sources/hashicorp.yaml`, `sources/secrets-store-csi.yaml`, `releases-kustomization.yaml`.

### New files to create (infra repo)
Terraform environment or module for the Vault VM, including cloud-init configuration for static IP, HashiCorp RPM install, step-ca install, vault.hcl, systemd units for vault-unseal and vault-backup, and injection of `/etc/vault.d/k8s-ca.crt`.

---

## 12. Deferred Items

**Prometheus metrics for external Vault.** The Vault VM exposes `/v1/sys/metrics` on port 8200. Because Vault is external, a ServiceMonitor cannot target it directly. A future phase should add an external scrape config to the kube-prometheus-stack values targeting `192.168.122.55:8200`. At minimum, an alert for Vault being sealed for more than 2 minutes should be added, as it is the most operationally impactful failure mode.

**NetworkPolicies.** No NetworkPolicies are defined. A future security hardening phase should add namespace-scoped policies restricting cross-namespace and cluster-to-external traffic to only the required flows listed in section 10.2.

**Secret rotation.** No automated rotation of the Vaultwarden admin token or Cloudflare tunnel token is implemented. Both are static secrets written manually during bootstrap.

**step-ca as general-purpose homelab CA.** step-ca is currently scoped only to Vault's TLS cert. A future phase could expand it to issue certs for other VMs, ingress controllers, or internal services, replacing or augmenting the cluster's self-signed ClusterIssuer.

**services/ layer in FluxCD.** The `services/` directory and `services-kustomization.yaml` should be introduced when a second backend service (beyond Vault, which is external) is deployed inside the cluster. At that point the dependency chain becomes releases -> services -> apps.

**Vault HA.** Vault runs standalone on a single VM. A future phase could provision a second Vault VM and configure Raft replication, eliminating the single point of failure. This is out of scope for the initial deployment.

---

## 13. Implementation Order

### Phase 1: infra repo — provision the Vault VM

1. Create Vault VM Terraform module/environment using `modules/libvirt-domain`
2. Write cloud-init config: static IP `192.168.122.55`, HashiCorp RPM repo, `vault` + `step` + `step-ca` packages, vault.hcl, systemd units for vault-unseal and vault-backup, write `/etc/vault.d/k8s-ca.crt` from Terraform variable
3. `terraform apply` to provision the VM
4. SSH into the VM and run Part A of the bootstrap sequence (section 9): initialize Vault, create unseal keys file, start unseal timer, configure Vault (KV, audit, Kubernetes auth, policies, roles, secrets), configure NFS backup

### Phase 2: fleet-homelab repo — cluster-side setup

5. Add `sources/hashicorp.yaml` and `sources/secrets-store-csi.yaml`
6. Add `releases/secrets-store-csi-driver.yaml` and `releases/vault-csi-provider.yaml`
7. Add `releases-kustomization.yaml`
8. Remove `services/vault/` directory entirely
9. Fix `apps/vaultwarden/secret-provider-class.yaml`: remove cloudflared object, update vaultAddress to `https://192.168.122.55:8200`, add `vaultCACertPath`
10. Update `apps/vaultwarden/deployment.yaml` to mount `vault-ca-cert` ConfigMap
11. Delete `apps/vaultwarden/cloudflared.yaml`
12. Update `apps/vaultwarden/kustomization.yaml`
13. Add `apps/cloudflared/` directory with all manifests (namespace, service-account, deployment with vault-ca mount, secret-provider-class, kustomization)
14. Update `apps-kustomization.yaml` to `dependsOn: [{name: releases}]`
15. Commit and push to trigger FluxCD reconciliation

### Phase 3: cluster-side bootstrap (manual)

16. Run Part B of the bootstrap sequence (section 9): copy step-ca root cert from Vault VM, create `vault-ca-cert` ConfigMaps in `csi-secrets`, `vaultwarden`, and `cloudflared` namespaces
17. Verify CSI driver and provider pods are running in `csi-secrets`
18. Verify Vaultwarden and Cloudflared pods reach Running state
