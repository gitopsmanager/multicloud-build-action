# 🚀 Multicloud Build Action (Composite) — *Beta* `@main`

Build and optionally push Docker images to **AWS ECR** and/or **Azure ACR** from a single action.  
Optimized for **self-hosted AKS/EKS** runners (with a BuildKit sidecar) and works on **GitHub-hosted** runners too.  
Uses **Buildx Bake** for parallel multi-image builds and reads default registries from a **JSON** env map.

> **Beta:** Pin to `@main` while testing. Stable tags (e.g., `v1`) will follow after broader validation.

---

## ✅ Highlights

- 🐳 **Build once** and **push** to **AWS**, **Azure**, **both**, or **build-only**.  
- 🧺 **Buildx Bake**: parallel multi-image builds from either:
  - a single `{ image, build_file, path }` triple, or
  - a **JSON array** of `{ context, build_file, image_name }`.  
- 🧠 **Smart cache**: BuildKit cache targeted to **the cloud the runner is on** (via `detect-cloud`), with sensible fallback on GitHub-hosted runners.  
- 🗺️ **CD config (JSON)**: reads default registries from `cd/config/env-map.json`.  
- 🧰 **ECR ensure**: auto-creates the ECR repository on first push if missing.  

---

- **Env map is JSON (not YAML).**  
  The action reads `cd/config/env-map.json`:

```json
{
  "build": {
    "aws_default_container_registry": "123456789012.dkr.ecr.eu-west-1.amazonaws.com",
    "azure_default_container_registry": "myacr.azurecr.io"
  }
}
```

---

## 🔑 Required permissions

No additional permissions are required.  
This action works with the default `contents: read` permission that GitHub provides to all jobs.


---

## 🧩 Inputs

| Input                   | Required | Default      | Description                                                                                                   |
|-------------------------|:--------:|--------------|---------------------------------------------------------------------------------------------------------------|
| `image_details`         | ❌       |              | **JSON array** of `{context, build_file, image_name}`. If set, takes precedence.                              |
| `image`                 | ✅*      |              | Single image repository/name (e.g., `team/svc-a`). **Required unless** `image_details` is provided.           |
| `build_file`            | ❌       | `Dockerfile` | Dockerfile path **relative to** `path` (single-image mode).                                                   |
| `path`                  | ❌       | `.`          | Build context dir (single-image mode).                                                                        |
| `tag`                   | ❌       | `""`         | Optional extra tag. Images are **always** tagged with `GITHUB_RUN_ID`; if set, this tag is added as well.     |
| `push`                  | ❌       | `none`       | Where to push: `aws` \| `azure` \| `both` \| `none`.                                                          |
| `buildkit_cache_mode`   | ❌       | `max`        | Cache mode: `none` \| `min` \| `max`. Cache goes to **one** registry (the detected cloud).                    |
| `extra_args`            | ❌       | `""`         | Additional args to pass to BuildKit/Bake (advanced).                                                          |
| `cd_repo`               | ✅       |              | CD repo (`owner/repo`) containing `cd/config/env-map.json` with default registries.                           |
| `cd_app_id`             | ✅       |              | GitHub App ID (for reading the CD repo).                                                                      |
| `cd_app_private_key`    | ✅       |              | GitHub App private key (PEM) for the CD repo.                                                                 |
| `azure_client_id`       | ❌       | `""`         | Azure client ID (fallback when not using WI/MSI).                                                             |
| `azure_client_secret`   | ❌       | `""`         | Azure client secret (fallback).                                                                               |
| `azure_tenant_id`       | ❌       | `""`         | Azure tenant ID (fallback).                                                                                   |
| `aws_access_key_id`     | ❌       | `""`         | AWS access key (fallback when not using Pod Identity / node role).                                            |
| `aws_secret_access_key` | ❌       | `""`         | AWS secret key (fallback).                                                                                    |

\* `image` is required **unless** you set `image_details` (array mode).

---

## 🔐 Auth (automatic selection)

- **Azure (ACR):** AKS **Workload Identity** → node **MSI** (UAMI/SAI via IMDS) → **client secret** fallback.  
- **AWS (ECR):** **Pod Identity** (EKS) → **node role** (IMDS) → **static access keys** fallback.

> On **GitHub-hosted** runners, use Azure client secret or AWS keys when pushing to that cloud.

---

## 🛠 What the action does

1. **Detects cloud** (`azure` / `aws` / `unknown`) using `gitopsmanager/detect-cloud@main`.  
2. **Loads registries** from `cd/config/env-map.json`:
   - `build.aws_default_container_registry`
   - `build.azure_default_container_registry`
3. **Normalizes inputs** (single image or array) → generates `docker-bake.json`.  
4. **Provider-aware cache**: chooses exactly **one** registry for cache (`cache-to` / `cache-from`).  
5. **Logs into registries**:
   - **ACR:** REST flow (no `az`) with WI/MSI/secret.
   - **ECR:** Ensures repo exists (SigV4) and logs in (identity or keys).
6. **Runs Buildx Bake** with `source: .` and pushes if requested.

---

## 🔁 Normalized inputs

**Array mode** example:

```json
[
  {"context":"./services/a","build_file":"Dockerfile","image_name":"team/svc-a"},
  {"context":"./services/b","build_file":"Dockerfile","image_name":"team/svc-b"}
]
```

**Single image** (the action converts it to an array internally):

```yaml
with:
  image: team/svc-a
  build_file: Dockerfile
  path: ./services/a
```

---

## ☁️ Provider-aware caching

