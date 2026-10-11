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

## Startup methods & core entry points

### Three launch modes

| Surface | Command / call | Entry point |
|---|---|---|
| **CLI** | `markitdown <file>` or `python -m markitdown` (after `pip install -e 'packages/markitdown[all]'`) | `packages/markitdown/src/markitdown/__main__.py::main()` — supports stdin, files, `http(s)`, `file:`, `data:` URIs |
| **Python API** | `from markitdown import MarkItDown` → `MarkItDown(...)` → `.convert(...)` | `packages/markitdown/src/markitdown/_markitdown.py::MarkItDown`; prefer the narrow `convert_local()` / `convert_response()` over `convert_uri()` / `convert_stream()` |
| **MCP server** | install `markitdown-mcp` and run its entrypoint | `packages/markitdown-mcp/src/markitdown_mcp/__main__.py` — MCP-protocol wrapper around `MarkItDown` |

Optional siblings: `markitdown-ocr` (LLM-Vision OCR), `markitdown-sample-plugin` (3rd-party plugin template).

### Call chain at a glance

```
__main__.py::main()                          ← CLI argv
   └─ creates MarkItDown
      └─ .convert() / convert_stream() / convert_uri()
                       / convert_local() / convert_response()
         └─ ._convert(stream_info)           ← real dispatch
            ├─ _load_plugins()               ← entry_points(group="markitdown.plugin")
            ├─ enable_builtins()             ← registers every built-in converter
            ├─ sort _converters by priority  (SPECIFIC=0.0 before GENERIC=10.0)
            └─ for each converter: accepts() → convert()  (first non-None wins)
```

### Key files

- **Entry points** — `packages/markitdown/src/markitdown/__main__.py` (CLI), `packages/markitdown-mcp/src/markitdown_mcp/__main__.py` (MCP).
- **Orchestrator** — `packages/markitdown/src/markitdown/_markitdown.py::MarkItDown` (the `convert*` family, `_convert`, `enable_builtins`, `register_converter`, `_load_plugins`).
- **Converter contract** — `packages/markitdown/src/markitdown/_base_converter.py` (`DocumentConverter`, `DocumentConverterResult`).
- **Stream metadata** — `packages/markitdown/src/markitdown/_stream_info.py` (`StreamInfo` frozen dataclass).
- **Exceptions** — `packages/markitdown/src/markitdown/_exceptions.py` (and the `MISSING_DEPENDENCY_MESSAGE` template).
- **URI parsing** — `packages/markitdown/src/markitdown/_uri_utils.py` (`parse_data_uri`, `file_uri_to_path`).
- **Built-in converters** — `packages/markitdown/src/markitdown/converters/` (one file per format).
- **Shared helpers** — `packages/markitdown/src/markitdown/converter_utils/` (docx math, image extraction, xlsx image handling).

### Plugin extension point

Third-party packages register via the standard Python entry-point group:

- group: `markitdown.plugin`
- module-level constant: `__plugin_interface_version__ = 1`
- entry function: `register_converters(markitdown: MarkItDown, **kwargs)` — calls `markitdown.register_converter(MyConverter())` and may read per-call kwargs like `_llm_client` / `_llm_model` off the `MarkItDown` instance.
- Plugins are loaded lazily; an exception only emits a `warn()` and skips that plugin.
- Cloud converters (`DocumentIntelligenceConverter`, `ContentUnderstandingConverter`) are inserted at the top of the stack when their endpoint kwarg is set in `enable_builtins`.

Reference: `packages/markitdown-sample-plugin/pyproject.toml`.

## 功能全景图（Feature Landscape）

### 1. 总览（一图看全）

```
                          ┌───────────────────────────────────────────────────────────────┐
                          │              用户（CLI / Python API / MCP / 插件）            │
                          └───────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
                  ┌───────────────────────────────────────────────────────────────┐
                  │  入口层  __main__.py::main()  /  MarkItDown 类  /  markitdown-mcp │
                  └───────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
                  ┌───────────────────────────────────────────────────────────────┐
                  │            编排层  _convert(stream_info)                          │
                  │   （magika 识别 + charset_normalizer + entry_points 插件加载）    │
                  └───────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
        ┌───────────────────────────────────────────────────────────────────────────────────┐
        │                    转换器调度（按 priority 排序）                                 │
        │   • PRIORITY_SPECIFIC_FILE_FORMAT = 0.0  （具体格式，先跑）                        │
        │   • PRIORITY_GENERIC_FILE_FORMAT = 10.0 （catch-all，后跑）                        │
        │   • 云转换器当 endpoint 启用时 → 插到栈顶                                          │
        └───────────────────────────────────────────────────────────────────────────────────┘
                                                  │
        ┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
        ▼              ▼              ▼              ▼              ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ PDF/DOCX │  │ XLSX/XLS │  │ PPTX/图片 │  │ HTML/RSS │  │ ZIP/EPUB │  │ Outlook  │  │ IPYNB/   │
  │ /PPTX    │  │ /CSV     │  │ /音频    │  │ /Wiki    │  │ /递归    │  │ MSG      │  │ YouTube/ │
  │          │  │          │  │          │  │ /YouTube │  │          │  │          │  │ Bing     │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
        │              │              │              │              │              │              │
        └──────────────┴──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
                                                  │
                                                  ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────┐
        │  契约层  DocumentConverter + DocumentConverterResult + StreamInfo            │
        │   accepts() 仅读 StreamInfo；convert() 读流 → 产 Markdown                         │
        │   （seek/tell 不推进流位置；缺包抛 MissingDependencyException）                   │
        └─────────────────────────────────────────────────────────────────────────────────────────┘
                                                  │
                                                  ▼
                  ┌───────────────────────────────────────────────────────────────┐
                  │           输出层  Markdown 文本 + 可选 title                    │
                  │   规范化：3+ 空行折叠为 \n\n；每行 rstrip                        │
                  └───────────────────────────────────────────────────────────────┘
```

