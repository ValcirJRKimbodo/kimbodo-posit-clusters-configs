# kimbodo-posit‑clusters‑configs

> **Central repository for declarative Kubernetes deployments of the Posit platform (Workbench, Connect, Package Manager) across GKE, AKS and EKS using Helmfile + SOPS.**



---

## Table of Contents

1. [Project Goals](#project-goals)
2. [Repository Structure](#repository-structure)
3. [Helmfile Overview](#helmfile-overview)
4. [Values & Secrets Management](#values--secrets-management)
5. [Local Usage](#local-usage)
6. [CI/CD Pipeline](#cicd-pipeline)
7. [Custom Tooling Image](#custom-tooling-image)
8. [Contributing](#contributing)
9. [License](#license)

---

## Project Goals

- **Single source‑of‑truth** for all cluster configurations of the Posit stack.
- **Environment isolation** (`mvp-gcp`, `mvp-aks`, `mvp-eks`, …) via [Helmfile](https://github.com/helmfile/helmfile) environments.
- **Zero‑touch secrets** – only encrypted blobs live in Git thanks to [SOPS](https://github.com/getsops/sops) + cloud KMS.
- **Portable automation** – same workflow locally or in the GitHub Actions pipeline.

---

## Repository Structure

```text
.
├── helmfile.yaml               # Root Helmfile (repositories, releases, defaults…)
├── .sops.yaml                  # SOPS rules – which keys/fields are encrypted
├── values-environments/        # Per‑env overrides + secrets
│   ├── mvp-gcp/
│   │   ├── global.yaml         # General Posit tuning (license, domain, …)
│   │   ├── rstudio-connect.yaml
│   │   ├── rstudio-workbench.yaml
│   │   └── postgresql*.yaml
│   └── mvp-aks/ …              # Same layout for other clouds
└── .github/workflows/
    └── deploy.yaml             # GitHub Actions workflow (manual & on‑push)
```

### Naming conventions

| Folder / File                           | Purpose                                            |
| --------------------------------------- | -------------------------------------------------- |
| `values-environments/<env>/global.yaml` | Cluster‑wide values (cloud, region, Posit license) |
| `*-package-manager.yaml`                | Dedicated overrides for Posit Package Manager      |
| `postgresql*.yaml`                      | External DB or side‑car Postgres parameters        |

---

## Helmfile Overview

- **Helm Repositories**: *rstudio*, *jetstack*, *bitnami*, *ingress‑nginx*, plus the private chart repo `kimbodo-posit`.
- **Releases** (excerpt):
  - `cert-manager` – SSL certificates (Let’s Encrypt)
  - `ingress-nginx` – cluster ingress
  - `nfs-server-provisioner` – RWX storage for Posit
  - `rstudio-connect`, `rstudio-workbench`, `rstudio-pm` – Posit platform charts
  - `kimbodo-posit-base-resources` – helper chart (RBAC, PVC, ConfigMaps)
- **Helm Defaults**: `wait: true`, `timeout: 10m`, plus secrets handled via the `helm-secrets` plugin.

---

## Values & Secrets Management

- **Encryption**: only selected keys are encrypted – e.g. `global.license`, `postgres.postgresPassword` – controlled by `.sops.yaml`:

  ```yaml
  creation_rules:
    - path_regex: values-environments/.*/.*\.yaml
      encrypted_regex: 'license|postgresPassword'
      kms: projects/gcp-k8s-demo-posit/locations/global/keyRings/sops-keyring/cryptoKeys/sops-key
  ```

- **Decrypt locally**:

  ```bash
  gcloud auth application-default login   # or aws/az equivalents
  helmfile -e mvp-gcp secrets decrypt     # temporary cleartext
  ```

- **Encrypt a file**:

  ```bash
  sops -e -i values-environments/mvp-gcp/global.yaml
  ```

---

## Local Usage

Requirements:

- Docker (optional but handy)
- Helm ≥ 3.14
- Helmfile ≥ 0.144
- SOPS ≥ 3.8 + cloud KMS access
- kubectl ≥ 1.29 (configured)

```bash
# clone
git clone https://github.com/ValcirJRKimbodo/kimbodo-posit-clusters-configs.git
cd kimbodo-posit-clusters-configs

# choose environment
export ENV=mvp-gcp

# diff first (needs helm‑diff plugin)
helmfile -e "$ENV" diff

# apply
helmfile -e "$ENV" apply
```

> 💡 Run inside the **kimbodovalcir/kimbodo-posit-actions** container to avoid installing dependencies.

---

## CI/CD Pipeline

- **Workflow**: `.github/workflows/deploy.yaml`
- **Container**: `ghcr.io/kimbodovalcir/kimbodo-posit-actions:latest`
- **Triggers**:
  - Manual (`workflow_dispatch`) – select environment
- **Key steps**:
  1. Checkout code
  2. Auth to GCP & fetch cluster credentials
  3. `helmfile -e $ENV apply`

---

## Custom Tooling Image

```bash
docker pull kimbodovalcir/kimbodo-posit-actions:latest
```

Includes:

| Tool         | Version |
| ------------ | ------- |
| Terraform    | 1.8.4   |
| Helm         | 3.14.0  |
| Helmfile     | 0.144.0 |
| helm‑secrets | latest  |
| helm‑diff    | latest  |
| SOPS         | 3.8.1   |
| gcloud SDK   | 470+    |

Run an ephemeral shell:

```bash
docker run -it --rm \
  -e HELM_PLUGINS=/root/.local/share/helm/plugins \
  -v "$HOME/.kube:/root/.kube" \
  -v "$PWD:/work" -w /work \
  kimbodovalcir/kimbodo-posit-actions bash
```

---

## Contributing

Bug fixes and improvements are welcome! Feel free to open an issue or a PR.

1. Fork the repo
2. Create a feature branch (`git checkout -b feat/my-feature`)
3. Commit & push (`git commit -m "feat: …"`)
4. Open a Pull Request – please follow the PR template

---

## License

This project is licensed under the **MIT License**. See [`LICENSE`](LICENSE) for details.

