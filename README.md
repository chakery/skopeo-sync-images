# Docker Image Sync to Multiple Registries (GitHub Actions)

Syncs images listed in `images.txt` from Docker Hub to one or more container registries using `skopeo`.

**You never edit the workflow to add a registry.** Registries are configured entirely through GitHub's **Variables** and **Environments**.

## How it works

1. `check` reads the `SYNC_TARGETS` repository variable and validates it.
2. `sync` runs once **per registry** — each matrix entry declares its own GitHub Environment, so each registry gets its own isolated credentials.

Workflow file: `.github/workflows/skopeo-sync-images.yml`

---

## Configuration: two pieces per registry

### A. `SYNC_TARGETS` — a repository **Variable**

**Settings → Secrets and variables → Actions → Variables → New repository variable**, name `SYNC_TARGETS`.

Its value is a JSON array. This is **not** sensitive, so you can see and edit it freely:

```json
[
  { "env": "acr",    "host": "registry.cn-hangzhou.aliyuncs.com", "namespace": "docker_hub_mirrors" },
  { "env": "acr_kr", "host": "crpi-xxxx.cn-guangzhou.personal.cr.aliyuncs.com", "namespace": "kr-images" }
]
```

| Field | Required | Notes |
|---|---|---|
| `env` | yes | Name of a GitHub Environment (see below). Shows up as the job name. |
| `host` | yes | Registry host only — no scheme, no trailing path. |
| `namespace` | no | Destination namespace. Optional on personal editions, required on enterprise ones. |

### B. One GitHub **Environment** per registry

**Settings → Environments → New environment**, named to match `env` above (e.g. `acr`).

In each environment, add **two secrets** with exactly these names:

| Secret name | Value |
|---|---|
| `USER` | Login username (Alibaba Cloud account ID) |
| `PASS` | Access credential password |

Because the names are fixed, the workflow never has to change.

> Use the fixed "访问凭证" password from the ACR console. The main account login password will not work.

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

---

## 2) Add a new registry

1. Create an Environment, e.g. `tcr_sh`
2. Add `USER` and `PASS` secrets to it
3. Add one entry to the `SYNC_TARGETS` variable
4. Push

No workflow edits, so GitHub never flags the file as needing approval.

### Tencent Cloud TCR

- `host` is `ccr.ccs.tencentyun.com` — **not** the instance-specific address shown in the console.
- `namespace` is the 12-digit instance ID.

```json
{ "env": "tcr_sh", "host": "ccr.ccs.tencentyun.com", "namespace": "100012345678" }
```

---

## 3) Concurrency model

- **Between registries**: `strategy.matrix` — every registry is an independent job.
  - `fail-fast: false` — one registry failing does not cancel the others.
  - `max-parallel: 4` — caps simultaneous jobs.
  - **Re-run failed jobs** retries only the registries that failed.
- **Within a registry**: images are pushed sequentially, for readable logs.

Environments also unlock per-registry protection rules (required reviewers, wait timers) if you ever want a registry to pause before pushing.

---

## 4) Trigger

- Manual: **Actions → Sync Docker Images To Registries → Run workflow**
- Automatic on changes to `images.txt` or the workflow file

---

## Troubleshooting

- **`Repository variable SYNC_TARGETS is not set`**
    - The variable is missing or has no value.
- **`SYNC_TARGETS must be a non-empty JSON array`**
    - Malformed JSON. Check for trailing commas or single quotes. Validate locally with `jq . <<< "$JSON"`.
- **`Invalid entries in SYNC_TARGETS: item 0 (acr): missing 'host'`**
    - A field is missing. The message names the offending item.
- **This environment's USER / PASS secrets are missing or empty**
    - The Environment named by `env` either does not exist or lacks those two secrets.
- **Login failed**
    - For ACR, confirm the `host` region matches the image address, and that you used the access credential password rather than the account password.
- **`requested access to the resource is denied` while uploading blobs**
    - The repository path is wrong. ACR only accepts `<namespace>/<repository>`, two levels. A path like `<namespace>/docker.io/library/maven` means `--scoped` was used with `skopeo sync`, which carries the `docker.io/library/` prefix over from the source. The workflow uses `skopeo copy` with an explicit destination instead, so this should not occur.
    - If it still happens, check that `namespace` is an existing namespace in that ACR instance and that the account can push to it.
- **Image landed under the wrong path**
    - Destination is built as `<host>/<namespace>/<image as written in images.txt>`. `apache/dubbo-admin:0.6.0` becomes `<host>/<namespace>/apache/dubbo-admin:0.6.0`, and `maven:3.10.0` becomes `<host>/<namespace>/maven:3.10.0` — note the `library/` prefix that Docker Hub uses internally is *not* added.
    - If your registry cannot nest repositories several levels deep, use single-segment image names only.

---

## Security

- Passwords are passed to `skopeo login` via `--password-stdin`, so they never appear in the process list.
- Secrets referenced literally are auto-masked by GitHub in logs; the workflow calls `::add-mask::` on them as well, as a second layer.
- `set -x` is deliberately **not** enabled — shell xtrace would print expanded credentials.
- The workflow uses only `actions/checkout` and built-in shell. No third-party actions.

### Why this design

GitHub Actions cannot index `secrets` by a dynamic name — `${{ secrets[matrix.key] }}` silently yields nothing. Every alternative has a worse failure mode:

| Approach | Problem |
|---|---|
| Put secrets in `strategy.matrix` | GitHub interpolates them into job metadata **before** log masking applies, leaking them into the run's job listing. |
| Serialize the whole `secrets` context | GitHub blocks the workflow as potentially malicious and requires manual approval on every change. |
| One JSON secret holding all credentials | Editing a JSON blob through the secret editor every time you add a registry — easy to corrupt, painful to review. |
| **Environment + fixed secret names (this design)** | Requires creating an Environment per registry, but keeps credentials isolated and the workflow untouched. |

### ACR and OCI attestations

Alibaba Cloud ACR has incomplete support for OCI 1.1 empty-blob attestation manifests. If a multi-arch image built with Buildx (carrying provenance/SBOM) fails with `unsupported media type`, that is the cause — the manifests need their attestation entries stripped. Official Docker Hub images rarely carry attestations, so this is uncommon.