### 2. 支持的格式矩阵

| 类别 | 格式 | 入口转换器 | 可选依赖组 | 优先级 |
|---|---|---|---|---|
| 文档 | PDF | `_pdf_converter.py::PdfConverter` | `[pdf]` | SPECIFIC (0.0) |
| 文档 | DOCX | `_docx_converter.py::DocxConverter`（mammoth + `_markdownify`） | `[docx]` | SPECIFIC |
| 文档 | PPTX | `_pptx_converter.py::PptxConverter` | `[pptx]` | SPECIFIC |
| 表格 | XLSX | `_xlsx_converter.py::XlsxConverter` | `[xlsx]` | SPECIFIC |
| 表格 | XLS | `_xls_converter.py::XlsConverter` | `[xls]` | SPECIFIC |
| 表格 | CSV | `_csv_converter.py::CsvConverter` | 核心 | SPECIFIC |
| 网页 | HTML | `_html_converter.py::HtmlConverter`（`_markdownify`） | 核心 | GENERIC (10.0) |
| 网页 | 维基百科 | `WikipediaConverter`（HTML 派发） | 核心 | SPECIFIC |
| 网页 | RSS | `_rss_converter.py::RssConverter` | 核心 | SPECIFIC |
| 网页 | YouTube | `_youtube_converter.py::YouTubeConverter` | `[youtube-transcription]` | SPECIFIC |
| 网页 | Bing SERP | `BingSerpConverter` | 核心 | SPECIFIC |
| 邮件 | Outlook MSG | `_outlook_msg_converter.py::OutlookMsgConverter` | `[outlook]` | SPECIFIC |
| 多媒体 | 图片 | `_image_converter.py::ImageConverter`（含 EXIF） | 核心 | SPECIFIC |
| 多媒体 | 音频 | `_audio_converter.py::AudioConverter` | `[audio-transcription]` | SPECIFIC |
| 容器 | ZIP | `_zip_converter.py::ZipConverter`（递归分派子条目） | 核心 | GENERIC |
| 容器 | EPUB | `_epub_converter.py::EpubConverter` | 核心 | GENERIC |
| 数据 | IPYNB | `_ipynb_converter.py::IpynbConverter` | 核心 | SPECIFIC |
| 兜底 | 纯文本 | `PlainTextConverter` | 核心 | GENERIC |
| 云 | Document Intelligence | `DocumentIntelligenceConverter` | `[az-doc-intel]` | 栈顶（启用时） |
| 云 | Content Understanding | `ContentUnderstandingConverter` | `[az-content-understanding]` | 栈顶（启用时） |

### 3. 核心能力（横切关注点）

- **多形态输入统一**：`convert(str | Path | requests.Response | BinaryIO | URI)` 全部路由到统一的 `StreamInfo`
- **非可寻流重缓冲**：`convert_stream` 与 CLI 主动将不可寻流 rewind 进 `BytesIO`（规避 Windows 管道 stdin 的 `seekable()==True` 假象）
- **自动识别**：magika 内容嗅探 + mimetypes 兜底 + `charset_normalizer` 字符集 + 文件名后缀
- **可选依赖管理**：缺包时抛 `MissingDependencyException`，消息模板指向正确的 `pip install 'markitdown[<extra>]'`
- **失败隔离**：每个转换器用 `try/except` 收集 `FailedConversionAttempt`，继续尝试下一个；首个非 None 结果胜出
- **LLM 增强**：`MarkItDown(llm_client=…, llm_model=…, llm_prompt=…)` 启用图片/PPT 字幕（OCR 插件扩展到 PDF/DOCX/XLSX）
- **HTML 渲染管线**：所有 HTML 出口经 `_CustomMarkdownify` —— 禁 `javascript:`/非 http 链接、懒加载图、大 `data:` URI 截断、`<u>` 直传、复选框 `[x]/[ ]`、`%HH` 编码保留
- **插件隔离**：`markitdown.plugin` entry-point 组加载；单插件异常仅 `warn()` 不传播
- **统一异常族**：`MarkItDownException` → `MissingDependencyException` / `UnsupportedFormatException` / `FileConversionException` / `FailedConversionAttempt`
- **URI 三态**：`http(s)` / `file:` / `data:` 全部支持，由 `_uri_utils.py` 解析

### 4. 安全边界（最小权限原则）

