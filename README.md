# 🚀 Multicloud Build Action (Composite) — *Open Source* & Stable `@v1`

Build and optionally push Docker images to **AWS ECR** and/or **Azure ACR** from a single action.  
Optimized for **self-hosted AKS/EKS** runners (with a BuildKit sidecar) and works on **GitHub-hosted** runners too.  
Uses **Buildx Bake** for parallel multi-image builds and reads default registries from inputs.

🧩 **Open Source:** Maintained by **GitOps Manager, S.L.**, this action is part of the [**GitOps Manager**](https://gitopsmanager.io) open-source toolchain — powering both community and enterprise automation for multi-cloud Kubernetes environments.

> **Official Release:** This marks the first stable release under the new versioning policy (see below).  
> **Stable:** Pin to `@v1`. Immutable releases (e.g., `v1.0.0`) are also available.


---

## 🌐 GitOps Manager™ Enterprise Platform

[**GitOps Manager Enterprise**](https://gitopsmanager.io) is the full platform that powers this open-source action.  
It’s a **turnkey GitOps automation platform** for AWS and Azure — combining open-source GitHub Actions, Kubernetes infrastructure automation, and global-scale CI/CD.

**Highlights:**
- Secure, opinionated **multi-cloud GitOps automation** for Kubernetes workloads.  
- Deep integration with **ArgoCD**, **Argo Workflows**, **Traefik**, **ECK**, and **Kubernetes Dashboard**.  
- Built for **high availability**, **autoscaling**, and **managed upgrades**.  
- Supports **Workload Identity**, **Pod Identity**, and **private, network-isolated clusters**.  
- Enables **global deployments**, **secret management**, and **production-grade infrastructure** with **zero vendor lock-in**.

🔗 Learn more: [https://gitopsmanager.io](https://gitopsmanager.io)


---

## ✅ Highlights - Multicloud Build Action

- 🐳 **Build once** and **push** to **AWS**, **Azure**, **both**, or **build-only**.  
- 🧺 **Buildx Bake**: parallel multi-image builds from either:
  - a single `{ image, build_file, path }` triple, or
  - a **JSON array** of `{ context, build_file, image_name }`.  
- 🧠 **Smart cache**: BuildKit cache targeted to **the cloud the runner is on** (via `detect-cloud`), with sensible fallback on GitHub-hosted runners.   
- 🧰 **ECR ensure**: auto-creates the ECR repository on first push if missing.  

---




## 🔑 Required permissions

No additional permissions are required for the credential sources that come from
the runner itself (Pod Identity, Workload Identity, MSI, static keys). This
action works with the default `contents: read` that GitHub gives every job.

**GitHub OIDC is the exception.** If you set `aws_ecr_oidc_role_arn` or
`azure_acr_oidc_client_id`, the calling workflow must grant the token — a
composite action cannot grant it for itself:

```yaml
permissions:
  id-token: write
  contents: read
```

Without it the action fails and says so, rather than quietly falling back to a
stored key.

### Where to put it

On the job that runs the action, so the rest of the workflow keeps the
repository default. Declaring `permissions:` makes it **exhaustive** — every
scope you do not list becomes `none` — so a workflow-level block can silently
revoke what another job was relying on:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: gitopsmanager/multicloud-build-action@v1
        with:
          push: both
          image: team/svc-a
          aws_registry:             ${{ vars.AWS_ECR_REGISTRY }}
          aws_ecr_oidc_role_arn:    ${{ vars.AWS_ECR_OIDC_ROLE_ARN }}
          azure_registry:           ${{ vars.AZURE_ACR_REGISTRY }}
          azure_acr_oidc_client_id: ${{ vars.AZURE_ACR_OIDC_CLIENT_ID }}
          azure_acr_oidc_tenant_id: ${{ vars.AZURE_ACR_OIDC_TENANT_ID }}
```

**If this action runs inside a reusable workflow, the grant must come from the
workflow that CALLS it.** Permissions only ever narrow down a chain: a called
workflow cannot give itself a scope the caller withheld, so adding the block to
the inner job alone leaves `id-token` at `none`.

```yaml
# the TOP-level workflow
jobs:
  build:
    uses: my-org/shared-actions/.github/workflows/docker-build.yaml@v1
    permissions:
      id-token: write
      contents: read
```

None of these values are secret — a role ARN and a client id are useless without
the trust relationship — so they belong in organisation **variables**, not
secrets.

---

## 📦 External Actions Used

| Action                               | Pinned Version | Purpose                                                                                 |
|--------------------------------------|----------------|-----------------------------------------------------------------------------------------|
| `actions/checkout`                   | `v4`           | Check out source repo (and optionally a CD repo)                                       |
| `actions/github-script`              | `v7`           | Registry config, ACR REST login, ECR ensure (SigV4), input normalization, bake file gen |
| `aws-actions/amazon-ecr-login`       | `v2`           | Docker login to ECR (Pod Identity / IMDS / static keys)                                |
| `docker/setup-buildx-action`         | `v3`           | Setup Docker Buildx on GitHub-hosted runners                                           |
| `docker/bake-action`                 | `v6`           | Build & push with BuildKit Bake                                                        |
| `gitopsmanager/detect-cloud`         | `v1`           | Detect cloud provider (AWS / Azure / unknown)                                          |

> 🔒 **Pinning advice:**  
> - For most users, pinning to the **major tag** (e.g., `@v3`) is a good balance of stability + updates.  
> - For **strict reproducibility**, pin by **commit SHA** (e.g., `docker/bake-action@<sha>`).

---

## 🧩 Inputs

| Input                   | Required | Default      | Description                                                                                                   |
|-------------------------|:--------:|--------------|---------------------------------------------------------------------------------------------------------------|
| `image_details`         | ❌       |              | **JSON array** of `{context, build_file, image_name}`. If set, takes precedence.                              |
| `image`                 | ❌*      |              | Single image repository/name (e.g., `team/svc-a`). **Required unless** `image_details` is provided.           |
| `build_file`            | ❌       | `Dockerfile` | Dockerfile path (absolute or relative to `$GITHUB_WORKSPACE`).                                                |
| `path`                  | ❌       | `.`          | Build context directory (absolute or relative to `$GITHUB_WORKSPACE`).                                        |
| `tag`                   | ❌       | `""`         | Optional extra tag. Images are **always** tagged with `GITHUB_RUN_ID`; if set, this tag is added as well.     |
| `push`                  | ❌       | `both`       | Where to push: `aws` \| `azure` \| `both` \| `none`.                                                       |
| `buildkit_cache_mode`   | ❌       | `max`        | Cache mode: `none` \| `min` \| `max`. Cache is skipped when `push=none` or `buildkit_cache_mode=none`.      |
| `extra_args`            | ❌       | `""`         | Additional args to pass to BuildKit/Bake (advanced).                                                          |
| `aws_registry`          | ❌       | `""`         | AWS ECR registry URL (e.g., `123456789012.dkr.ecr.eu-west-1.amazonaws.com`). **Required if** `push` includes `aws`. |
| `azure_registry`        | ❌       | `""`         | Azure ACR registry URL (e.g., `myacr.azurecr.io`). **Required if** `push` includes `azure`.                   |
| `aws_ecr_oidc_role_arn` | ❌       | `""`         | IAM role assumed via GitHub OIDC to push to ECR. **Setting it makes OIDC the ECR auth method.** Account and region come from `aws_registry`. |
| `aws_ecr_oidc_audience` | ❌       | `sts.amazonaws.com` | Audience requested for the GitHub token. Must appear in the IAM OIDC provider's client ID list.        |
| `azure_acr_oidc_client_id` | ❌    | `""`         | Client ID of the Entra app registration with a federated credential for this repo. **Setting it makes OIDC the ACR auth method.** Not the same identity as `azure_client_id`. |
| `azure_acr_oidc_tenant_id` | ❌    | `""`         | Tenant of that app registration. Falls back to `azure_tenant_id`.                                        |
| `azure_acr_oidc_audience`  | ❌    | `api://AzureADTokenExchange` | Audience requested for the GitHub token on the Azure exchange.                          |
| `azure_client_id`       | ❌       | `""`         | Azure client ID (fallback when not using WI/MSI).                                                             |
| `azure_client_secret`   | ❌       | `""`         | Azure client secret (fallback).                                                                               |
| `azure_tenant_id`       | ❌       | `""`         | Azure tenant ID (fallback).                                                                                   |
| `aws_access_key_id`     | ❌       | `""`         | AWS access key (fallback when not using Pod Identity / node role).                                            |
| `aws_secret_access_key` | ❌       | `""`         | AWS secret key (fallback).                                                                                    |

\* `image` is required **unless** you set `image_details` (array mode).

---

## 📊 Collection of Usage Metrics

To understand adoption and improve our open-source tooling, **GitOps Manager** collects minimal, anonymized usage data from automated GitHub Actions that use our workflows and integrations.

### What We Collect
Each participating GitHub Action sends a small JSON payload containing:
- **Action Type** – `"build"` or `"deploy"`
- **GitHub Organization** and **Repository** names — sent over HTTPS, then hashed on receipt (HMAC-SHA256 with a secret key). Only the hashes are stored, and only totals are published.
- **Timestamp** of the event

Example of what is stored (after hashing):
```json
{
  "action": "build",
  "org": "11a6114071b1...",
  "repo": "3ad2023afd63...",
  "timestamp": "2025-10-20T19:04:18Z"
}
```

---

## 🔐 Auth (automatic selection)

- **Azure (ACR):** **GitHub OIDC** → AKS **Workload Identity** → node **MSI** (UAMI/SAI via IMDS) → **client secret** fallback.  
- **AWS (ECR):** **GitHub OIDC** → **Pod Identity** (EKS) → **node role** (IMDS) → **static access keys** fallback.

GitHub OIDC is only attempted when you name an identity for that cloud, and then
it is the **only** method tried: naming a role or client id says which identity
to use, so falling back to a different one would defeat the point of configuring
it. Set neither and the chains behave exactly as they always have.

Because the token comes from GitHub rather than from the cloud the runner sits
in, the same identity works **from anywhere** — an EKS pod pushing to ACR, an AKS
pod pushing to ECR, or a GitHub-hosted runner with no cloud identity at all. It
is also the only option here that stores no key and no secret.

> ⚠️ **Token Expiry Notice**  
> Credentials obtained from Pod Identity, Node MSI/IMDS, or other metadata sources typically **expire after about one hour**.  
> If your builds may run longer, break them into smaller Bake targets or sequential jobs to avoid mid-build authentication failures.

> On **GitHub-hosted** runners, use GitHub OIDC — it needs no stored credential.
> Azure client secret and AWS access keys remain supported for installations that
> have not set up federated identity yet.

---

## 🛠 What the action does

1. **Detects cloud** (`azure` / `aws` / `unknown`) using `gitopsmanager/detect-cloud@v1`.  
2. **Normalizes inputs** (single image or array) → generates `docker-bake.json`.  
3. **Provider-aware cache**: chooses exactly **one** registry for cache (`cache-to` / `cache-from`).  
4. **Logs into registries**:
   - **ACR:** REST flow (no `az`) with WI/MSI/secret.
   - **ECR:** Ensures repo exists (SigV4) and logs in (identity or keys).
5. **Runs Buildx Bake** with `source: .` and pushes if requested.

---

## 🔢 Versioning Policy — Official Release

Starting with this release, all future versions follow the new **stable semantic tagging policy**.

Previously, tags like `v1`, `v1.4`, and `v1.4.7` might have moved unpredictably.  
From now on, they will always follow these simple, reliable rules:

| Tag | Moves When | Purpose |
|------|-------------|----------|
| **`v1`** | Any new release in the `v1.x.x` series | Always points to the latest stable release in the major version line. |
| **`v1.4`** | A new patch in the same minor version (e.g. `v1.4.7 → v1.4.8`) | Stays within that feature line. Receives only bug fixes and optimizations — no breaking changes. |
| **`v1.4.7`** | Never | Fully immutable and reproducible. |

### How to choose
- **`@v1`** → Always up to date with the newest non-breaking changes.  
- **`@v1.4`** → Stable feature line with fixes only.  
- **`@v1.4.7`** → Frozen snapshot for exact reproducibility.

All tags will now **increment forward permanently** — no re-use or re-tagging of old versions.  
This marks the **official release** of the action under the **GitOps Manager** stable versioning model.



## 🧾 License

MIT © 2025 GitOps Manager, S.L. — see `LICENSE`.
