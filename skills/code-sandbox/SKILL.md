---
name: code-sandbox
description: "Run code and create files with the code-sandbox MCP tools (sandbox_execute_code, sandbox_read_file, sandbox_publish_files, sandbox_manage_packages): isolated Python, Node.js and bash sessions whose results reach the user as download links. Use this skill whenever a task needs code to run or a file to be generated, even if the user never mentions code - calculations, CSV or Excel analysis, charts, Word, PowerPoint, Excel or PDF files, file conversions, OCR, HTML pages or small web apps with screenshots. Also use it when the user asks for a downloadable file, or when you are unsure how sessions, input files or published links work."
compatibility: "Requires the code-sandbox MCP server with the tools sandbox_execute_code, sandbox_read_file, sandbox_publish_files and sandbox_manage_packages. Gateways may prefix the tool names (for example code-sandbox__sandbox_execute_code)."
---

# Working in the code sandbox

The sandbox is a private Linux machine that lives for one conversation. You send it a program, it runs,
and you get back what the program printed plus the list of files it created. Nothing reaches the user's
computer, and nothing reaches the user unless you publish it as a download link. That boundary shapes the
whole workflow below: you do the work inside the sandbox, check it there, and only then hand it over.

If none of the four tools is available in this conversation, say so. Do not describe results of code
you did not run.

## The tools

| Tool | Use it to | `session_id` |
|------|-----------|--------------|
| `sandbox_execute_code` | run Python, JavaScript (Node) or bash; upload input files | optional: omit to start a sandbox |
| `sandbox_read_file` | page through a text file, see an image, list a folder | required |
| `sandbox_publish_files` | upload final files to file storage, get download links | required |
| `sandbox_manage_packages` | list or install Python / npm packages | optional, but omitting it starts a new empty sandbox |

Parameters and result fields are in [references/tools.md](references/tools.md).

## The loop

1. **Start.** Call `sandbox_execute_code` without `session_id`. The reply begins with
   `session_id: <24 hex characters>`. Keep it: it is the only handle to your files.
2. **Reuse it** on every later call. A new session means an empty workspace, so starting one per step loses your work.
3. **Run in scripts, not notebooks.** Each call is a fresh process. Files in `/mnt/data` survive; variables,
   imports and background servers do not. Save anything the next step needs to a file.
4. **Check.** Read `status`, then look at what you made (next section).
5. **Publish** the final files and link them in your answer.

How sessions live, expire and queue: [references/sessions.md](references/sessions.md).

## Input files

Put files into the sandbox with `input_files` on `sandbox_execute_code`. Each item has a `name` (path inside
`/mnt/data`) and exactly one of `url`, `content_text` or `content_base64`. Use `url` for anything large or
uploaded (the MCP server downloads it, up to 100 MB), `content_text` for small text such as CSV, JSON or a
script, and `content_base64` only for small binaries, because base64 text costs context. If the user
attached a file but you have no usable link or text for it, ask for one rather than guessing its contents.
Details: [references/file-storage.md](references/file-storage.md).

## Reading the result

| `status` | Meaning | Next step |
|----------|---------|-----------|
| `ok` | exit code 0 | continue |
| `error` | non-zero exit, see `stderr` | read the last lines of the traceback, fix, rerun |
| `timeout` | exceeded `timeout_seconds` (default 60, max 300) | raise the limit or split the work |
| `killed` | stopped by the system, almost always the 2 GB memory limit | process in chunks, lower resolution |

`stdout` and `stderr` keep only their start and end (about 16,000 characters each), so print summaries and
write large results to files. `files_changed` lists what the run created or modified; use those exact paths
for reading and publishing.

## Check before you show

A file that opens is not the same as a file that is right, and the user cannot fix it from a link. So look at
what you made before you publish it:

- text and data: `sandbox_read_file` returns numbered lines (`offset_lines` continues)
- images up to 2 MB (PNG, JPEG, WebP, GIF): returned as images you can see
- PDF, PPTX, DOCX, XLSX cannot be shown directly. Render them to PNG first, then read the PNGs:
  `soffice --headless --convert-to pdf f.pptx` then `pdftoppm -png -r 60 f.pdf page`

Look for overflowing text, overlaps, empty charts and numbers that do not add up; fix and render again.

## Publish

Call `sandbox_publish_files` with the final files, for example `paths=["deck.pptx"]` (up to 20 per call). Each
file gets a time-limited download link. Give it to the user as a Markdown link and say when it expires
(`expires_in_seconds`, usually 24 hours). Publish deliverables, not scratch files. Folders are rejected, so zip
them first. Never paste file contents, base64 or `/mnt/data/...` paths as the answer, because the user cannot
open sandbox paths. If you change a file after publishing, publish again: links keep the bytes of the moment
they were made. Links work for anyone who holds them, so publish only what the user asked for.

## Which reference to read

Read the one that matches the task. They are independent; you rarely need more than two.

| The task involves | Read |
|-------------------|------|
| Word, PowerPoint, Excel, PDF, form filling, Markdown or HTML conversions, OCR | [references/documents.md](references/documents.md) |
| a CSV/Excel/Parquet file, statistics, tables, charts | [references/data-analysis.md](references/data-analysis.md) |
| an HTML page, React app, screenshot or browser test | [references/web-frontend.md](references/web-frontend.md) |
| exact parameters, result fields, limits of a tool | [references/tools.md](references/tools.md) |
| session expiry, "session does not exist", capacity | [references/sessions.md](references/sessions.md) |
| how to get a file in, how links work, storage, privacy | [references/file-storage.md](references/file-storage.md) |
| is library X installed, paths, memory and time limits, network | [references/environment.md](references/environment.md) |
| any error message you do not understand | [references/troubleshooting.md](references/troubleshooting.md) |

## Habits that prevent most failures

- Start bash programs with `set -eo pipefail`. Without it, `build | tail` reports `ok` even when the build failed.
- Pass `chromium_sandbox=False` (Python) or `chromiumSandbox: false` (Node) when launching Chromium; its own
  sandbox cannot start inside this one.
- Start a server, test it and stop it inside one call; it will not survive to the next.
- Run LibreOffice once per command with all files (`soffice --headless --convert-to pdf a.docx b.pptx`); a
  second instance in the same session fails.
- Check for a library before installing: `sandbox_manage_packages(action="list", query="docx", session_id=...)`.
- Internet is on for downloads and installs, but internal services are unreachable. Do not send the user's
  private data to third-party services unless they asked.

## Example: a five-slide deck from a sales CSV

1. `sandbox_execute_code` with `input_files=[{"name": "sales.csv", "url": "<link>"}]` loads the data, prints
   its shape and a few rows, and saves a cleaned `quarterly.parquet`. Note the returned `session_id`.
2. Same session: compute the trend and write `trend.png` with matplotlib. `sandbox_read_file("trend.png")` shows
   whether the chart is readable.
3. Same session: build `deck.pptx` with pptxgenjs, convert to PDF, render pages to PNG, look at each, fix.
4. `sandbox_publish_files(session_id, ["deck.pptx"])`, then answer with the key numbers, a one-line description
   of the deck and the link with its expiry.

If the sandbox service itself is unavailable after one retry, tell the user plainly and offer what you can
do without code, such as giving the result as text or a table.