| 入口 | 适用场景 | 风险面 |
|---|---|---|
| `convert_local(path)` | 仅本地文件 | 文件系统 I/O |
| `convert_response(requests.Response)` | 调用方已构造好的响应 | 调用方可控的网络 |
| `convert_stream(stream)` | 任意可读流 | 调用方提供 |
| `convert_uri(uri)` / `convert(...)` | `http(s)` / `file:` / `data:` 全支持 | **最宽权限**，最后选择 |
| 库本身 | 不暴露 REST/HTTP 服务 | 无 web/REST 端点 |

### 5. 可选依赖一览

`pip install 'markitdown[<extra>]'` 的可选组：

| 组 | 触发的格式/能力 |
|---|---|
| `[pdf]` | PDF |
| `[docx]` | DOCX（mammoth） |
| `[xlsx]` | Excel 2007+ |
| `[xls]` | Excel 97-2003 |
| `[pptx]` | PowerPoint |
| `[outlook]` | Outlook .MSG |
| `[audio-transcription]` | 音频转写 |
| `[youtube-transcription]` | YouTube 转写 |
| `[az-doc-intel]` | Azure Document Intelligence（云） |
| `[az-content-understanding]` | Azure Content Understanding（云） |
| `[all]` | 上述全部 + LLM 客户端依赖 |

### 6. 扩展点（第三方接入）

- **entry-point 组**：`markitdown.plugin`
- **协议版本常量**：`__plugin_interface_version__ = 1`
- **入口函数**：`register_converters(markitdown: MarkItDown, **kwargs)`
- **可读实例属性**：`_llm_client` / `_llm_model`（LLM 增强插件使用）
- **云端插入时机**：检测到 endpoint 启用时 → 转换器栈顶插入
- **优先级约定**：数字越小越先排；同优先级保留插入顺序；`register_converter()` 走 `insert(0, …)` 故新注册者在桶内优先
- **参考模板**：`packages/markitdown-sample-plugin`
- **失败行为**：单插件异常 → `warn()` 并跳过，不影响主流程

### 7. 测试入口

- **向量测试**：`tests/_test_vectors.py` 定义 `GENERAL_TEST_VECTORS` / `DATA_URI_TEST_VECTORS`
- **CLI 向量**：`tests/test_cli_vectors.py`
- **库 API 向量**：`tests/test_module_vectors.py`
- **格式专项**：`tests/test_<format>.py`（与转换器一一对应）
- **杂项**：`tests/test_module_misc.py`（库基础设施、charset、URI、音频/ipynb 旧回归） + `tests/test_cli_misc.py`（CLI 行为）
- **跑测**：`hatch test`（全量）或 `hatch test tests/test_pdf.py`（单文件）
- **类型检查**：`hatch run types:check`（mypy，src + tests）

## 模块项目图（Module Project Diagram）

### 1. 仓库顶层结构（4 包 monorepo）

```
markitdown/                                ← Git 仓库根
├── CLAUDE.md / README.md / Dockerfile / LICENSE / .github/ ...
├── packages/
│   ├── markitdown/                        ← 核心库（PyPI: markitdown）
│   ├── markitdown-mcp/                    ← MCP 服务器包装（PyPI: markitdown-mcp）
│   ├── markitdown-ocr/                    ← OCR 增强插件（PyPI: markitdown-ocr）
│   └── markitdown-sample-plugin/          ← 第三方插件骨架（PyPI: markitdown-sample-plugin）
└── .venv/ .pre-commit-config.yaml .gitattributes ...
```

### 2. `packages/markitdown/src/markitdown/` 目录树

```
markitdown/
├── __init__.py              ← 公共 API 门面：导出 MarkItDown / DocumentConverter /
│                              StreamInfo / 异常族 / 优先级常量
├── __about__.py             ← __version__
├── __main__.py              ← CLI 入口 main()（argparse）
├── _markitdown.py           ← 核心编排器：MarkItDown 类 + _convert() +
│                              register_converter() + enable_builtins() + _load_plugins()
├── _base_converter.py       ← 抽象契约：DocumentConverter / DocumentConverterResult
├── _stream_info.py          ← StreamInfo frozen dataclass
├── _exceptions.py           ← 异常族：MarkItDownException /
│                              MissingDependencyException /
│                              UnsupportedFormatException /
│                              FileConversionException /
│                              FailedConversionAttempt +
│                              MISSING_DEPENDENCY_MESSAGE 模板
├── _uri_utils.py            ← parse_data_uri / file_uri_to_path
│
├── converter_utils/         ← 共享工具子包（不依赖转换器，仅供 Converter 调用）
│   ├── __init__.py
│   ├── _image.py            ← Pillow 包装 / EXIF 抽取
│   ├── _xlsx_images.py      ← openpyxl 图片读取
│   └── docx/                ← DOCX 专用工具（由 _docx_converter 独立调用）
│       ├── __init__.py
│       ├── _images.py       ← mammoth 图片处理
│       ├── pre_process.py   ← DOCX 预处理（数学公式/嵌入式样式/嵌入图片）
│       └── math/
│           ├── __init__.py
│           ├── omml.py      ← Office Math Markup Language → LaTeX 转换
│           └── latex_dict.py← LaTeX 替换字典（公式语法）
│
└── converters/              ← 格式转换器（20 个模块，对外仅暴露 __init__.py）
    ├── __init__.py          ← 统一 re-export 所有内置转换器类
    ├── _plain_text_converter.py           ← 兜底
    ├── _html_converter.py                 ← HtmlConverter + _CustomMarkdownify
    ├── _markdownify.py                    ← markdownify.MarkdownConverter 子类
    ├── _rss_converter.py
    ├── _wikipedia_converter.py
    ├── _youtube_converter.py
    ├── _ipynb_converter.py
    ├── _bing_serp_converter.py
    ├── _pdf_converter.py
    ├── _docx_converter.py                 ← 复用 _html_converter + converter_utils.docx
    ├── _xlsx_converter.py                 ← 复用 converter_utils._xlsx_images
    ├── _xls_converter.py
    ├── _csv_converter.py
    ├── _pptx_converter.py
    ├── _image_converter.py                ← 复用 _llm_caption + converter_utils._image
    ├── _llm_caption.py                    ← LLM 图片字幕辅助
    ├── _audio_converter.py                ← 复用 _transcribe_audio
    ├── _transcribe_audio.py
    ├── _outlook_msg_converter.py          ← 复用 _exiftool
    ├── _exiftool.py                       ← EXIF 元数据读取
    ├── _zip_converter.py                  ← 递归派发子条目
    ├── _epub_converter.py
    ├── _doc_intel_converter.py            ← Azure Document Intelligence（云）
    └── _cu_converter.py                   ← Azure Content Understanding（云）
```

