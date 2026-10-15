# Docker Image Sync to Multiple Registries (GitHub Actions)

Syncs images listed in `images.txt` from Docker Hub to one or more container registries using `skopeo`.

- Each registry syncs in its **own parallel job** — no cross-account coupling
- Each registry has its **own pair of secrets** — no shared JSON blob to edit
- One registry failing never blocks the others, and only the failures re-run

## How it works

The `sync` job runs once **per registry** via `strategy.matrix`, pushing every image from `images.txt`.

Workflow file: `.github/workflows/skopeo-sync-images.yml`

---

## 1) Prepare `images.txt`

One image per line, in the repository root.

```txt
#nginx:1.25.6-alpine
apache/dubbo-admin:0.6.0
```

Notes:

- Lines starting with `#` and blank lines are ignored.
- Do **not** write the `docker.io/` prefix. `nginx:1.25-alpine` and `apache/dubbo-admin:0.6.0` are both valid.
- Always pin an explicit tag. Omitting it makes skopeo sync **every tag** in the repo.

## 2) Configure the registries

The registry list is the `matrix.include` block in `.github/workflows/skopeo-sync-images.yml`:

```yaml
matrix:
  include:
    - id: ACR
      host: registry.cn-hangzhou.aliyuncs.com
      namespace: docker_hub_mirrors
      user: ${{ secrets.SYNC_ACR_USER }}
      pass: ${{ secrets.SYNC_ACR_PASS }}
```

| Field | Required | Notes |
|---|---|---|
| `id` | yes | Any short label. Used in job names and log output. |
| `host` | yes | Registry host only — no scheme, no trailing path. |
| `namespace` | no | Destination namespace. Optional on personal editions, required on enterprise ones. |
| `user` / `pass` | yes | Secret references — see below. |

The workflow file contains no credentials, so it is safe to publish.

## 3) Create the secrets

For each matrix entry, create **two** secrets under **Settings → Secrets and variables → Actions**:

| Secret name | Value |
|---|---|
| `SYNC_<id>_USER` | Login username (Alibaba Cloud account ID) |
| `SYNC_<id>_PASS` | Access credential password |

For the example above: `SYNC_ACR_USER` and `SYNC_ACR_PASS`.

> Use the fixed "访问凭证" password from the ACR console. The main account login password will not work.

---

## 4) Add a new registry

1. Add a block to `matrix.include` in the workflow
2. Create `SYNC_<id>_USER` and `SYNC_<id>_PASS`
3. Push

No JSON to edit — just one YAML block and two secrets.

### Tencent Cloud TCR

- `host` is `ccr.ccs.tencentyun.com` — **not** the instance-specific address shown in the console.
- `namespace` is the 12-digit instance ID.

---

## 5) Concurrency model

- **Between registries**: GitHub Actions `strategy.matrix` — every registry is an independent job.
  - `fail-fast: false` — one registry failing does not cancel the others.
  - `max-parallel: 4` — caps simultaneous jobs.
  - **Re-run failed jobs** retries only the registries that failed.
- **Within a registry**: images are pushed sequentially, for readable logs.

---

## 6) Trigger

- Manual: **Actions → Sync Docker Images To Registries → Run workflow**
- Automatic on changes to `images.txt` or the workflow file

---

## Troubleshooting

- **`Secret SYNC_ACR_USER / SYNC_ACR_PASS is missing or empty`**
    - That pair of secrets does not exist or has an empty value. Check the `id` in the matrix matches the secret names exactly.
- **`No active images found in images.txt`**
    - `images.txt` has only comments and blank lines.
- **Login failed**
    - For ACR, confirm the `host` region matches the image address, and that you used the access credential password rather than the account password.
- **Image landed under the wrong path**
    - Multi-segment names keep their full path (`apache/dubbo-admin:0.6.0` → `<host>/<namespace>/apache/dubbo-admin:0.6.0`) because the workflow passes `--scoped`. Images synced before this was enabled live at `<namespace>/dubbo-admin` and will need re-syncing.

---

## Security

- Credentials must never be committed. The workflow file contains only secret *references*.
- Passwords are passed to `skopeo login` via `--password-stdin`, so they never appear in the process list.
- Secrets referenced literally are auto-masked by GitHub in logs; the workflow calls `::add-mask::` on them as well, as a second layer.
- `set -x` is deliberately **not** enabled — shell xtrace would print expanded credentials.
- The workflow uses only `actions/checkout` and built-in shell. No third-party actions.

### Why literal secret references?

GitHub Actions cannot index `secrets` by a dynamic name — `${{ secrets[matrix.key] }}` silently yields nothing. The alternatives are both worse:

- **Serializing the whole `secrets` context** puts every repository secret into the job environment. GitHub treats that as potentially malicious, blocks the workflow, and requires a maintainer to approve each time the file changes.
- **A single JSON secret** holding all credentials means editing one blob every time you add a registry — easy to corrupt, and past failures suggest it is easy to get wrong.

One secret pair per registry, referenced literally, is the approach that keeps each registry independent and avoids both problems.

### ACR and OCI attestations

Alibaba Cloud ACR has incomplete support for OCI 1.1 empty-blob attestation manifests. If a multi-arch image built with Buildx (carrying provenance/SBOM) fails with `unsupported media type`, that is the cause — the manifests need their attestation entries stripped. Official Docker Hub images rarely carry attestations, so this is uncommon.
