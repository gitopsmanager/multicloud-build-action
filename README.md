# 🚀 Multicloud Build Action (Composite) — *Beta* `@main`

Build and optionally push Docker images to **AWS ECR** and/or **Azure ACR** from a single action.  
Designed for **self-hosted AKS/EKS** or **GitHub-hosted** runners. Uses **AKS Workload Identity** / **Azure MSI** or **EKS Pod Identity** / static keys, loads default registries from your **CD repo**, supports **BuildKit caching**, and **auto-creates ECR repositories** when needed.

> **Beta:** Pin to `@main` while you test. Stable tags (`v1`) will follow after broader validation.

---

## ✅ Highlights

- 🐳 **Build** with Docker/BuildKit and **push** to **AWS**, **Azure**, **both**, or **build-only**
- 🔐 **Auth**:
  - **Azure:** AKS **Workload Identity** → Node **MSI** → **Client secret** (fallback)
  - **AWS:** **EKS Pod Identity** / node role → **Access keys** (fallback)
- 🗺️ **CD repo config:** reads default registries from `cd/config/env-map.yaml`
- 🧠 **Smart cache:** BuildKit cache (min/max/none), **cached to the cloud you’re on**
- 🧰 **ECR repo ensure:** creates the ECR repo on first push if missing

---

## ⚙️ What this action assumes

- The **caller job has already checked out** the source repo/ref (this action does not `checkout` your code).
- You provide a **CD repo** that contains:
  ```yaml
  # config/env-map.yaml
  build:
    aws_default_container_registry: 123456789012.dkr.ecr.eu-west-1.amazonaws.com
    azure_default_container_registry: myacr.azurecr.io


## 🧩 Inputs

| Input                  | Required | Default     | Description                                                                 |
|------------------------|:--------:|-------------|-----------------------------------------------------------------------------|
| `path`                 | ❌       | `.`         | Docker build context path.                                                  |
| `image`                | ✅       |             | Image name (e.g., `myteam/myapp`).                                          |
| `tag`                  | ❌       | `""`        | Optional tag; action also tags with `${{ github.run_id }}`.                 |
| `build_file`           | ❌       | `Dockerfile`| Dockerfile path relative to `path`.                                         |
| `extra_args`           | ❌       | `""`        | Extra build args (newline-separated).                                       |
| `push`                 | ❌       | `none`      | Where to push: `aws` \| `azure` \| `both` \| `none`.                        |
| `buildkit_cache_mode`  | ❌       | `max`       | BuildKit cache: `min` \| `max` \| `none`. Cached to current cloud only.     |
| `cd_repo`              | ✅       |             | CD repo `owner/name` containing `cd/config/env-map.yaml`.                   |
| `cd_app_id`            | ✅       |             | GitHub App ID (to read the CD repo).                                        |
| `cd_app_private_key`   | ✅       |             | GitHub App private key (PEM) for the CD repo.                               |
| `azure_client_id`      | ❌       | `""`        | Azure UAMI/App client ID (needed for WI/MSI or secret fallback).            |
| `azure_client_secret`  | ❌       | `""`        | Azure client secret (fallback when not on AKS/MSI).                         |
| `azure_tenant_id`      | ❌       | `""`        | Azure tenant ID (required for WI/secret).                                   |
| `aws_access_key_id`    | ❌       | `""`        | AWS access key (fallback when not on EKS/node role).                        |
| `aws_secret_access_key`| ❌       | `""`        | AWS secret key (fallback).                                                  |

**Auth selection is automatic:**
- **Azure:** AKS WI (token file) → MSI → client secret.  
- **AWS:** Pod/instance identity → access keys.

---

## 🔐 What credentials are actually required?

- **On AKS (self-hosted):** set `AZURE_CLIENT_ID` + `AZURE_TENANT_ID` inputs; the action uses the pod’s **Workload Identity** token file. No client secret needed.
- **On Azure VM runner:** **MSI** is used if available; optionally set `azure_client_id` to select a **user-assigned** identity.
- **Off Azure (e.g., EKS or GH-hosted) and pushing to ACR:** pass **Azure client secret** (`azure_client_id`, `azure_client_secret`, `azure_tenant_id`).
- **On EKS/EC2 (self-hosted):** **Pod/instance identity** is used automatically for ECR.
- **Off AWS (e.g., AKS or GH-hosted) and pushing to ECR:** pass **AWS keys** (`aws_access_key_id`, `aws_secret_access_key`).

This action **does not require GitHub OIDC (`id-token: write`)**, since it uses AKS WI/MSI or Azure client secret, and EKS Pod Identity or AWS keys.

---

## 🛠 Tools used / required

**Included by action:** `yq` (installed if missing), **AWS CLI** (installed if missing).

**Docker & Buildx** — Works on both:
- GitHub-hosted runners: Buildx is set up via `docker/setup-buildx-action`.
- Self-hosted runners: the action connects to a sidecar BuildKit daemon at `tcp://localhost:12345` (you provide the sidecar).

