# Docker Image Sync to Multiple Registries (GitHub Actions)

Syncs images listed in `images.txt` from Docker Hub to one or more container registries using `skopeo`.

- Registry list lives in **`targets.yml`** (version controlled, safe to publish)
- Credentials live in **GitHub Secrets**, two per registry
- Each registry syncs in its **own parallel job** — no cross-account coupling

## How it works

1. `prepare` job reads `targets.yml`, validates it, checks every required Secret exists, and builds a job matrix.
2. `sync` job runs once **per registry**, in parallel (max 4 at a time), pushing every image from `images.txt`.

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

## 2) Configure `targets.yml`

One entry per target registry. This file contains no credentials and can be public.

```yaml
targets:
  - id: ACR_HZ                              # required, ^[A-Z0-9_]+$
    host: registry.cn-hangzhou.aliyuncs.com # required
    namespace: team-a                       # optional

  - id: TCR_SH
    host: ccr.ccs.tencentyun.com
    namespace: "100012345678"
```

| Field | Required | Notes |
|---|---|---|
| `id` | yes | Uppercase letters, digits, underscores. Used directly as the Secret name prefix. |
| `host` | yes | Registry host only — no scheme, no trailing path. |
| `namespace` | no | Destination namespace. Quote numeric IDs (`"100012345678"`). |

### Tencent Cloud TCR

- `host` is `ccr.ccs.tencentyun.com` — **not** the instance-specific address shown in the console.
- `namespace` is the 12-digit instance ID.

---

## 3) Create the Secrets

For every entry in `targets.yml`, create **two** secrets:

| Secret name | Value |
|---|---|
| `SYNC_<id>_USER` | Login username (Alibaba Cloud account ID / Tencent Cloud account ID) |
| `SYNC_<id>_PASS` | Access credential password |

For the example above:

```text
SYNC_ACR_HZ_USER   → your Alibaba Cloud account ID
SYNC_ACR_HZ_PASS   → your ACR access credential password
SYNC_TCR_SH_USER   → your Tencent Cloud account ID
SYNC_TCR_SH_PASS   → your TCR access credential password
```

Go to **Settings → Secrets and variables → Actions → New repository secret**.

> Use the fixed "访问凭证" password from the ACR console. The main account login password will not work.

---

## 4) Add a new registry

1. Add 3 lines to `targets.yml`
2. Create `SYNC_<id>_USER` and `SYNC_<id>_PASS`
3. Push — the workflow triggers automatically on changes to `targets.yml`

You never edit the workflow file to add a registry.

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
- Automatic on changes to `images.txt`, `targets.yml`, or the workflow file

---

## Troubleshooting

- **`targets.yml must define a non-empty 'targets' array`**
    - The `targets:` key is missing, misspelled, or empty.
- **`Missing GitHub Secret(s): SYNC_XXX_PASS`**
    - The `prepare` job caught it before any login attempt. Create the listed secrets.
- **`Secret ... is missing or empty`**
    - Secret exists but has an empty value.
- **`Every target needs an 'id' matching ^[A-Z0-9_]+$`**
    - The `id` is used to build the Secret name, so lowercase letters and dashes break it.
- **Login failed**
    - For ACR, confirm the `host` region matches the image address, and that you used the access credential password rather than the account password.
- **Image landed under the wrong path**
    - Multi-segment names keep their full path (`apache/dubbo-admin:0.6.0` → `<host>/<namespace>/apache/dubbo-admin:0.6.0`) because the workflow passes `--scoped`. Images synced before this was enabled live at `<namespace>/dubbo-admin` and will need re-syncing.

---

## Security

- `targets.yml` may be public; credentials must never be committed.
- The username is a Secret too, because a public repo that spells out usernames leaks who owns which account.
- Passwords are passed to `skopeo login` via `--password-stdin`, so they never appear in the process list.
- The workflow calls `::add-mask::` on credentials before use, so they stay out of logs. GitHub only auto-masks values it knows verbatim — values unpacked from a JSON blob are **not** masked automatically, which is why credentials are stored one-per-secret instead of bundled in a JSON array.
- The workflow uses only `actions/checkout` and built-in shell. No third-party actions.

### Known limitation

`SYNC_<id>_USER` / `SYNC_<id>_PASS` are read via `toJSON(secrets)` because GitHub Actions cannot index `secrets` by a dynamic name. This serializes every repository secret into the job environment, which security scanners (CodeQL, Datadog) flag as CWE-312. It is accepted here because the workflow runs no third-party actions and the repository is public — but be aware of it if you add external actions.

### ACR and OCI attestations

Alibaba Cloud ACR has incomplete support for OCI 1.1 empty-blob attestation manifests. If a multi-arch image built with Buildx (carrying provenance/SBOM) fails with `unsupported media type`, that is the cause — the manifests need their attestation entries stripped. Official Docker Hub images rarely carry attestations, so this is uncommon.

---

## Migrating from `ALIYUN_REGISTRIES`

The previous version stored all registries as one JSON blob in a single secret. To migrate:

1. **Back up the current `ALIYUN_REGISTRIES` value** to a password manager — GitHub cannot show it again after deletion.
2. Commit `targets.yml` and the new workflow.
3. Create `SYNC_<id>_USER` / `SYNC_<id>_PASS` for each registry.
4. Run the workflow manually and confirm it succeeds.
5. Delete the old `ALIYUN_REGISTRIES` secret.

Credentials must be rotated if the old runs leaked them into logs — the JSON blob format was not auto-masked.
