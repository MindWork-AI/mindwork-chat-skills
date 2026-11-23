# Troubleshooting

Match the symptom, apply the fix. All messages come from the server and are meant to be acted on.

| Symptom / message | Cause | Fix |
|-------------------|-------|-----|
| "Session '…' does not exist or has expired" | idle 20 min, 4 h cap, or evicted | omit `session_id`; recreate inputs and rerun |
| "'…' is not a valid session_id" | typo or a made-up id | use the exact id from a previous reply |
| "All sandboxes are busy right now" | server at its sandbox limit, all running | wait a few seconds, retry the same call |
| "This sandbox stopped unexpectedly … discarded" | usually out of memory | start a new session; use chunks, smaller images |
| "The sandbox could not be started because the runtime is unavailable" | Docker/runtime problem | retry once; then tell the user the service needs an operator |
| `status: timeout` | program ran past `timeout_seconds` | raise it (max 300) or split the work across calls |
| `status: killed` | memory limit | `pd.read_csv(chunksize=…)`, `usecols=`, lower resolution, `chdb` |
| `status: error`, traceback in `stderr` | your program failed | read the last lines, fix, rerun |
| `status: ok` but the build clearly failed | piped command hid the failure | start bash with `set -eo pipefail` |
| "Input file '…': set exactly one of url, content_text or content_base64" | none or several sources | give exactly one |
| "Input file '…': download failed with HTTP …" | link expired or not reachable from the server | get a fresh link; internal links must be reachable from the MCP server |
| "Input file '…' is larger than 100 MB" | size limit | download it inside your code instead, or filter at the source |
| "Path '…' is outside the workspace" | absolute path or `..` outside `/mnt/data` | use a path relative to `/mnt/data` |
| "'…' does not exist in /mnt/data" | wrong path or the run did not create it | check `files_changed` or list the folder with `sandbox_read_file` |
| "'…' is a directory" on publish | folders are not accepted | `zip -r out.zip folder`, publish the zip |
| "'…' is … ; the limit is 200 MB" | file too big | compress, split, or lower quality |
| "'…' is a binary file" on read | PDF/Office/large image | render to PNG or extract text with code |
| "Invalid package spec(s)" | URL, path or flag in `packages` | registry names only; run `pip install <url>` in bash if you must |
| install `status: failed` | wrong name/version or no network | check spelling and version; check network |
| "file storage is unavailable" on publish | storage outage | the file is still in the sandbox; retry in a moment |
| Chromium fails to launch | Chromium's own sandbox | pass `chromium_sandbox=False` / `chromiumSandbox: false` |
| `soffice` hangs or fails when run twice | one LibreOffice instance per session | convert all files in one command; do not run in parallel |
| `javaldx` warning on stderr | harmless | ignore if the output file exists |
| An old link shows old content | you changed the file after publishing | publish again, give the new link |

## When to stop and tell the user

- The same tool call fails twice with the same service-level message (runtime unavailable, storage unavailable).
- The task needs something the sandbox does not allow (internal network access, root, a paid service).
- A file would need to contain data you were not given. Ask for it instead of inventing values.

Say what you tried, what failed, and what you can still do (for example give the result as text or
a table in the answer).