### 3. 内部模块依赖图（`import` 关系）

```
                            ┌──────────────────────────────────────────┐
                            │  __init__.py 公共 API 门面                │
                            │  MarkItDown / DocumentConverter /         │
                            │  StreamInfo / 异常 / 优先级常量            │
                            └──────────────────────────────────────────┘
                                              │ 全部导出
                                              ▼
              ┌────────────────────┐    ┌──────────────────────────────────────┐
              │  __main__.py       │    │  _markitdown.py  MarkItDown 编排器   │◀──────┐
              │  CLI 入口 main()   │───▶│  convert* / _convert()                │       │ entry_points
              └────────────────────┘    │  enable_builtins()                    │       │ group="markitdown.plugin"
                                        │  register_converter(priority=...)     │       │ (importlib.metadata)
                                        │  _load_plugins()                      │       │
                                        └──────────────────────────────────────┘       │
                                              │            │            │                 │
                                              ▼            ▼            ▼                 │
                              ┌────────────────┐ ┌────────────────┐ ┌──────────────────┐  │
                              │ _base_converter│ │ _stream_info.py│ │ _exceptions.py  │  │
                              │ 抽象契约        │ │ frozen dataclass│ │ 5 个异常类 +   │  │
                              └────────────────┘ └────────────────┘ │ 模板             │  │
                                    ▲                                  └──────────────────┘  │
                                    │ 所有 20 个转换器继承                                       │
                                    │                                                         │
                                    │             ┌─────────────────┐                        │
                                    │             │ _uri_utils.py   │                        │
                                    │             │ parse_data_uri  │                        │
                                    │             │ file_uri_to_path│                        │
                                    │             └─────────────────┘                        │
                                    │                                                         │
                                    │  ┌────────────────────────────────────────────────────┐│
                                    │  │                  converters/  (20 模块)              ││
                                    │  └────────────────────────────────────────────────────┘│
                                    │                                                         │
                                    ├─ _plain_text_converter.py           [Generic 兜底]      │
                                    ├─ _html_converter.py ──────────────▶ _markdownify.py       │
                                    ├─ _rss_converter.py                                       │
                                    ├─ _wikipedia_converter.py                                │
                                    ├─ _youtube_converter.py                                   │
                                    ├─ _ipynb_converter.py                                     │
                                    ├─ _bing_serp_converter.py                                 │
                                    ├─ _pdf_converter.py                                       │
                                    ├─ _docx_converter.py ─┬────────────▶ _html_converter.py     │
                                    │                       ├────────────▶ converter_utils/docx/pre_process.py │
                                    │                       ├────────────▶ converter_utils/docx/_images.py │
                                    │                       ├────────────▶ converter_utils/docx/math/omml.py │
                                    │                       └────────────▶ converter_utils/docx/math/latex_dict.py │
                                    ├─ _xlsx_converter.py ─────────────▶ converter_utils/_xlsx_images.py │
                                    ├─ _xls_converter.py                                          │
                                    ├─ _csv_converter.py                                          │
                                    ├─ _pptx_converter.py                                         │
                                    ├─ _image_converter.py ─┬──────────▶ _llm_caption.py          │
                                    │                       └────────────▶ converter_utils/_image.py │
                                    ├─ _audio_converter.py ────────────▶ _transcribe_audio.py    │
                                    ├─ _outlook_msg_converter.py ──────▶ _exiftool.py            │
                                    ├─ _zip_converter.py              （递归派发其他 converter）│
                                    ├─ _epub_converter.py                                        │
                                    ├─ _doc_intel_converter.py           （Azure 云）              │
                                    └─ _cu_converter.py                  （Azure 云）              │
```

### 4. 跨包依赖图（`packages/` 之间）