- Chooses **one** cache registry (AWS or Azure) based on detected provider.
- On `unknown` (e.g., GH-hosted), falls back based on `push` target.
- Adds `cache-from` and `cache-to` per target; images can still be pushed to **both** clouds.

---
## BuildKit on port 12345 — requirements & runner support

This workflow expects, **on self-hosted runners**, a BuildKit daemon (`buildkitd`) running and reachable at: 12345

---

### Self-hosted runners (required setup)
- **Listener:** start `buildkitd` with `--addr=tcp://0.0.0.0:12345` so the runner container can reach it at `127.0.0.1:12345` (same pod network namespace).
- **Worker:** OCI worker (common today) or containerd worker — both are supported.
---

### Kubernetes privileges for BuildKit (self-hosted)

To support Dockerfile `RUN` steps (e.g., `apt-get`) the BuildKit sidecar may need:
- **Cgroups:** mount `/sys/fs/cgroup` **read-write** into the container.
- **Full privilege escalation:** run the sidecar **privileged** (or at minimum `allowPrivilegeEscalation: true`).
- **Security profiles:** use **Unconfined** for both Seccomp and AppArmor.
- **Microsoft/AKS AppArmor annotation:** set the per-container annotation to unconfined.


**How the workflow connects:**
```bash
docker buildx create --name remote-builder --driver remote tcp://127.0.0.1:12345
docker buildx use remote-builder
docker buildx inspect --bootstrap
```
---
### GitHub-hosted runners

GitHub-hosted runners do **not** expose a `buildkitd` at `127.0.0.1:12345`.

Use the managed builder instead:

```yaml
- uses: docker/setup-buildx-action@v3
```
The “remote BuildKit on :12345” requirement applies only to self-hosted runners.
---

## 🧩 Bake plan & execution

- Generates **one target per item** in `docker-bake.json`.  
- If `push: both`, each target receives **two tags** (one per registry).  
- Bake runs with **`source: .`** so any files created earlier in the job are included.

---

## 🧪 Usage (Beta `@main`)

### A) Single image

```yaml
jobs:
  build:
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest # or self-hosted
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/multicloud-build-action@main
        with:
          cd_repo: your-org/continuous-deployment
          cd_app_id: ${{ secrets.CD_APP_ID }}
          cd_app_private_key: ${{ secrets.CD_APP_PRIVATE_KEY }}
          image: team/svc-a
          build_file: Dockerfile
          path: ./services/a
          push: both                 # aws | azure | both | none
          buildkit_cache_mode: max   # none | min | max
```

### B) Multiple images (JSON array)

```yaml
jobs:
  build:
    permissions:
      contents: read
      id-token: write
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/multicloud-build-action@main
        with:
          cd_repo: your-org/continuous-deployment
          cd_app_id: ${{ secrets.CD_APP_ID }}
          cd_app_private_key: ${{ secrets.CD_APP_PRIVATE_KEY }}
          image_details: |
            [
              {"context":"./services/a","build_file":"Dockerfile","image_name":"team/svc-a"},
              {"context":"./services/b","build_file":"Dockerfile","image_name":"team/svc-b"}
            ]
          push: both
          buildkit_cache_mode: min
```

---

## 🧱 BuildKit & Bake notes

- **Parallelism:** Bake schedules targets in parallel; with a **single** BuildKit sidecar, parallelism occurs **inside that pod**.  
- **Disk/CPU:** Size the BuildKit sidecar (CPU/RAM/ephemeral or PVC). Consider **generic ephemeral PVCs** for ephemeral runners.  
- **Cache:** Keep a **registry cache** (cache-to/from) so fresh pods/runners warm quickly.  
- **Multi-builder:** If one sidecar bottlenecks, consider a multi-node builder and point Buildx at multiple endpoints.

---

## 🧰 External actions used internally

| Action                          | Version | Purpose(s)                                                                 |
|---------------------------------|---------|----------------------------------------------------------------------------|
| `actions/checkout`              | `v4`    | Checkout source repo; checkout CD repo                                     |
| `tibdex/github-app-token`       | `v2.1.0`| Generate GitHub App token for CD repo access                               |
| `actions/github-script`         | `v7`    | Load env config; ACR REST login; ECR ensure; normalize inputs; cache plan; generate `docker-bake.json` |
| `aws-actions/amazon-ecr-login`  | `v2`    | Docker login to ECR (Pod Identity / IMDS / static keys)                    |
| `docker/setup-buildx-action`    | `v3`    | Setup Docker Buildx on GitHub-hosted runners                               |
| `gitopsmanager/detect-cloud`    | `@main` | Detect cloud provider (AWS / Azure / unknown)                              |
| `docker/bake-action`            | `v6`    | Build & push with BuildKit Bake                                            |

> If you push from **GH-hosted** runners, provide appropriate secrets for that cloud (Azure client secret or AWS keys).

---

## 🧯 Troubleshooting

- **`unknown` provider on GH-hosted:** expected; caching falls back from `push`.  
- **ECR push denied:** Ensure the identity has `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:PutImage`, etc., and that the repo exists (auto-create should handle it).  
- **ACR `insufficient_scope`:** Make sure the ACR token request uses `repository:*:pull,push` (the action does).  
- **OOM/Disk issues:** Increase BuildKit sidecar resources or split the build into two Bake groups.  
- **Files missing in build context:** Ensure the step runs Bake with `source: .` (this action does).

---

## 🧾 License

MIT © 2025 Affinity7 Consulting Ltd — see `LICENSE`.
