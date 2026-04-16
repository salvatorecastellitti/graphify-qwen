# graphify

AI coding assistant skill that turns any folder of code, docs, papers, images, or videos into a queryable knowledge graph.

## Project Overview

graphify is a Python library and AI coding assistant skill (Claude Code, Codex, OpenCode, Cursor, Gemini CLI, GitHub Copilot CLI, VS Code Copilot Chat, Aider, OpenClaw, Factory Droid, Trae, Hermes, Kiro, Qwen Code, Google Antigravity) that builds knowledge graphs from mixed corpora. It extracts structural information from code via tree-sitter AST parsing, transcribes video/audio locally with faster-whisper, and uses LLM subagents to extract concepts from docs, papers, and images. The result is a NetworkX graph with community detection, exported as interactive HTML, queryable JSON, and plain-language reports.

**Key capabilities:**
- 25 languages supported via tree-sitter AST (Python, JS/TS, Go, Rust, Java, C/C++, Ruby, C#, Kotlin, Scala, PHP, Swift, Lua, Zig, PowerShell, Elixir, Objective-C, Julia, Verilog, Vue, Svelte, Dart)
- Multimodal: code, markdown, PDFs, images, videos, audio files
- Leiden community detection for graph clustering
- SHA256 cache for incremental rebuilds
- Git hooks for auto-rebuild on commit/branch switch
- MCP server for structured graph queries

## Building and Running

### Requirements
- Python 3.10+
- One AI coding assistant: Claude Code, Codex, OpenCode, Cursor, Gemini CLI, GitHub Copilot CLI, VS Code Copilot Chat, Aider, OpenClaw, Factory Droid, Trae, Hermes, Kiro, Qwen Code, or Google Antigravity

### Installation

```bash
# Install package (PyPI name is graphifyy)
pip install graphifyy

# Install skill for your platform
graphify install                    # Claude Code (default)
graphify install --platform codex   # Codex
graphify install --platform opencode  # OpenCode
graphify install --platform qwen-code  # Qwen Code
graphify cursor install             # Cursor
graphify gemini install             # Gemini CLI
graphify copilot install            # GitHub Copilot CLI
graphify vscode install             # VS Code Copilot Chat
# ... see README for full platform list
```

### Optional Dependencies

```bash
# Video/audio transcription (local, via faster-whisper + yt-dlp)
pip install 'graphifyy[video]'

# MCP server support
pip install 'graphifyy[mcp]'

# PDF extraction
pip install 'graphifyy[pdf]'

# Obsidian export
pip install 'graphifyy[leiden]'

# All extras
pip install 'graphifyy[all]'
```

### Usage

```bash
# Build graph from current directory
/graphify .

# Build with options
/graphify ./src --mode deep       # More aggressive inference
/graphify ./src --update          # Incremental update (changed files only)
/graphify ./src --watch           # Auto-sync on file changes
/graphify ./src --wiki            # Generate wiki articles

# Add content
/graphify add https://arxiv.org/abs/1706.03762   # Fetch paper
/graphify add <video-url>         # Download + transcribe

# Query graph (no AI assistant needed)
graphify query "show the auth flow"
graphify path "DigestAuth" "Response"
graphify explain "SwinTransformer"

# MCP server
python -m graphify.serve graphify-out/graph.json
```

### Testing

```bash
# Run all tests
pytest tests/ -q

# Run specific test module
pytest tests/test_extract.py -v
```

Test fixtures are in `tests/fixtures/`. All tests are pure unit tests with no network calls.

## Development Conventions

### Code Style
- Python 3.10+ type hints throughout
- Functions documented with Google-style docstrings
- Modules organized by pipeline stage (detect → extract → build → cluster → analyze → report → export)

### Architecture

The pipeline is modular with each stage as a separate module:

| Module | Function | Purpose |
|--------|----------|---------|
| `detect.py` | `collect_files(root)` | Filter directory to relevant files |
| `extract.py` | `extract(path)` | AST parsing for code files |
| `build.py` | `build_graph(extractions)` | Construct NetworkX graph |
| `cluster.py` | `cluster(G)` | Leiden community detection |
| `analyze.py` | `analyze(G)` | Find god nodes, surprising connections |
| `report.py` | `render_report(G, analysis)` | Generate GRAPH_REPORT.md |
| `export.py` | `export(G, out_dir)` | Write JSON, HTML, SVG, Obsidian |
| `ingest.py` | `ingest(url)` | Fetch external content |
| `cache.py` | `check_semantic_cache()` | SHA256 caching |
| `serve.py` | `start_server()` | MCP stdio server |
| `watch.py` | `watch(root)` | Filesystem watcher |

### Extraction Schema

All extractors return:

```json
{
  "nodes": [
    {"id": "unique_id", "label": "name", "source_file": "path", "source_location": "L42"}
  ],
  "edges": [
    {"source": "id_a", "target": "id_b", "relation": "calls", "confidence": "EXTRACTED"}
  ]
}
```

Confidence levels:
- `EXTRACTED`: Directly in source (imports, calls)
- `INFERRED`: Reasonable deduction with confidence score
- `AMBIGUOUS`: Uncertain, flagged for review

### Adding a Language Extractor

1. Add `extract_<lang>(path: Path) -> dict` in `extract.py`
2. Register suffix in `extract()` dispatch and `collect_files()`
3. Add to `CODE_EXTENSIONS` in `detect.py` and `_WATCHED_EXTENSIONS` in `watch.py`
4. Add tree-sitter package to `pyproject.toml`
5. Add fixture to `tests/fixtures/` and tests to `tests/test_languages.py`

### Security

All external input passes through `security.py`:
- URLs validated (http/https only, no private IPs)
- Downloads capped at 50MB
- Graph paths validated to stay within `graphify-out/`
- Labels sanitized (no control chars, 256 char limit, HTML-escaped)

See `SECURITY.md` for full threat model.

## Output Structure

```
graphify-out/
├── graph.html          # Interactive visualization
├── GRAPH_REPORT.md     # God nodes, connections, suggested questions
├── graph.json          # Persistent graph data
├── cache/              # SHA256 extraction cache
├── transcripts/        # Video/audio transcripts (if used)
└── wiki/               # Wiki articles (if --wiki flag)
```

## Worked Examples

See `worked/` directory for example outputs:
- `karpathy-repos/` - 52 files, 71.5x token reduction
- `mixed-corpus/` - Code + Transformer paper
- `httpx/` - Synthetic Python library
- `example/` - Basic example

## Key Files

| File | Purpose |
|------|---------|
| `pyproject.toml` | Package config, dependencies, extras |
| `ARCHITECTURE.md` | Pipeline overview, module responsibilities |
| `CHANGELOG.md` | Version history with detailed changes |
| `SECURITY.md` | Threat model, security practices |
| `graphify/__main__.py` | CLI entry point, platform installers |
| `graphify/extract.py` | Tree-sitter AST extraction (3200+ lines) |
| `graphify/skill*.md` | Platform-specific skill definitions |

## Token Efficiency

graphify achieves **71.5x fewer tokens per query** vs reading raw files on mixed corpora. The first run builds the graph (costs tokens). Subsequent queries read the compact `graph.json` instead of raw files. SHA256 cache means re-runs only process changed files.

## Privacy

- Code: processed locally via tree-sitter (no network)
- Video/audio: transcribed locally with faster-whisper
- Docs/papers/images: sent to LLM API during extraction (your API key)
- No telemetry, usage tracking, or analytics