```
    ┌─────────────────────────────┐
    │   markitdown-mcp/           │
    │   __main__.py ───import───▶ │
    │   MarkItDown                │
    │   FileConversionException   │
    │   UnsupportedFormatException│
    │   (以及 starlette / mcp /   │
    │    uvicorn / requests)      │
    └─────────────────────────────┘
                  │
                  │ pip install ./packages/markitdown ./packages/markitdown-mcp
                  ▼
    ┌─────────────────────────────┐         ┌─────────────────────────────┐
    │   markitdown/ 核心库        │ ◀───── │   markitdown-ocr/ 插件       │
    │   MarkItDown / Document…   │         │   _plugin.py                 │
    └─────────────────────────────┘  pip    │   register_converters()      │
           ▲             ▲                │   (priority=-1.0, 取代内置)   │
           │             │                └─────────────────────────────┘
           │             │                            │
           │             │                            │ pip install ./packages/markitdown
           │             │                            ▼ ./packages/markitdown-ocr
           │             │       ┌─────────────────────────────┐
           │             └───── │   markitdown-sample-plugin/  │
           │                     │   _plugin.py (RtfConverter 模板)│
           │                     │   __plugin_interface_version=1 │
           │                     └─────────────────────────────┘
           │
           │   所有插件包遵循 entry-point 协议：
           │     组名 markitdown.plugin
           │     常量 __plugin_interface_version__ = 1
           │     函数 register_converters(markitdown: MarkItDown, **kwargs)
```

### 5. 外部依赖矩阵（`pyproject.toml` features）

| extras 组 | 触发依赖 | 对应转换器 |
|---|---|---|
| `[pdf]` | pdfminer.six / pypdf | `_pdf_converter.py` |
| `[docx]` | mammoth | `_docx_converter.py` |
| `[xlsx]` | openpyxl | `_xlsx_converter.py` + `converter_utils._xlsx_images` |
| `[xls]` | xlrd | `_xls_converter.py` |
| `[pptx]` | python-pptx | `_pptx_converter.py` |
| `[outlook]` | olefile / compressed-rtf | `_outlook_msg_converter.py` |
| `[audio-transcription]` | speechrecognition / ffmpeg / exiftool | `_audio_converter.py` + `_transcribe_audio.py` |
| `[youtube-transcription]` | youtube-transcript-api | `_youtube_converter.py` |
| `[az-doc-intel]` | azure-ai-documentintelligence | `_doc_intel_converter.py` |
| `[az-content-understanding]` | azure-ai-contentunderstanding | `_cu_converter.py` |
| `[all]` | 上述全部 | — |
| **核心必选** | magika / charset_normalizer / requests / markdownify | 编排器 / `_markdownify.py` |

### 6. 入口 → 出口（包级别调用链）

```
CLI   → markitdown.__main__.main()
         └─ markitdown.MarkItDown  (构造)
            ├─ enable_builtins()   → 加载 packages/markitdown/src/markitdown/converters/__init__.py
            ├─ _load_plugins()     → importlib.metadata.entry_points(group="markitdown.plugin")
            │   命中 markitdown-ocr / markitdown-sample-plugin
            └─ .convert() / convert_uri() / ...
                 └─ _convert() → 调度所有已注册转换器

MCP   → markitdown-mcp.__main__  (MCPServer 注册 convert_to_markdown tool)
         └─ MarkItDown.convert_uri(uri)  (走与 CLI 相同的转换路径)

插件 → register_converters(markitdown, **kwargs)
         └─ markitdown.register_converter(MyConverter(), priority=...)
```

## 运行流程图（Runtime Flow）

### 1. 主流程（用户调用 → 输出 Markdown）

