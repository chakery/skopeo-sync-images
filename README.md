# Docker Image Sync to Alibaba Cloud Registry (GitHub Actions)

This repository includes a GitHub Actions workflow that reads Docker images from `images.txt` and syncs them to one or more Alibaba Cloud registries using `skopeo`.

## What the workflow does

- Reads image entries from `images.txt`
- Ignores:
    - Commented lines (lines starting with `#`)
    - Empty lines
- Logs in to each configured Alibaba Cloud registry
- Syncs every active image to every configured registry using:

```bash
skopeo sync --all --src docker --dest docker <source-image> <destination-image>
```

Workflow file: `.github/workflows/skopeo-sync-images.yml`

## 1) Prepare `images.txt`

Create or update `images.txt` in the repository root.

Example:

```txt
#golang:1.25.6-alpine3.23
apache/dubbo-admin:0.6.0
```

Notes:

- Lines starting with `#` are ignored.
- Use one image per line.
- `docker.io/` prefix is optional for Docker Hub images.
    - `apache/dubbo-admin:0.6.0` is valid.
    - `docker.io/apache/dubbo-admin:0.6.0` is also valid.

## 2) Configure GitHub Secret

Add a repository secret named `ALIYUN_REGISTRIES`.

Expected format: JSON array.

```json
[
  {
    "registry": "registry.cn-hangzhou.aliyuncs.com",
    "namespace": "team-a",
    "username": "your-username-a",
    "password": "your-password-a"
  },
  {
    "registry": "registry.cn-shanghai.aliyuncs.com",
    "namespace": "team-b",
    "username": "your-username-b",
    "password": "your-password-b"
  }
]
```

Field description:

- `registry` (required): Alibaba registry host
- `username` (required): login username
- `password` (required): login password
- `namespace` (optional): destination namespace/path prefix

If `namespace` is empty, images are pushed as:

`<registry>/<image>`

If `namespace` is set, images are pushed as:

`<registry>/<namespace>/<image>`

## 3) Trigger the workflow

The workflow supports:

- Manual trigger (`workflow_dispatch`)
- Auto trigger on changes to:
    - `images.txt`
    - `.github/workflows/skopeo-sync-images.yml`

### Manual run

1. Open your GitHub repository
2. Go to **Actions**
3. Select **Sync Docker Images To Alibaba Registries**
4. Click **Run workflow**

## 4) Troubleshooting

- **`images.txt not found`**
    - Ensure `images.txt` is in repository root.
- **`ALIYUN_REGISTRIES is empty`**
    - Confirm the secret exists and is named exactly `ALIYUN_REGISTRIES`.
- **Invalid JSON**
    - Validate your secret value is a valid JSON array.
- **Login failed**
    - Verify `registry`, `username`, and `password` are correct.

## Security recommendations

- Store credentials in GitHub Secrets only.
- Do not commit registry passwords into source files.
- Prefer dedicated robot/service accounts with minimal permissions.
