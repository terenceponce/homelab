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

## Tools

- [Flux](https://fluxcd.io/) - GitOps operator
- [SOPS](https://github.com/getsops/sops) - Secrets encryption
- [Kustomize](https://kustomize.io/) - Kubernetes configuration management