```
                ┌───────────────────────────────────────────┐
                │  用户入口                                   │
                ├────────────┬──────────────┬────────────────┤
                │ CLI        │ Python API   │ MCP            │
                │ markitdown  │ MarkItDown  │ convert_to_   │
                │ <file>     │ (...)        │ markdown(uri) │
                └─────┬──────┴──────┬───────┴───────┬────────┘
                      │             │               │
                      ▼             ▼               ▼
              ┌──────────────────────────────────────────────┐
              │  MarkItDown(...) 构造                       │
              │   ├─ self._requests_session（headers 协商） │
              │   ├─ self._magika = magika.Magika()        │
              │   ├─ llm_client / model / prompt 字段        │
              │   ├─ exiftool_path 字段（env / PATH）   │
              │   └─ style_map 字段                          │
              │                                              │
              │  enable_builtins(**kwargs)  ← 默认开启    │
              │  enable_plugins(**kwargs)    ← 显式开启   │
              └─────────────────────┬────────────────────┘
                                    │
                                    ▼
              ┌──────────────────────────────────────────────┐
              │  用户调用 .convert() / .convert_uri() /     │
              │  .convert_local() / .convert_stream() / │
              │  .convert_response()                         │
              │                                              │
              │  ├─ str → urlparse()                       │
              │  │   ├─ http(s)/file/data → convert_uri    │
              │  │   └─ 其他  → convert_local              │
              │  ├─ Path            → convert_local         │
              │  ├─ requests.Response → convert_response  │
              │  └─ BinaryIO-like → convert_stream        │
              └─────────────────────┬────────────────────┘
                                    │
                                    ▼
              ┌──────────────────────────────────────────────┐
              │  输入归一化                                    │
              │  ├─ 构造 base StreamInfo（mimetype/ext/      │
              │  │   charset/url/local_path/filename）       │
              │  ├─ _get_stream_info_guesses()               │
              │  │   ├─ mimetypes.guess_type 兜底            │
              │  │   ├─ magika.Magika() 内容嗅探            │
              │  │   └─ charset_normalizer 字符集识别        │
              │  └─ 多个 StreamInfo 候选（按置信度排序）     │
              │                                              │
              │  非可寻流 → BytesIO 重缓冲（规避 Windows     │
              │  pipe stdin 假性可寻）                       │
              └─────────────────────┬────────────────────┘
                                    │
                                    ▼
              ┌──────────────────────────────────────────────┐
              │  _convert(file_stream, stream_info_guesses) │
              │                                              │
              │  1. 快照 stream 位置 cur_pos = tell()       │
              │  2. sorted_registrations = sorted(           │
              │     self._converters, key=priority)         │
              │     （稳定排序；同优先级按注册先后）         │
              │ │
              │  3. 循环：                                    │
              │     for stream_info in guesses + [空]:       │
              │       for converter in sorted:                │
              │         ├─ converter.accepts(stream, info)?  │
              │         │   （NotImplementedError → 跳过）   │
              │         ├─ seek(cur_pos) 不变（断言）        │
              │         ├─ accepts → True → converter.convert│
              │         │   ├─ 异常 → FailedConversionAttempt│
              │         │   │     seek(cur_pos) 重置        │
              │         │   ├─ 返回 None → 继续下一转换器 │
              │         │   └─ 非 None → 胜出，跳出循环    │
              │         └─ 未命中 → 下一转换器               │
              └─────────────────────┬────────────────────┘
                                    │
                                    ▼
              ┌──────────────────────────────────────────────┐
              │  输出规范化                                    │
              │  res.text_content = "\n".join(              │
              │      line.rstrip() for line in              │
              │      re.split(r"\r?\n", res.text_content))  │
              │  res.text_content = re.sub(                  │
              │      r"\n{3,}", "\n\n", res.text_content)   │
              │  → DocumentConverterResult(markdown, title) │
              └─────────────────────┬────────────────────┘
                                    │
                                    ▼
                ┌───────────────────────────────────────────┐
                │  返回                                       │
                │  CLI   → 写文件或 stdout                │
                │  Python → str(result) 取 markdown          │
                │  MCP   → Tool 返回 markdown 字符串      │
                └───────────────────────────────────────────┘
```

### 2. 插件加载子流程（`_load_plugins`）

```
              MarkItDown(enable_plugins=True).enable_plugins()
                              │
                              ▼
              ┌──────────────────────────────────────┐
              │  _load_plugins()                     │
              │  （模块全局 _plugins 缓存；首次调用）│
              └──────────────────────┬───────────────┘
                                     │
                                     ▼
              ┌──────────────────────────────────────┐
              │  entry_points(group="markitdown.plugin") │
              │  （importlib.metadata）            │
              │  → 每个 entry_point:                │
              │     ├─ try: entry_point.load()       │
              │     │   返回 plugin 模块对象        │
              │     └─ except:                       │
              │         warn("Plugin '...' failed    │
              │              to load ... skipping:\n │
              │              {traceback}")           │
              │         → 跳过该插件，不影响主流程   │
              └──────────────────────┬───────────────┘
                                     │
                                     ▼
              ┌──────────────────────────────────────┐
              │  对每个 plugin 对象:                  │
              │  ├─ assert __plugin_interface_version__ == 1 │
              │  └─ try: plugin.register_converters( │
              │       │      markitdown_instance, **kwargs)  │
              │       │   内部调用 markitdown        │
              │       │   .register_converter(...)   │
              │       └─ except: warn(...) 跳过       │
              └──────────────────────────────────────┘
```

### 3. 转换器注册与优先级排序（`register_converter` + `sorted`）

```
              register_converter(converter, priority=PRIORITY_SPECIFIC_FILE_FORMAT)
                              │
                              ▼
              ┌──────────────────────────────────────────┐
              │  self._converters.insert(                │
              │      0,                                  │
              │      ConverterRegistration(              │
              │          converter=converter,            │
              │          priority=priority))             │
              │  → 列表头插入；最新注册者在最前          │
              └──────────────────────────────────────────┘
                              │
                              ▼
              每次 _convert() 都重新排序：
              ┌──────────────────────────────────────────┐
              │  sorted_registrations = sorted(          │
              │      self._converters,                    │
              │      key=lambda x: x.priority)           │
              │  → 优先级低（小）先跑；同优先级稳定保留   │
              └──────────────────────────────────────────┘

典型优先级分布：
  ┌────────────────────────────────────────────────────────────┐
  │  PRIORITY = -1.0   │ markitdown-ocr 插件（OCR 增强 PDF/DOCX/PPTX/XLSX）│
  ├────────────────────┼─────────────────────────────────────────┤
  │  PRIORITY = 0.0    │ DocumentIntelligenceConverter（云，启用时）│
  │                    │ ContentUnderstandingConverter（云，启用时）│
  │                    │ 大部分内置具体格式转换器                  │
  ├────────────────────┼─────────────────────────────────────────┤
  │  PRIORITY = 10.0   │ PlainTextConverter / HtmlConverter / ZipConverter│
  │                    │ （兜底 catch-all）                       │
  └────────────────────┴─────────────────────────────────────────┘
```

