# 🚀 Multicloud Build Action (Composite) — *Beta* `@main`

Build and optionally push Docker images to **AWS ECR** and/or **Azure ACR** from a single action.  
Optimized for **self-hosted AKS/EKS** runners (with a BuildKit sidecar) and works on **GitHub-hosted** runners too.  
Now uses **Buildx Bake** for parallel multi-image builds and reads default registries from a **JSON** env map.

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


- **Env map is now JSON, not YAML.**  
  The action reads `cd/config/env-map.json`:
  ```json
  {
    "build": {
      "aws_default_container_registry": "123456789012.dkr.ecr.eu-west-1.amazonaws.com",
      "azure_default_container_registry": "myacr.azurecr.io"
    }
  }


## Bake by default

Internally the action **normalizes your inputs** → **generates** a `docker-bake.json` → **runs** `docker buildx bake` with `source: .` so files created earlier in the job are included.

---

## Inputs can be simple or advanced

Provide a single `{ image, build_file, path }` **or** pass a **JSON array** of `{ context, build_file, image_name }`.

---

## 🧩 Inputs

| Input                 | Required | Default      | Description                                                                 |
|----------------------|:--------:|--------------|-----------------------------------------------------------------------------|
| `image_details`      | ❌       |              | **JSON array** of `{context, build_file, image_name}`. If set, takes precedence. |
| `image`              | ❌       |              | **Single image** repository/name (e.g., `team/svc-a`). Used when `image_details` is not provided. |
| `build_file`         | ❌       | `Dockerfile` | Dockerfile path **relative to** `path` (single-image mode).                |
| `path`               | ❌       | `.`          | Build context dir for single-image mode.                                    |
| `push`               | ❌       | `none`       | Where to push: `aws` \| `azure` \| `both` \| `none`.                        |
| `buildkit_cache_mode`| ❌       | `min`        | Cache mode: `none` \| `min` \| `max`. Cached **only** to the detected cloud’s registry. |

**`image_details` example (two images):**
```json
[
  {"context":"./services/a","build_file":"Dockerfile","image_name":"team/svc-a"},
  {"context":"./services/b","build_file":"Dockerfile","image_name":"team/svc-b"}
]
```
## 🔐 Auth (automatic selection)

- **Azure (ACR):** AKS **Workload Identity** → VM **MSI** → **client secret** fallback.  
- **AWS (ECR):** **Pod/instance identity** (IRSA/role) → **access keys** fallback.

> On **GitHub-hosted** runners, use Azure client secret or AWS keys when pushing to that cloud.

---

## 🛠 What the action does (under the hood)

1. **Detects cloud** (`azure` / `aws` / `unknown`) using `gitopsmanager/detect-cloud@main`.
2. **Loads registries** from `cd/config/env-map.json` (JSON) in your repo:
   ```text
   build.aws_default_container_registry
   build.azure_default_container_registry


## 🔁 Normalizes inputs

- If **`image_details`** is provided, parses it as a **JSON array**.
- Otherwise builds a **single-item array** from `{ image, build_file, path }`.

---

## ☁️ Provider-aware caching

- Chooses **one** registry for cache (**AWS** or **Azure**) based on detected cloud (fallbacks if unknown).
- Adds **`cache-from` / `cache-to`** to each target.

---

## 🧩 Bake plan & execution

- **Generates** `docker-bake.json` (**one target per item**).  
  If `push: both`, each target is tagged for **both registries**; Buildx pushes both after a **single** build graph execution.
- **Runs Bake** with **`source: .`** and pushes if requested.

---

## 🧪 Usage (Beta `@main`)

### A) Single image (simple)
```yaml
jobs:
  build:
    runs-on: ubuntu-latest # or self-hosted
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/multicloud-build-action@main
        with:
          image: team/svc-a
          build_file: Dockerfile
          path: ./services/a
          push: both                 # aws | azure | both | none
          buildkit_cache_mode: max   # none | min | max

```

## B) Multiple images (JSON array)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/multicloud-build-action@main
        with:
          image_details: |
            [
              {"context":"./services/a","build_file":"Dockerfile","image_name":"team/svc-a"},
              {"context":"./services/b","build_file":"Dockerfile","image_name":"team/svc-b"}
            ]
          push: both
          buildkit_cache_mode: min
```
> **Note:** Place your registries in `cd/config/env-map.json` (**JSON**). The action reads this file directly; no YAML and no `yq` required.

---

## 🧱 BuildKit & Bake notes

- **Bake parallelism:** targets are scheduled **in parallel** by BuildKit. With a **single sidecar** BuildKit, all parallelism happens **inside that pod** (size CPU/RAM/disk accordingly).
- **`source: .`** ensures Bake sees files you created earlier in the job (not a Git snapshot).
- **Cache to one cloud:** to reduce upload/egress, cache layers to a **single** registry—the detected cloud. Images can still be pushed to **both** registries by giving **two tags** per target.

---

## 🔧 Self-hosted tips (AKS/EKS)

- Run a **BuildKit sidecar** (`buildkitd`) next to the runner; mount a **fast SSD PVC** at `/var/lib/buildkit`; set resource requests/limits.
- Consider **generic ephemeral PVCs** for per-pod cache disks if runners are ephemeral.
- Keep a **registry cache** in Bake (`cache-to`/`cache-from`) so fresh pods/runners warm quickly.
- If one sidecar is a bottleneck, scale out to a **multi-node builder** and append multiple BuildKit endpoints to your Buildx builder.

---

## 🧰 External actions used internally

| Action                         | Version | Purpose                                         |
|--------------------------------|---------|-------------------------------------------------|
| `gitopsmanager/detect-cloud`   | `main`  | Detect cloud provider (`azure` / `aws` / `unknown`) |
| `docker/setup-buildx-action`   | `v3`    | Set up Docker Buildx (on GH-hosted runners)     |
| `docker/bake-action`           | `v6`    | Execute the Bake plan (`docker-bake.json`)      |

> If you push to ACR/ECR from **GH-hosted** runners, also use the appropriate **login** steps before invoking this action.

---

## 🧯 Troubleshooting

- **Files changed earlier aren’t in the build:** Ensure Bake runs with **`source: .`** (this action does).
- **`unknown` provider on GH-hosted:** Expected; caching falls back based on `push`.
- **OOMKilled / disk errors:** Increase sidecar **memory/CPU**, **PVC** size, or split targets into **two Bake groups** to reduce concurrency.
- **ECR push denied:** Verify Pod/instance role or AWS keys allow `ecr:*` actions for authentication and push.

---

## 🧾 License

MIT © 2025 Affinity7 Consulting Ltd — see `LICENSE`.
