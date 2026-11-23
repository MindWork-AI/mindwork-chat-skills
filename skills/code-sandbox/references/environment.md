# The sandbox environment

Read this when you need to know whether something is installed, where files go or why a program was stopped.

## Where things live

| Path | Writable | Survives the call | Use it for |
|------|----------|-------------------|------------|
| `/mnt/data` | yes | yes (session) | inputs, outputs, projects, installed packages. Working directory. |
| `/tmp` | yes | no guarantee | scratch (512 MB, counts against RAM) |
| `/home/sandbox` | yes | no guarantee | `$HOME`, tool config (64 MB) |
| everything else | no | – | read-only system image |

The program runs as an unprivileged user. `apt-get` and `sudo` do not work.

## Preinstalled

**Python 3.12**: numpy, pandas, scipy, statsmodels, scikit-learn, scikit-image, sympy, networkx, pyarrow,
chdb (SQL over files); matplotlib, seaborn, plotly, Pillow, opencv, cairosvg, qrcode; openpyxl, xlsxwriter,
python-docx, docxtpl, python-pptx, markitdown; reportlab, pypdf, pdfplumber, pypdfium2, pytesseract,
weasyprint; geopandas, rasterio, folium; playwright, requests, httpx, beautifulsoup4, lxml, jinja2.

**Node 22** (`npm`, `npx`, `pnpm`), importable from any folder: pptxgenjs, docx, exceljs, pdf-lib, sharp,
jszip, archiver, playwright, react, react-dom, react-icons, cheerio, marked, markdown-it, papaparse, lodash,
dayjs, date-fns, zod, js-yaml, handlebars, ejs.

**Command line**: `soffice` (LibreOffice), `pdftoppm`, `pdftotext`, `pdfinfo`, `qpdf`, `pandoc`, `tesseract`
(English), `gs`, `convert` (ImageMagick), `git`, `curl`, `jq`, `zip`, `unzip`.

**Browser**: one Playwright Chromium for Python and Node. Fonts: Liberation (Arial/Times), Carlito/Caladea
(Calibri/Cambria), DejaVu, Noto including CJK and emoji.

Check an exact name with `sandbox_manage_packages(action="list", language="python", query="pdf", session_id=...)`.
The image may change; the list tool is the source of truth.

## Limits

| Limit | Value | When you hit it |
|-------|-------|-----------------|
| Time per call | 60 s default, up to 300 s | `status: timeout`, program stopped |
| Memory | 2 GB, no swap | `status: killed` |
| CPU | 2 cores | slower |
| Processes | 512 | fork/thread errors |
| Open files | 4096 | `EMFILE` |
| stdout/stderr | about 16,000 characters each | middle is cut |
| Input file | 100 MB each | call rejected |
| Published file | 200 MB each | rejected; compress or split |
| Image via read_file | 2 MB | downscale it |
| Idle / total session | 20 min / 4 h | sandbox and files removed |

Operators can change these values; the numbers above are the defaults. If behaviour differs, trust the
error message.

## Network

Outbound internet works (pip, npm, git clone, public APIs). Other sandboxes, the host and internal services
are blocked, and nothing can connect in from outside. Installs and downloads need the network, so if the
operator turned it off they fail; say so.
