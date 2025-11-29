# Homelab

[![CI](https://github.com/terenceponce/homelab/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/terenceponce/homelab/actions/workflows/ci.yaml)

Kubernetes manifests for my homelab, managed with [Flux](https://fluxcd.io/).

## Structure

```
.
├── apps/                   # Application workloads
│   ├── base/               # Base manifests
│   └── overlays/           # Environment-specific overrides
├── infrastructure/         # Cluster infrastructure (namespaces, storage)
│   ├── base/
│   └── overlays/
└── clusters/               # Flux configuration per cluster
    └── production/
```

## CI Checks

Pull requests are validated with:

| Check | Tool | Description |
|-------|------|-------------|
| Validate Kustomize | kustomize + kubeconform | Builds overlays and validates against K8s schemas |
| Kube Linter | kube-linter | Security and best practices |
| Detect Deprecated APIs | pluto | Finds deprecated Kubernetes APIs |

## Setup

### Prerequisites

- A Kubernetes cluster (We used [K3s](https://k3s.io/))
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Flux CLI](https://fluxcd.io/flux/installation/)
- [SOPS](https://github.com/getsops/sops)
- [age](https://github.com/FiloSottile/age)

### 1. Generate an age key pair

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

The output will look like:

```
# created: 2025-11-24T...
# public key: age1j5lrkmy3aqk4ht2mxn74rk5arux52wnaxh5m6lhe3nwskhe3pptqyu8nzf
AGE-SECRET-KEY-1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

**Back up this file to a password manager.** If you lose it, you cannot decrypt your secrets.

### 2. Configure SOPS

Update `.sops.yaml` with your public key:

```yaml
creation_rules:
  - path_regex: .*secret.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: <your-age-public-key>
```

This config:
- Only encrypts files with "secret" in the filename
- Only encrypts the `data` and `stringData` fields (metadata stays readable)

Set the environment variable so SOPS can find your key:

```bash
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
```

Add this to your shell profile (`.bashrc`, `.zshrc`, etc.) to make it permanent.

### 3. Create a GitHub Personal Access Token

Flux needs a GitHub token to access the repository:

1. Go to https://github.com/settings/tokens
2. Click "Generate new token (classic)"
3. Select the `repo` scope (Full control of private repositories)
4. Generate and copy the token

### 4. Bootstrap Flux

```bash
GITHUB_TOKEN=<your-token> flux bootstrap github \
  --owner=<github-username> \
  --repository=homelab \
  --path=clusters/production \
  --personal
```

### 5. Create the SOPS secret in the cluster

Flux needs the age private key to decrypt secrets:

```bash
cat ~/.config/sops/age/keys.txt | \
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

### 6. Working with secrets

Encrypt a secret file:

```bash
sops --encrypt --in-place path/to/secret.yaml
```

Edit an encrypted secret (decrypts, opens editor, re-encrypts on save):

```bash
sops path/to/secret.yaml
```

View decrypted contents without editing:

```bash
sops -d path/to/secret.yaml
```

## New Machine Setup

When setting up on a new machine:

1. Retrieve age key from password manager
2. Save to `~/.config/sops/age/keys.txt`
3. Set permissions: `chmod 600 ~/.config/sops/age/keys.txt`
4. Set environment variable: `export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt`
5. Create sops-age secret in the cluster:
   ```bash
   cat ~/.config/sops/age/keys.txt | \
   kubectl create secret generic sops-age \
     --namespace=flux-system \
     --from-file=age.agekey=/dev/stdin
   ```

## Troubleshooting

### K3s won't start after reboot

Try these steps in order:

1. **Simple restart**: `sudo systemctl restart k3s` (wait 60s)
2. **Kill orphaned processes**: `sudo k3s-killall.sh && sudo systemctl start k3s`
3. **Full reset** (last resort - loses cluster state):
   ```bash
   sudo systemctl stop k3s
   sudo k3s-killall.sh
   sudo rm -rf /var/lib/rancher/k3s/server/db/
   sudo rm -rf /var/lib/rancher/k3s/agent/
   sudo systemctl start k3s
   ```

After a full reset, re-bootstrap Flux (step 4) and recreate the sops-age secret (step 5).

**Recommended shutdown procedure:**
```bash
sudo systemctl stop k3s
sudo k3s-killall.sh
```
This cleans up stale network state and prevents startup issues.

### "Failed to get the data key required to decrypt"

Flux can't find the decryption key:

```bash
kubectl get secret sops-age -n flux-system
kubectl describe kustomization apps -n flux-system
```

Solution: Recreate the sops-age secret (see step 4).

### "no key could be found to encrypt"

SOPS can't find your age key:

```bash
echo $SOPS_AGE_KEY_FILE
cat ~/.config/sops/age/keys.txt
```

Solution: Set the `SOPS_AGE_KEY_FILE` environment variable.

## Backups

App configs are backed up daily at 3am to `/mnt/storage/backups/`. Last 7 days are retained.

### Check backup status

```bash
kubectl get cronjobs -A
ls -la /mnt/storage/backups/bazarr/
ls -la /mnt/storage/backups/radarr/
ls -la /mnt/storage/backups/sonarr/
ls -la /mnt/storage/backups/prowlarr/
ls -la /mnt/storage/backups/qbittorrent/
ls -la /mnt/storage/backups/nextcloud/
```

### Manually trigger a backup

```bash
kubectl create job --from=cronjob/bazarr-backup bazarr-backup-manual -n bazarr
kubectl create job --from=cronjob/radarr-backup radarr-backup-manual -n radarr
kubectl create job --from=cronjob/sonarr-backup sonarr-backup-manual -n sonarr
kubectl create job --from=cronjob/prowlarr-backup prowlarr-backup-manual -n prowlarr
```

### Restore from backup

After a fresh cluster setup:

```bash
# 1. Suspend Flux reconciliation
flux suspend kustomization apps

# 2. Scale down the app
kubectl scale deployment sonarr -n sonarr --replicas=0

# 3. Find PVC path
PVC_NAME=$(kubectl get pvc sonarr-config -n sonarr -o jsonpath='{.spec.volumeName}')
sudo ls /var/lib/rancher/k3s/storage/ | grep $PVC_NAME

# 4. Extract backup
sudo tar -xzf /mnt/storage/backups/sonarr/latest.tar.gz -C /var/lib/rancher/k3s/storage/<pvc-directory>/

# 5. Scale up
kubectl scale deployment sonarr -n sonarr --replicas=1

# 6. Resume Flux reconciliation
flux resume kustomization apps
```

## Tools

- [Flux](https://fluxcd.io/) - GitOps operator
- [SOPS](https://github.com/getsops/sops) - Secrets encryption
- [age](https://github.com/FiloSottile/age) - Encryption tool
- [Kustomize](https://kustomize.io/) - Kubernetes configuration management
