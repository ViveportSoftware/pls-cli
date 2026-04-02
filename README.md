# pls-cli

**Polygon Streaming CLI** — upload and replace 3D model assets to [VIVERSE](https://www.viverse.com) from the command line.

```
pls-cli upload model.glb --group=<group-uuid> --ai-enhance
```

---

## Installation

### Download a pre-built binary (recommended)

Go to the [Releases](https://github.com/ViveportSoftware/pls-cli/releases/latest) page and download the binary for your platform.

| Platform | Binary |
|---|---|
| macOS (Apple Silicon) | `pls-cli-darwin-arm64` |
| macOS (Intel) | `pls-cli-darwin-amd64` |
| Linux | `pls-cli-linux-amd64` |
| Windows | `pls-cli-windows-amd64.exe` |

**Quick install on macOS / Linux:**

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
VERSION="v1.0.0"

case "$ARCH" in
  x86_64)        ARCH="amd64" ;;
  arm64|aarch64) ARCH="arm64" ;;
esac

curl -fsSL \
  "https://github.com/ViveportSoftware/pls-cli/releases/download/${VERSION}/pls-cli-${OS}-${ARCH}" \
  -o /usr/local/bin/pls-cli

chmod +x /usr/local/bin/pls-cli
pls-cli version
```

**Windows:** download `pls-cli-windows-amd64.exe`, rename it to `pls-cli.exe`, and add it to your `PATH`.

---

## Quick Start

```bash
# 1. Log in (credentials saved to ~/.pls-cli/credentials.json)
pls-cli login --email=you@example.com --password=yourpassword

# 2. Upload a model
pls-cli upload model.glb --group=<group-uuid>

# 3. Check upload status (the CLI waits for conversion to complete)
#    Exit 0 = ready, exit 1 = failed
```

---

## Commands

### `login`

```bash
pls-cli login --email=<email> --password=<password>

# Staging environment
pls-cli login --stage --email=<email> --password=<password>
```

Credentials are saved to `~/.pls-cli/credentials.json`. Token lifetime is 24 hours — re-run `login` when it expires.

### `status`

```bash
pls-cli status
```

Prints your account email, account ID, environment (stage / prod), and token expiry.

### `upload`

```bash
# Single file — --group is optional (auto-selects your first group)
pls-cli upload model.zip

# With explicit group
pls-cli upload model.glb --group=<group-uuid>

# Multiple files (max 10)
pls-cli upload file1.zip file2.glb file3.obj --group=<group-uuid>

# Staging environment
pls-cli upload model.obj --group=<group-uuid> --stage

# All conversion options
pls-cli upload model.glb \
  --group=<group-uuid> \
  --ai-enhance \
  --collider \
  --resolution=high \
  --collider-scale=5

# Machine-readable JSON output
pls-cli upload model.zip --json
```

**Upload flags:**

| Flag | Values | Default | Description |
|---|---|---|---|
| `--group` | UUID | auto-selected | Group to upload into. Omit to use your first group. |
| `--stage` | — | prod | Use staging environment. |
| `--ai-enhance` | — | off | Enable AI enhancement. |
| `--collider` | — | off | Generate a collision mesh. |
| `--resolution` | `performance` / `balanced` / `high` / `ultra` | `balanced` | Resolution preset. |
| `--collider-scale` | `0.3` / `2` / `5` / `10` / `100` | `2.0` | Collision mesh scale factor. |
| `--secure` | — | off | Enable encryption. |
| `--json` | — | off | Write JSON result to stdout; progress messages go to stderr. |

### `replace`

```bash
# Replace an existing asset by its ID
pls-cli replace <old-asset-id> new-model.zip

# With options
pls-cli replace <old-asset-id> new-model.glb --stage --collider --json
```

`replace` accepts the same flags as `upload` except `--group` (the original asset ID is used instead).

### `version`

```bash
pls-cli version
# pls-cli v1.0.0
```

---

## Supported File Formats

| Format | Notes |
|---|---|
| `.zip` | Bundle multiple resources into one upload |
| `.glb` | glTF binary |
| `.obj` | Wavefront OBJ |

**Limits:** max 10 files per call · max 500 MB per file

---

## JSON Output (`--json`)

Pass `--json` to get structured output suitable for scripting. Human-readable progress goes to stderr; the result JSON goes to stdout.

**Successful upload:**
```json
{
  "files": [
    {
      "file": "model.glb",
      "assetId": "abc-123-uuid",
      "status": "ready"
    }
  ]
}
```

**Failed upload:**
```json
{
  "files": [
    {
      "file": "bad-model.zip",
      "assetId": "abc-123-uuid",
      "status": "failed",
      "failedType": "convert",
      "error": "Model file corrupted",
      "errorCode": "INVALID_MODEL"
    }
  ]
}
```

**Successful replace:**
```json
{
  "originId": "old-asset-uuid",
  "file": "new-model.glb",
  "assetId": "new-asset-uuid",
  "status": "ready"
}
```

**Parse with shell:**
```bash
result=$(pls-cli upload model.zip --json 2>/dev/null)
asset_id=$(echo "$result" | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['assetId'])")
status=$(echo "$result" | python3 -c "import sys,json; print(json.load(sys.stdin)['files'][0]['status'])")
```

---

## Environments

| | API | Login flag |
|---|---|---|
| **Production** | `https://stream.viverse.com` | (default) |
| **Staging** | `https://stream-stage.viverse.com` | `--stage` |

> The `--stage` flag must match between `login` and `upload` / `replace`. The CLI will error immediately if they don't match — no API call is made.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` | Token expired | Re-run `pls-cli login` |
| `credentials are for prod, but --stage was provided` | Environment mismatch | Re-login with `--stage` |
| `status: "failed"` in JSON | Conversion error | Check `failedType` and `errorCode` |
| `missing client ID` | Dev build without ldflags | Download the official binary from Releases |

---

## How it Works

```
pls-cli upload model.glb
       │
       ├─ 1. Validate file (format, size, count)
       ├─ 2. POST /management/asset         → { id, uploadUrl }
       ├─ 3. PUT  $uploadUrl                → S3 direct upload (progress bar)
       ├─ 4. POST /management/asset/:id/convert
       └─ 5. WebSocket                      → stream conversion progress → exit 0/1
```

---

## License

Internal tool — © VIVEPORT Engineering
