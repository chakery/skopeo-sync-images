# Docker Image Sync to Multiple Registries (GitHub Actions)

Syncs images listed in `images.txt` from Docker Hub to one or more container registries using `skopeo`.

- Registry list lives in **`targets.yml`** (version controlled, safe to publish)
- Credentials live in **one** GitHub Secret, `SYNC_CREDS`
- Each registry syncs in its **own parallel job** — no cross-account coupling
- **The workflow file never needs to change when you add a registry**

## How it works

1. `prepare` reads `targets.yml`, validates it, checks every id has credentials, builds a job matrix.
2. `sync` runs once **per registry**, in parallel (max 4 at a time), pushing every image from `images.txt`.

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

One entry per target registry. Contains no credentials, so it can be public.

```yaml
targets:
  - id: ACR                              # required, ^[A-Z0-9_]+$
    host: registry.cn-hangzhou.aliyuncs.com # required
    namespace: docker_hub_mirrors        # optional

  - id: ACR_KR
    host: crpi-xxxx.cn-guangzhou.personal.cr.aliyuncs.com
    namespace: kr-images
```

| Field | Required | Notes |
|---|---|---|
| `id` | yes | Uppercase letters, digits, underscores. Keys into `SYNC_CREDS`. |
| `host` | yes | Registry host only — no scheme, no trailing path. |
| `namespace` | no | Destination namespace. Quote numeric IDs (`"100012345678"`). |

### Tencent Cloud TCR

- `host` is `ccr.ccs.tencentyun.com` — **not** the instance-specific address shown in the console.
- `namespace` is the 12-digit instance ID.

---

## 3) Configure the `SYNC_CREDS` secret

Create **one** repository secret named `SYNC_CREDS` (**Settings → Secrets and variables → Actions → New repository secret**).

Its value is a JSON object keyed by the `id` values from `targets.yml`:

```json
{
  "ACR":    { "user": "your-alibaba-account-id", "pass": "your-acr-password" },
  "ACR_KR": { "user": "your-alibaba-account-id", "pass": "your-acr-password" }
}
```

- Use the fixed "访问凭证" password from the ACR console. The main account login password will not work.
- Every id in `targets.yml` must have a matching key with **both** `user` and `pass` non-empty. A missing one fails the run in `prepare`, naming the offending id.

---

## 4) Add a new registry

1. Add an entry to `targets.yml`
2. Add the matching key to the `SYNC_CREDS` secret
3. Push

**The workflow file is never touched**, so it never needs approval from GitHub's workflow security checks.

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
    - The `targets:` key is missing, misspelled, or an empty list.
- **`Secret SYNC_CREDS is empty`**
    - The secret does not exist or has no value.
- **`Secret SYNC_CREDS must be a JSON object`**
    - Malformed JSON. Check for trailing commas, unescaped quotes, or single quotes.
- **`Secret SYNC_CREDS has no credentials for: ACR_KR`**
    - `targets.yml` lists an id that has no matching key in the secret (or its `user`/`pass` is empty).
- **`Every target needs an 'id' matching ^[A-Z0-9_]+$`**
    - The `id` is used as the credential key, so lowercase letters and dashes break it.
- **Login failed**
    - For ACR, confirm the `host` region matches the image address, and that you used the access credential password rather than the account password.
- **Image landed under the wrong path**
    - Multi-segment names keep their full path (`apache/dubbo-admin:0.6.0` → `<host>/<namespace>/apache/dubbo-admin:0.6.0`) because the workflow passes `--scoped`. Images synced before this was enabled live at `<namespace>/dubbo-admin` and will need re-syncing.

---

## Security

- `targets.yml` may be public; credentials must never be committed.
- The username is kept in a Secret too, because a public repo that spells out usernames leaks who owns which account.
- Passwords are passed to `skopeo login` via `--password-stdin`, so they never appear in the process list.
- GitHub only auto-masks a secret **verbatim** — values unpacked from a JSON blob are *not* masked automatically. The workflow therefore calls `::add-mask::` on each credential immediately after extracting it, before anything can print it. Keep `set -x` off (it is not enabled).
- The workflow references exactly one secret, by literal name. It deliberately avoids `toJSON(secrets)`, which GitHub flags as potentially malicious and blocks pending manual approval.

### Why one secret instead of one per registry?

GitHub Actions cannot index `secrets` by a dynamic name — `${{ secrets[matrix.key] }}` silently yields nothing. The only workarounds are:

- referencing each secret **literally**, which means editing the workflow file every time you add a registry, or
- serializing the entire `secrets` context with `toJSON(secrets)`, which GitHub blocks pending approval.

Keeping one `SYNC_CREDS` secret and adding a key to it is the only option that lets you add registries by editing configuration alone.

### ACR and OCI attestations

Alibaba Cloud ACR has incomplete support for OCI 1.1 empty-blob attestation manifests. If a multi-arch image built with Buildx (carrying provenance/SBOM) fails with `unsupported media type`, that is the cause — the manifests need their attestation entries stripped. Official Docker Hub images rarely carry attestations, so this is uncommon.
