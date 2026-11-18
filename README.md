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
VERSION=$(curl -fsSL -o /dev/null -w '%{url_effective}' \
  https://github.com/ViveportSoftware/pls-cli/releases/latest | grep -oE 'v[0-9]+\.[0-9]+\.[0-9]+$')

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
# 1. Log in — you are prompted for your password (credentials saved to ~/.pls-cli/credentials.json)
pls-cli login --email=you@example.com

# 2. Upload a model
pls-cli upload model.glb --group=<group-uuid>

# 3. Check upload status (the CLI waits for conversion to complete)
#    Exit 0 = ready, exit 1 = failed
```

---

## Commands

### `login`

```bash
# Interactive — prompts for the password without echoing it
# (and for the email too, if --email is omitted)
pls-cli login --email=<email>

# Staging environment
pls-cli login --stage --email=<email>
```

For security, the password is never accepted as a command-line flag, so it cannot leak
into shell history or `ps` output. In scripts and CI, pass it in one of these ways instead:

```bash
# From stdin
printf '%s' "$PLS_CLI_PASSWORD" | pls-cli login --email=<email> --password-stdin

# From the PLS_CLI_PASSWORD environment variable
PLS_CLI_PASSWORD="..." pls-cli login --email=<email>
```

The password is read from `--password-stdin` first, then `PLS_CLI_PASSWORD`, then an
interactive prompt. Without a terminal and without either of the first two, `login` fails
instead of waiting for input.

> **Upgrading from v1.x:** `--password` was removed in v2.0.0. Replace
> `--password="$PW"` with `PLS_CLI_PASSWORD="$PW" pls-cli login ...` or `--password-stdin`.

Credentials are saved to `~/.pls-cli/credentials.json`. Token lifetime is 24 hours — re-run `login` when it expires.

### `status`

```bash
pls-cli status
```

Prints your account email, account ID, environment (stage / prod), and token expiry.

### `logout`

```bash
pls-cli logout
```

Deletes `~/.pls-cli/credentials.json`. The token is not revoked on the server; it stays valid until it expires. Running `logout` when you are not logged in also succeeds.

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
| `--tags` | comma-separated names | — | Tag names to assign after conversion. Missing tags are created. |

### `replace`

```bash
# Replace an existing asset by its ID
pls-cli replace <old-asset-id> new-model.zip

# With options
pls-cli replace <old-asset-id> new-model.glb --stage --collider --json
```

`replace` accepts the same flags as `upload` except `--group` (the original asset ID is used instead).

### `list`

```bash
# List assets in your first group
pls-cli list

# With explicit group
pls-cli list --group=<group-uuid>

# Machine-readable JSON output
pls-cli list --group=<group-uuid> --json
```

Prints each asset's ID, status and name. Flags: `--group`, `--stage`, `--json`.

> When `--group` is omitted, the CLI prints an `auto-selected group: ...` line on stdout before the result. Pass `--group` with `--json` so stdout holds only the JSON.

```json
{
  "assets": [
    {
      "id": "asset-uuid",
      "name": "example-model",
      "status": "ready",
      "createdAt": "2026-11-18T09:00:00Z"
    }
  ]
}
```

### `delete`

```bash
pls-cli delete <asset-id>

# Staging environment, JSON output
pls-cli delete <asset-id> --stage --json
```

Deletes the asset immediately — there is no confirmation prompt and it cannot be undone. Flags: `--stage`, `--json`.

```json
{ "assetId": "asset-uuid", "deleted": true }
```

On failure, `deleted` is `false` and the exit code is 1.

### `tag`

Tags belong to a group. `create` and `list` take `--group` (omit it to use your first group); every `tag` subcommand accepts `--stage` and `--json`.

```bash
# Create a tag
pls-cli tag create <name> --group=<group-uuid>

# List tags in a group
pls-cli tag list --group=<group-uuid>

# Assign one or more tags to an asset (by tag UUID)
pls-cli tag assign <tag-uuid> [tag-uuid...] --asset=<asset-id>
```

To tag a new asset in one step, pass tag **names** to `upload` or `replace` with `--tags`. Missing tags are created and all of them are assigned after conversion:

```bash
pls-cli upload model.glb --group=<group-uuid> --tags=foo,bar
```

JSON output:

| Command | Shape |
|---|---|
| `tag create` | `{ "uuid": "...", "name": "..." }` |
| `tag list` | `{ "tags": [ { "uuid": "...", "name": "..." } ] }` |
| `tag assign` | `{ "assetId": "...", "tagUuids": ["..."] }` |
| `upload` / `replace` with `--tags` | each upload file result, or the replace result, gains `"assignedTags": [ { "uuid": "...", "name": "..." } ]` |

### `version`

```bash
pls-cli version
# pls-cli v2.0.0
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
| `unknown flag: --password` | Script written for v1.x | Use `--password-stdin` or `PLS_CLI_PASSWORD` (see `login`) |
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