- **`az: command not found`**
  - On **Ubuntu runners** (including `ubuntu-latest`): the action **auto-installs Azure CLI** when `push` is `azure` or `both` via the built-in “Ensure Azure CLI (Ubuntu)” step.
  - On **non-Ubuntu or custom self-hosted runners**: pre-install Azure CLI or add your own install step (the auto-install only runs when `apt-get` is available).


## 🛠 External actions used internally

| Action | Version | Purpose |
|--------|---------|---------|
| [tibdex/github-app-token](https://github.com/tibdex/github-app-token) | `v2.1.0` | Generate a GitHub App installation token to access the CD repo |
| [actions/checkout](https://github.com/actions/checkout) | `v4` | Check out the **CD repo** (the caller checks out the source repo) |
| [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action) | `v3` | Set up Docker Buildx (used on GitHub-hosted runners) |
| [docker/build-push-action](https://github.com/docker/build-push-action) | `v5` | Build and optionally push images via BuildKit |
| [gitopsmanager/detect-cloud](https://github.com/gitopsmanager/detect-cloud) | `main` | Detect cloud provider (`azure` / `aws` / `unknown`) |


---

## 🧪 Usage (Beta `@main`)

### Build and push to both clouds using CD repo registries
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4  # caller checks out source repo

      - uses: your-org/multicloud-build-action@main
        with:
          path: .
          image: myteam/myapp
          tag: ${{ github.sha }}
          push: both
          buildkit_cache_mode: max
          cd_repo: your-org/your-cd-repo
          cd_app_id: ${{ secrets.CD_APP_ID }}
          cd_app_private_key: ${{ secrets.CD_APP_PRIVATE_KEY }}
          # Azure creds only needed when NOT on AKS/MSI:
          azure_client_id: ${{ secrets.AZURE_CLIENT_ID }}
          azure_client_secret: ${{ secrets.AZURE_CLIENT_SECRET }}
          azure_tenant_id: ${{ secrets.AZURE_TENANT_ID }}
          # AWS creds only needed when NOT on EKS/EC2:
          aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

Build only (no push)
- uses: your-org/multicloud-build-action@main
  with:
    path: .
    image: myteam/myapp
    push: none

🧠 How it works (flow)

- Detects cloud (azure/aws/unknown) with detect-cloud@main — used for cache targeting.
- CD repo checkout via GitHub App → reads default registries from cd/config/env-map.yaml.
- Azure auth: tries AKS WI (token file) → MSI → client secret.
- AWS auth: uses Pod/instance identity or keys, then logs in to ECR.
- ECR ensure: creates the ECR repo if missing (idempotent).
- Tagging: always tags with ${{ github.run_id }}; adds your tag if provided.
- BuildKit cache: min/max/none, cached to current cloud only.
- Build & push: via docker/build-push-action@v5.

🧰 Troubleshooting

- az: command not found — Install Azure CLI on your runner (see snippet above).
- No Azure auth method available — Provide azure_client_id + azure_tenant_id (for WI/MSI), or include azure_client_secret for fallback.
- AccessDeniedException pushing to ECR — Ensure your Pod Identity/role or keys include ECR permissions (GetAuthorizationToken, PutImage, UploadLayer*, etc.).
- GitHub-hosted runners — Cloud detection returns unknown; use Azure secret or AWS keys if pushing to that cloud.

🧪 Status & Versioning

- Beta: use @main while testing in your environments.
- Planned tags:
  - v1.0.0 — first stable
  - v1 — floating tag for latest v1
- Breaking changes may occur before v1.0.0.

📄 License

MIT © 2025 Affinity7 Consulting Ltd — see LICENSE.