### 4. 云转换器激活子流程

```
              MarkItDown(docintel_endpoint="https://...") 或
              MarkItDown(cu_endpoint="https://...")
                              │
                              ▼
              enable_builtins() 在注册完内置转换器之后：
              ┌──────────────────────────────────────────┐
              │  if docintel_endpoint is not None:      │
              │      self.register_converter(            │
              │          DocumentIntelligenceConverter( │
              │              endpoint=docintel_endpoint, │
              │              credential=...,            │
              │              file_types=...,            │
              │              api_version=...))          │
              │  if cu_endpoint is not None:            │
              │      self.register_converter(            │
              │          ContentUnderstandingConverter( │
              │              endpoint=cu_endpoint,      │
              │              credential=...,            │
              │              analyzer_id=...,            │
              │              file_types=...))           │
              └──────────────────────────────────────────┘
                              │
                              ▼
              因 insert(0, ...) + 同优先级稳定排序，
              云转换器先于同优先级的内置转换器胜出
```

### 5. LLM 重试与降级子流程（`_llm_caption`）

```
              ImageConverter.convert() / PptxConverter.convert()
                              │
                              ▼
              ┌──────────────────────────────────────────┐
              │  self._llm_client is not None            │
              │  且接受图片/* / pptx 内容？            │
              │  └─ 否：纯本地处理（无 LLM 调用）       │
              └──────────────────────┬───────────────────┘
                                     │ 是
                                     ▼
              ┌──────────────────────────────────────────┐
              │  llm_caption(stream, info,              │
              │             client=self._llm_client,    │
              │             model=self._llm_model,     │
              │             prompt=self._llm_prompt)    │
              │                                          │
              │  ├─ 读 stream → base64 → data URI       │
              │  ├─ cur_pos = tell()                    │
              │  ├─ try: 编码 base64                    │
              │  │   except: 返回 None（降级）          │
              │  ├─ finally: seek(cur_pos)             │
              │  └─ client.chat.completions.create(...) │
              │      └─ 客户端自带 retry（外部配置）    │
              │          └─ 重试耗尽 → raise            │
              │              → ImageConverter 抛 →     │
              │              FailedConversionAttempt   │
              │              → 链上其他转换器尝试       │
              └──────────────────────────────────────────┘
```

### 6. ZIP/EPUB 递归分派子流程（`_parent_converters` 透传）

```
              ZipConverter.accepts() == True
                              │
                              ▼
              ┌──────────────────────────────────────────┐
              │  ZipConverter.convert()                  │
              │  └─ 打开 zip entries │
              │     for entry_name, entry_stream in zip: │
              │       ├─ 构造子 StreamInfo（filename/ext）│
              │       ├─ 子调用 _convert(                │
              │       │     file_stream=entry_stream,   │
              │       │     stream_info_guesses=...)    │
              │       │  （构造器收到 kwargs 中的       │
              │       │   _parent_converters=self._converters）│
              │       └─ 返回 DocumentConverterResult  │
              │           → 拼回 Markdown                │
              └──────────────────────────────────────────┘

  类似地适用于 EpubConverter。
```

### 7. 异常路径汇总

| 异常类型 | 触发条件 | 处理 |
|---|---|---|
| `MissingDependencyException` | 转换器内部 `try/except ImportError` | 上抛，携带 `MISSING_DEPENDENCY_MESSAGE.format(extras="pdf")` 等提示 |
| `FailedConversionAttempt` | 单个转换器 `try/except` 内异常 | 不上抛；收集到 `failed_attempts` 列表；继续下一转换器 |
| `FileConversionException` | 所有尝试过的转换器全部失败 | 抛出，携带 `attempts=failed_attempts` 列表 |
| `UnsupportedFormatException` | 没有任何转换器调用 `convert()`（全部 `accepts()` 返回 False） | 抛出 `"filetype is not supported"` |
| 插件加载异常 | `entry_point.load()` 或 `register_converters()` 抛异常 | `warn()` 并跳过；不影响主流程 |

### 8. 状态机视角

```
  ┌─────────────────────┐
  │  INIT               │ MarkItDown 构造
  └──────────┬──────────┘
             │ enable_builtins + enable_plugins
             ▼
  ┌─────────────────────┐
  │  READY              │ 转换器已注册；sort 缓存待用
  └──────────┬──────────┘
             │ .convert*()
             ▼
  ┌─────────────────────┐
  │  GUESS              │ 构造 StreamInfo 候选
  └──────────┬──────────┘
             │ guesses + [空]
             ▼
  ┌─────────────────────┐
  │  DISPATCH           │ 顺序 accepts → convert
  └──────────┬──────────┘
             │ 首个非 None 胜出
             ▼
  ┌─────────────────────┐
  │  NORMALIZE          │ rstrip + 折叠空行
  └──────────┬──────────┘
             │ 全部 failed
             ▼
  ┌─────────────────────────────────────────┐
  │  RAISE                                │
  │  FileConversionException /            │
  │  UnsupportedFormatException           │
  └─────────────────────────────────────────┘
```

