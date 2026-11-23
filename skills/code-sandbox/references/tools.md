# Tool reference

Four tools. Parameters and result fields below match the server implementation
(`code-sandbox/server/src/code_sandbox/server.py`). Every error is an actionable message; read it and
follow it.

## sandbox_execute_code

Run a program and return its output and the files it changed.

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `code` | string | required | complete program source |
| `language` | `python` \| `javascript` \| `bash` | `python` | Python 3.12, Node 22 (ESM or CommonJS, top-level `await` works), bash |
| `session_id` | string | none | omit to start a new sandbox |
| `input_files` | list | none | files written to `/mnt/data` **before** the code runs |
| `timeout_seconds` | 1–300 | 60 | wall-clock limit for this run |

`input_files[]` items: `name` (path relative to `/mnt/data`, e.g. `data/sales.csv`) plus exactly one
of `url`, `content_text`, `content_base64`. Setting none or several is an error. Max 100 MB per file.

Result (text, plus the same fields as structured content):

| Field | Meaning |
|-------|---------|
| `session_id` | pass to every later call |
| `status` | `ok`, `error`, `timeout`, `killed` |
| `exit_code`, `duration_ms` | process result |
| `stdout`, `stderr` | each cut to head + tail at about 16,000 characters |
| `stdout_truncated`, `stderr_truncated` | true when the middle was cut |
| `files_changed` | up to 50 entries `{path, size}` created or modified by this run |
| `files_changed_total` | real count, when more than 50 |

`files_changed` leaves out dot-entries (`.runs`, `.local`, `.cache`), `node_modules` and `__pycache__`.
Files unpacked from an archive keep old timestamps and may not be listed; they exist anyway.

## sandbox_read_file

Look at a file without running code. Read-only.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `session_id` | required | |
| `path` | `.` | file or folder relative to `/mnt/data` |
| `offset_lines` | 0 | text files: first line (0-based) |
| `limit_lines` | 200 (1–2000) | text files: how many lines |

- Text: a page of lines with a header such as `report.md: lines 1-200 of 540 (continue with offset_lines=200)`.
- Image (PNG, JPEG, WebP, GIF, up to 2 MB): returned as an image you can see.
- Folder: a listing two levels deep, without dot-entries and `node_modules`.
- Binary files (PDF, PPTX, XLSX, images over 2 MB) are refused with a hint. Convert to text or PNG
  with `sandbox_execute_code`, or publish them. Text files over 20 MB are refused; use code (`head`, `grep`, pandas).

## sandbox_publish_files

Upload finished files to file storage and return download links.

| Parameter | Default | Notes |
|-----------|---------|-------|
| `session_id` | required | |
| `paths` | required | 1–20 files, relative to `/mnt/data`, exactly as in `files_changed` |
| `expires_in_seconds` | server default (24 h) | minimum 60; capped at the server maximum |

Result text is JSON: `{"message": "...", "files": [{"name", "url", "mime_type", "size", "expires_in_seconds"}]}`.
Structured content has `downloadUrl`, `expiresInSeconds`, `contentType` per file. Each file max 200 MB;
directories are rejected; missing files are an error naming the path. More in [file-storage.md](file-storage.md).

## sandbox_manage_packages

List or install packages inside one session.

| Parameter | Notes |
|-----------|-------|
| `action` | `list` or `install` |
| `language` | `python` (pip) or `javascript` (npm) |
| `packages` | install only; up to 20 registry names with optional version: `pyyaml==6.0.2`, `lodash@4`, `@scope/pkg@^1` |
| `query` | list only; substring filter on names |
| `session_id` | optional, but **omitting it creates a new sandbox**. Always pass it. |

URLs, local paths and flags in `packages` are rejected. Python installs go to `/mnt/data/.local`
(`pip --user`), npm installs to `/mnt/data/node_modules`; both are importable from later calls in the
session and vanish with it. Installs can take up to the 300 s limit. `status` is `installed`,
`failed` or `timeout`. A failed install is a tool error: check the names and versions.

## Which tools can create a session

Only `sandbox_execute_code` and `sandbox_manage_packages` accept a missing `session_id`.
`sandbox_read_file` and `sandbox_publish_files` need an existing one. A malformed id (not the 24-character
hex id a previous call returned) is rejected before anything runs.
