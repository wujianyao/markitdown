# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

MarkItDown — a Python utility that converts many file formats to Markdown for LLM/text-analysis pipelines. The repo is a **multi-package monorepo**:

- `packages/markitdown` — core library, CLI entry point (`markitdown`), and test suite
- `packages/markitdown-mcp` — MCP server wrapping the library
- `packages/markitdown-ocr` — optional plugin that adds LLM-Vision OCR
- `packages/markitdown-sample-plugin` — reference skeleton for 3rd-party plugins

Scope is intentionally narrow: improvements to existing converters, bug/perf/security fixes, the CLI, the MCP package, tests, and developer tooling. Out of scope (per `README.md`): web servers, REST/HTTP APIs, hosted services, web/desktop/mobile apps, and Electron/PyQt/Tkinter front-ends — those belong in separate packages on PyPI.

Python: **3.10 through 3.14**. Build/test runner: **hatch**. Required `pip install hatch` first.

## Security & I/O

MarkItDown executes I/O with the privileges of the current process — like `open()` / `requests.get()`. Inputs in untrusted environments must be sanitized before reaching it. Prefer the narrowest entry point that fits the use case:

- `convert_local()` — local path only
- `convert_response()` — caller-supplied `requests.Response`
- `convert_stream()` / `convert_uri()` — last resort; full permissiveness

REST/HTTP endpoints are intentionally **not** provided by this library.

## Common commands

Run from `packages/markitdown`:

```bash
# Install with every optional dependency (most tests assume this)
pip install -e 'packages/markitdown[all]'
pip install hatch            # if not already installed

hatch test                   # full suite
hatch test tests/test_pdf.py # single file
hatch run types:check        # mypy (hatch env "types", src + tests)

# Plugin / MCP / OCR packages install the local sibling first, then run pytest:
pip install ./packages/markitdown ./packages/markitdown-ocr
cd packages/markitdown-ocr && pytest

pre-commit run --all-files    # black 23.7.0 (only hook configured currently)
```

CLI quick reference (see `packages/markitdown/src/markitdown/__main__.py` for the full surface):

```bash
markitdown path/to/file.pdf -o out.md
markitdown -d -e "<docintel-endpoint>" report.pdf      # Azure Document Intelligence
markitdown --use-cu --cu-endpoint "<cu-endpoint>"      # Azure Content Understanding
markitdown --cu-analyzer my-invoice-analyzer inv.pdf   # custom CU analyzer (YAML front matter)
MARKITDOWN_DOCINTEL_ENDPOINT=… markitdown -d report.pdf
```

## Repository layout (core package)

`packages/markitdown/src/markitdown/`:

| File | Role |
| --- | --- |
| `_markitdown.py` | `MarkItDown` orchestrator class (converter registry, dispatch, `_get_stream_info_guesses`) |
| `_base_converter.py` | Abstract `DocumentConverter` + `DocumentConverterResult` |
| `_stream_info.py` | Frozen dataclass `StreamInfo` (mimetype, extension, charset, filename, local_path, url) |
| `_exceptions.py` | `MarkItDownException`, `MissingDependencyException`, `UnsupportedFormatException`, `FileConversionException`, `FailedConversionAttempt`, and `MISSING_DEPENDENCY_MESSAGE` template |
| `_uri_utils.py` | `parse_data_uri`, `file_uri_to_path` |
| `__main__.py` | `argparse` CLI; uses env vars `MARKITDOWN_DOCINTEL_ENDPOINT` / `MARKITDOWN_CU_ENDPOINT` |
| `converters/` | One converter per format — see "Adding a converter" below |
| `converter_utils/` | Shared format helpers (`docx/pre_process.py`, `docx/math/omml.py`+`latex_dict.py`, `_image.py`, `_xlsx_images.py`) |

## Big-picture architecture (read these files to understand the design)