## 使用示例（Usage Examples）

### 1. 环境激活

仓库内已包含 `.venv/` 虚拟环境。

```bash
# Windows (Git Bash / PowerShell / cmd)
.\.venv\Scripts\Activate

# Linux / macOS
source .venv/bin/activate
```

激活后终端提示符前缀变为 `(.venv)`，可直接使用 `markitdown` 与 `python` 命令。

### 2. CLI：单文件转换（最常用）

```bash
# 文件 → 文件输出
markitdown 文件路径 -o 输出.md

# 文件 → 报告输出（默认 stdout）
markitdown report.pdf

# 标准输入 → stdout（Windows 管道下走 BytesIO 重缓冲）
cat readme.md | markitdown
markitdown < readme.md

# 标准输入 → 文件
cat readme.md | markitdown -o out.md
```

### 3. CLI：URI 资源

```bash
# http(s) URL
markitdown https://example.com/article.html -o article.md

# file:/// URI
markitdown "file:///D:/docs/notes.txt"

# data: URI（base64 内嵌）
markitdown "data:text/markdown;base64,IkhlbGxvIg=="
```

### 4. CLI：显式提示（标准输入 / 无法自动识别时）

```bash
# 显式扩展名
markitdown -x .pdf < stdin

# 显式 MIME 类型
markitdown -m application/pdf < input.bin

# 显式字符集
markitdown -c utf-8 document.txt
```

### 5. CLI：Azure Document Intelligence（云）

```bash
# 方式 A：环境变量
export MARKITDOWN_DOCINTEL_ENDPOINT="https://xxx.cognitiveservices.azure.com/"
export MARKITDOWN_DOCINTEL_API_VERSION="2024-02-29"
markitdown -d report.pdf

# 方式 B：命令行参数
markitdown -e "https://xxx.cognitiveservices.azure.com/" report.pdf
```

### 6. CLI：Azure Content Understanding（云）

```bash
markitdown --use-cu \
  --cu-endpoint "https://xxx.cognitiveservices.azure.com/" \
  --cu-analyzer my-invoice-analyzer \
  invoice.pdf

# YAML front matter 形式提供自定义分析器（详见 CLI 帮助）
```

### 7. Python API：四个入口

```python
from markitdown import MarkItDown

md = MarkItDown()  # 内置转换器默认开启；插件需 enable_plugins=True

# 本地路径（最窄权限入口，推荐）
result = md.convert_local("report.pdf")

# 已构造的 HTTP 响应
import requests
resp = requests.get("https://example.com/doc.html")
result = md.convert_response(resp)

# 任意可读流
with open("notes.docx", "rb") as f:
    result = md.convert_stream(f)

# URI（http/https/file/data）
result = md.convert_uri("https://example.com/page.html")
result = md.convert_uri("file:///D:/docs/notes.txt")

print(result.markdown)  # 主体
print(result.title)      # 可选标题（None if 提取不到）
print(str(result))      # == result.markdown
```

### 8. Python API：启用插件

```python
from markitdown import MarkItDown

# 须先先 `pip install markitdown-ocr` 等带 label 的 entry-point 插件
md = MarkItDown(enable_plugins=True)
result = md.convert("scanned.pdf")  # 自动走 OCR 增强链
```

### 9. Python API：LLM 图像字幕

```python
from openai import OpenAI
from markitdown import MarkItDown

client = OpenAI()
md = MarkItDown(
    llm_client=client,
    llm_model="gpt-4o",
    llm_prompt="Write a detailed caption for this image.",
)
result = md.convert("photo.jpg")           # PPTX 图片同样适用
```

### 10. Python API：注册自定义转换器

```python
from markitdown import MarkItDown, DocumentConverter, DocumentConverterResult

class MyConverter(DocumentConverter):
    def accepts(self, file_stream, stream_info, **kwargs):
        return (stream_info.extension or "").lower() == ".myext"

    def convert(self, file_stream, stream_info, **kwargs):
        text = file_stream.read().decode("utf-8")
        return DocumentConverterResult(markdown=f"# Custom\n\n{text}")

md = MarkItDown()
md.register_converter(MyConverter(), priority=0.0)  # 数字越小越先跑
result = md.convert("file.myext")
```

### 11. MCP 服务器

```bash
# 安装
pip install ./packages/markitdown ./packages/markitdown-mcp

# 启动（stdio 传输）
markitdown-mcp

# 启用插件（环境变量）
MARKITDOWN_ENABLE_PLUGINS=true markitdown-mcp
```

MCP 客户端侧调用：`convert_to_markdown(uri: str) -> str`，支持 `http:` / `https:` / `file:` / `data:` URI。

### 12. 异常 / 退出码

CLI 异常 → 退出码非 0：

| 异常 | 触发 | 排查方向 |
|---|---|---|
| `UnsupportedFormatException` | 所有 `accepts()` 返回 False | 检查扩展名 / MIME 提示 |
| `FileConversionException` | 所有尝试过的转换器都失败 | 携带 `attempts` 列表，逐项排查 |
| `MissingDependencyException` | 转换器内部 `ImportError` | 按提示执行 `pip install 'markitdown[<extra>]'` |

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
