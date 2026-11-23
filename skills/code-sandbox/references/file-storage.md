# Files: in, around and out

There are three places a file can be. Knowing which one you are dealing with prevents most mistakes.

| Place | Who can see it | How it gets there |
|-------|----------------|-------------------|
| Sandbox workspace `/mnt/data` | only your code and `sandbox_read_file` | `input_files`, or your program writes it |
| File storage (object storage) | anyone holding a download link | `sandbox_publish_files` copies a file there |
| The user's computer / chat upload | the user | you cannot touch it; the user downloads a link or uploads a file |

## Bringing files in

Use `input_files` on `sandbox_execute_code` (see [tools.md](tools.md)). Choose the source by size and type:

| Source | Use for | Notes |
|--------|---------|-------|
| `content_text` | CSV/JSON/scripts you already have as text | UTF-8 only |
| `content_base64` | small binary files | keep them small; the base64 text costs context |
| `url` | everything large, uploads, public data | the **MCP server** downloads it; the link must be reachable from the server, not from the sandbox |

Limits: 100 MB per input file. Only `http://` and `https://` URLs. A download that fails returns the HTTP
status or the error type; check the link has not expired. Paths in `name` are relative to `/mnt/data`;
absolute paths are accepted only inside `/mnt/data`, and `..` escapes are refused.

If the user attached a file in chat and you have no usable link for it, say so and ask for a link or
for the content as text. Do not guess the contents.

You can also download inside your own code (`requests`, `curl`): the sandbox has internet access, but it
cannot reach internal services, so internal links only work through `url` in `input_files`.

## Paths inside the sandbox

Relative paths resolve to `/mnt/data`. Subfolders are created for you. Keep a tidy layout:
`data/` for inputs, `out/` for deliverables. Use the paths exactly as reported in `files_changed`
when reading or publishing.

## Taking files out: publishing

`sandbox_publish_files` copies each file from the workspace to file storage and returns a read-only,
time-limited link. Nothing else leaves the sandbox.

- **What the link is:** a signed URL on the organisation's file storage (S3-compatible storage or Azure Blob,
  chosen by the operator; it makes no difference to you). Anyone with the link can download until it expires.
- **Lifetime:** `expires_in_seconds` in the result. The default is usually 24 hours, which is also the
  maximum; you can ask for shorter (minimum 60 seconds). Storage may delete the object after about a day.
  Tell the user to download promptly.
- **Limits:** up to 20 files per call, 200 MB per file, no folders (zip them).
- **Names and types:** the link carries the original file name; the content type is set from the
  extension (`.pptx`, `.docx`, `.xlsx`, `.csv`, `.md`, `.json` and others are known).
- **Each publish is a new copy** with its own link. After you change a file, publish it again and give
  the new link. Do not reuse an old link for new content.
- **Failures:** "file storage is unavailable" means the upload failed but the file is still in the
  sandbox; retry shortly. A missing path or a directory returns an error naming it.

## Writing the answer

- Put each link in a sentence the user can act on: "[Quarterly deck](URL), valid for 24 hours."
- State what each file contains in one line.
- Show key numbers or findings in the answer text as well; the file is the proof, not the only place.
- Never print base64, bytes or `/mnt/data/...` as the deliverable.

## Privacy

Links are bearer links. Publish only what the user asked for. Do not publish raw personal data,
credentials or source files that were only intermediate. Remove scratch files from the list.