1. **`DocumentConverter` contract** (`_base_converter.py`):
   - `accepts(file_stream, stream_info, **kwargs) -> bool` — **must not** advance the stream position. Defensive `seek(tell())` is the pattern when peeking.
   - `convert(file_stream, stream_info, **kwargs) -> DocumentConverterResult` — returns Markdown `str` plus optional `title`.
   - Optional deps are imported under `try/except ImportError`; on miss, `convert()` raises `MissingDependencyException` whose message points to the right extras group (`MISSING_DEPENDENCY_MESSAGE` is filled in with the converter's `[pdf]`/`[docx]`/etc. feature flag).

2. **`MarkItDown.convert*` family** (`_markitdown.py`): entry points route any of `str` / `Path` / `requests.Response` / `BinaryIO` (with `http(s)` / `file:` / `data:` URIs supported) into a `StreamInfo` base guess, then call `_convert`. `convert_stream` re-buffers non-seekable streams into `BytesIO` before dispatch — Windows pipe-backed stdin reports `seekable() == True` but cannot rewind, which is why the CLI also reads `sys.stdin.buffer.read()` into `BytesIO` in `__main__`.

3. **Dispatch (`_convert`)**:
   - Stream hints + `magika` identify + `charset_normalizer` charset → a `List[StreamInfo]` of guesses.
   - `_converters` is sorted by `priority` on every call (lower value first); stable sort so equal-priority ties keep insertion order.
   - `register_converter()` does `insert(0, …)`, so **most recently registered runs first** within a priority bucket.
   - Per converter: invoke `accepts()` → if true, invoke `convert()` in a try/except that collects `FailedConversionAttempt`s and `seek(0)`s the stream after each attempt.
   - First non-None result wins, then output is normalized (`re.split(r"\r?\n")`, rstrip each line, collapse 3+ blank lines → `\n\n`).
   - Constant priorities: `PRIORITY_SPECIFIC_FILE_FORMAT = 0.0`, `PRIORITY_GENERIC_FILE_FORMAT = 10.0`. The generic bucket (`PlainTextConverter`, `ZipConverter`, `HtmlConverter`) intentionally runs after format-specific ones. Plugins pick any priority; cloud converters (DocumentIntelligence, ContentUnderstanding) are inserted at the top of the stack when their endpoint kwarg is set in `enable_builtins`.

4. **Plugin protocol**: third-party packages register themselves via the standard Python entry-point group `markitdown.plugin` (see `packages/markitdown-sample-plugin/pyproject.toml`). The plugin module must export `__plugin_interface_version__ = 1` and `register_converters(markitdown: MarkItDown, **kwargs)`, which calls `markitdown.register_converter(...)` and may also read per-call kwargs such as `llm_client` / `llm_model` from `_llm_client`/`_llm_model` stored on the `MarkItDown` instance. Loading is lazy and exception-isolated — failed plugins `warn()` and are skipped, not raised.

5. **HTML pipeline** (`converters/_markdownify.py` + `_html_converter.py`): every HTML-emitting format eventually funnels through `_CustomMarkdownify`, a `markdownify.MarkdownConverter` subclass that:
   - defaults to `ATX` heading style,
   - drops `javascript:`/non-`http(s)`/non-`file:` hrefs,
   - truncates large `data:` URI images to `data:<mime>;...` unless `keep_data_uris=True` is passed,
   - prefers `data-src` over `src` for lazy-load placeholders (unless the caller opts into `keep_data_uris`),
   - treats `<u>` as literal `<u>…</u>`,
   - maps checkboxes to `[x]` / `[ ]`,
   - preserves `%HH` octets in URL paths (see `_quote_path_preserving_percent_encoded_octets`).
   `DocxConverter` reuses `HtmlConverter` after running the embedded style map through `mammoth` and applying `pre_process.docx` (handles math, comments, embedded images).

6. **LLM features**: `MarkItDown(llm_client=…, llm_model=…, llm_prompt=…)` enables LLM-generated captions for image/pptx (and via the OCR plugin, PDF/DOCX/PPTX/XLSX). The client is retryable; if all retries fail, `convert()` raises and falls through to `FileConversionException` after the chain runs out.

## Tests

Layout and conventions are documented in `packages/markitdown/tests/README.md`. The key idea:

- `tests/_test_vectors.py` defines `GENERAL_TEST_VECTORS` and `DATA_URI_TEST_VECTORS` as `FileTestVector(filename, mimetype, charset, url, must_include, must_not_include)` records.
- `tests/test_cli_vectors.py` exercises the vectors through the CLI.
- `tests/test_module_vectors.py` exercises the same vectors through `convert()`, `convert_stream()`, and `convert_uri()`.
- Format-specific suites live next to that format (`test_docx.py`, `test_pdf.py`, `test_xlsx.py`, `test_pptx.py`, `test_html.py`, `test_rss.py`, `test_epub.py`, `test_outlook_msg.py`, `test_image.py`, `test_youtube.py`, `test_csv.py`, `test_zip.py`, `test_docintel.py`, `test_cu.py`).
- `test_module_misc.py` covers library infrastructure, cross-format options, charset detection, URI handling, plus the legacy audio/ipynb regressions. `test_cli_misc.py` covers CLI behavior outside the vector matrix.

Pre-existing skip conditions apply — remote, credential-, platform-, and optional-dependency-gated tests skip silently (e.g. `skip_remote = os.environ.get("GITHUB_ACTIONS")`).

PyMuPDF is only needed to **rebuild** PDF cleanup fixtures (`pdf_cleanup_{plain,form,mixed}.pdf`) via `tests/test_files/generate_pdf_cleanup_fixtures.py`; it is not a runtime test dependency.

## CI / hooks

`.github/workflows/`:

- `tests.yml` — matrix over Ubuntu × Python 3.10–3.14, plus Windows × 3.12, plus Windows-11-VS2026-ARM × 3.12. Installs `libimage-exiftool-perl` + `ffmpeg` on Linux; runs `hatch test --python <ver>` in `packages/markitdown`. Separate `ocr-tests` and `mcp-tests` jobs install the local sibling package then run their own `pytest`. Python is invoked with `PYTHONUTF8=1` to preserve Unicode in CLI output capture.
- `pre-commit.yml` — runs the repo's `.pre-commit-config.yaml` (currently just `black 23.7.0`).
- `dependabot.yml` — weekly GitHub Actions updates with a 7-day cooldown.
- `.github/constraints.txt` — pins `cryptography==46.0.3` on Windows ARM64 to work around missing wheels.

`.gitattributes` marks test fixture PDFs as binary and excludes `**/tests/test_files/**` from linguist so generated fixtures don't skew the language statistics.

## Adding a converter

Pattern: create `packages/markitdown/src/markitdown/converters/_<format>_converter.py` declaring module-level `ACCEPTED_MIME_TYPE_PREFIXES` and `ACCEPTED_FILE_EXTENSIONS`, then a `class XxxConverter(DocumentConverter)` with `accepts()` (check `stream_info.mimetype`/`stream_info.extension`, never advance the stream) and `convert()` that returns `DocumentConverterResult(markdown=…, title=…)`. Re-export it from `converters/__init__.py`, then register in `_markitdown.py: enable_builtins()` at either `PRIORITY_SPECIFIC_FILE_FORMAT` (format-specific) or `PRIORITY_GENERIC_FILE_FORMAT` (catch-all like plain text / HTML / ZIP). If the converter needs to recurse on nested files (e.g. ZIP/EPUB entries), the parent `_converters` list is forwarded in kwargs as `_parent_converters`.

For optional dependencies: use the `try/except ImportError` → `MissingDependencyException(message=MISSING_DEPENDENCY_MESSAGE.format(...))` pattern so users get an actionable hint pointing to the right `pip install 'markitdown[<extra>]'` invocation. Add the dep to the relevant feature group (`[pdf]`, `[docx]`, `[xlsx]`, `[xls]`, `[pptx]`, `[outlook]`, `[audio-transcription]`, `[youtube-transcription]`, `[az-doc-intel]`, `[az-content-understanding]`) **and** to `[all]`.
