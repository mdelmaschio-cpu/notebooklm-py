# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**IMPORTANT:** Follow documentation rules in [CONTRIBUTING.md](CONTRIBUTING.md) - especially the file creation and naming conventions.

## Project Overview

`notebooklm-py` is an unofficial Python client for Google NotebookLM that uses undocumented RPC APIs. The library enables programmatic automation of NotebookLM features including notebook management, source integration, AI querying, and studio artifact generation (podcasts, videos, quizzes, etc.).

**Critical constraint**: This uses Google's internal `batchexecute` RPC protocol with obfuscated method IDs that Google can change at any time. All RPC method IDs in `src/notebooklm/rpc/types.py` are undocumented and subject to breakage.

- **Version**: 0.3.4 (Beta)
- **Python**: 3.10–3.14
- **License**: MIT
- **Core deps**: `httpx`, `click`, `rich`; optional: `playwright` (browser auth)

## Development Commands

```bash
# Create/recreate venv with uv (recommended - relocatable venvs)
uv venv .venv
uv pip install -e ".[all]"
playwright install chromium

# Activate virtual environment
source .venv/bin/activate

# Run all tests (excluding e2e by default)
pytest

# Run with coverage
pytest --cov

# Run e2e tests (requires authentication)
pytest tests/e2e -m e2e

# CLI testing
notebooklm --help
```

## Pre-Commit Checks (REQUIRED before committing)

**IMPORTANT:** Always run these checks before committing to avoid CI failures:

```bash
# Format code with ruff
ruff format src/ tests/

# Check for linting issues
ruff check src/ tests/

# Type checking with mypy
mypy src/notebooklm --ignore-missing-imports

# Run tests
pytest
```

Or use this one-liner:
```bash
ruff format src/ tests/ && ruff check src/ tests/ && mypy src/notebooklm --ignore-missing-imports && pytest
```

**Ruff config**: `line-length = 100`, `target-version = "py310"`. All imports are auto-sorted via isort rules.

## Architecture

### Layered Design

```
CLI Layer (cli/)
    ↓
Client Layer (client.py, _*.py APIs)
    ↓
Core Layer (_core.py)
    ↓
RPC Layer (rpc/)
```

1. **RPC Layer** (`src/notebooklm/rpc/`):
   - `types.py`: All RPC method IDs and enums (source of truth). Key enums: `RPCMethod` (40+ method IDs), `ArtifactTypeCode` (AUDIO=1, REPORT=2, VIDEO=3, QUIZ=4, MIND_MAP=5, INFOGRAPHIC=7, SLIDE_DECK=8, DATA_TABLE=9), `ArtifactStatus` (PROCESSING=1, PENDING=2, COMPLETED=3, FAILED=4)
   - `encoder.py`: Request encoding — triple-nested array format
   - `decoder.py`: Response parsing, anti-XSSI stripping, error code mapping

2. **Core Layer** (`src/notebooklm/_core.py`):
   - HTTP client management (httpx, 30s default / 10s connect timeout)
   - RPC call abstraction with auth error detection and auto-refresh
   - Conversation cache (FIFO, max 100 entries)
   - Source ID extraction utilities

3. **Client Layer** (`src/notebooklm/client.py`, `_*.py`):
   - `NotebookLMClient`: Main async client with namespaced sub-APIs
   - `_notebooks.py`, `_sources.py`, `_artifacts.py`, etc.: Domain APIs

4. **CLI Layer** (`src/notebooklm/cli/`):
   - Modular Click commands, grouped and top-level
   - Rich output formatting via `helpers.py`

### Key Files

| File | Purpose |
|------|---------|
| `client.py` | Main `NotebookLMClient` class |
| `_core.py` | HTTP and RPC infrastructure |
| `_notebooks.py` | `client.notebooks` API |
| `_sources.py` | `client.sources` API |
| `_artifacts.py` | `client.artifacts` API |
| `_chat.py` | `client.chat` API |
| `_research.py` | `client.research` API |
| `_notes.py` | `client.notes` API |
| `_settings.py` | `client.settings` API (output language) |
| `_sharing.py` | `client.sharing` API (notebook visibility) |
| `_logging.py` | Logging config (`NOTEBOOKLM_LOG_LEVEL`, `NOTEBOOKLM_DEBUG_RPC`) |
| `_url_utils.py` | URL normalization helpers |
| `_version_check.py` | Version update check logic |
| `exceptions.py` | Full exception hierarchy (public API) |
| `paths.py` | Config path resolution (`NOTEBOOKLM_HOME`) |
| `auth.py` | Authentication — cookies, CSRF, regional domains |
| `types.py` | Public-facing dataclasses and enums |
| `rpc/types.py` | RPC method IDs (source of truth — obfuscated) |
| `cli/` | CLI command modules |

### Repository Structure

```
src/notebooklm/
├── __init__.py          # Public exports
├── __main__.py          # python -m notebooklm entrypoint
├── notebooklm_cli.py    # CLI entry point (main())
├── client.py            # NotebookLMClient
├── auth.py              # Authentication
├── types.py             # Public dataclasses and enums
├── exceptions.py        # Exception hierarchy
├── paths.py             # Config path resolution
├── _core.py             # Core infrastructure
├── _notebooks.py        # NotebooksAPI
├── _sources.py          # SourcesAPI
├── _artifacts.py        # ArtifactsAPI
├── _chat.py             # ChatAPI
├── _research.py         # ResearchAPI
├── _notes.py            # NotesAPI
├── _settings.py         # SettingsAPI
├── _sharing.py          # SharingAPI
├── _logging.py          # Logging configuration
├── _url_utils.py        # URL normalization
├── _version_check.py    # Version update checker
├── rpc/                 # RPC protocol layer
│   ├── __init__.py      # Re-exports
│   ├── types.py         # Method IDs and enums
│   ├── encoder.py       # Request encoding
│   └── decoder.py       # Response parsing
└── cli/                 # CLI implementation
    ├── __init__.py
    ├── helpers.py       # Shared utilities and Rich formatting
    ├── options.py       # Reusable Click options
    ├── error_handler.py # Error formatting
    ├── grouped.py       # Grouped command organization
    ├── language.py      # Language/locale handling
    ├── session.py       # login, use, status, clear
    ├── notebook.py      # list, create, delete, rename
    ├── source.py        # source add, list, delete, refresh
    ├── artifact.py      # artifact commands
    ├── generate.py      # generate audio, video, quiz, etc.
    ├── download.py      # download commands
    ├── download_helpers.py # Download utilities
    ├── chat.py          # ask, configure, history
    ├── note.py          # note commands
    ├── research.py      # research queries
    ├── share.py         # share notebook
    ├── skill.py         # agent skill install/show
    ├── agent.py         # agent commands
    └── agent_templates.py # Agent template management
```

## API Patterns

### Client Usage

```python
# Correct pattern - uses namespaced APIs
async with await NotebookLMClient.from_storage() as client:
    notebooks = await client.notebooks.list()
    await client.sources.add_url(nb_id, url)
    result = await client.chat.ask(nb_id, question)
    status = await client.artifacts.generate_audio(nb_id)
    await client.sharing.share(nb_id, access_level)
    await client.settings.set_output_language("en")
```

### CLI Structure

Commands are organized as:
- **Top-level**: `login`, `use`, `status`, `clear`, `list`, `create`, `ask`
- **Grouped**: `source add`, `artifact list`, `generate audio`, `download video`, `note create`, `share`, `research`
- **Agent integration**: `agent show <target>` (show instructions for "claude" or "codex" agents), `skill install`, `skill show`

### Exception Handling

All exceptions inherit from `NotebookLMError`. Full hierarchy:
- `ValidationError` — Invalid arguments/config
- `ConfigurationError` — Missing/invalid configuration
- `NetworkError` — HTTP-level failures
- `RPCTimeoutError` — Request timeout
- `RPCError` → `DecodingError`, `UnknownRPCMethodError`, `AuthError`, `RateLimitError`, `ServerError`, `ClientError`
- `NotebookError` → `NotebookNotFoundError`
- `SourceError` → `SourceAddError`, `SourceProcessingError`, `SourceTimeoutError`, `SourceNotFoundError`
- `ArtifactError` → `ArtifactNotFoundError`, `ArtifactNotReadyError`, `ArtifactParseError`, `ArtifactDownloadError`
- `ChatError`

```python
from notebooklm import NotebookLMError, AuthError, RateLimitError

try:
    await client.notebooks.list()
except AuthError:
    # Re-authenticate
except RateLimitError:
    # Backoff and retry
except NotebookLMError as e:
    # Catch-all
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `NOTEBOOKLM_HOME` | Base directory for config files (default: `~/.notebooklm`) |
| `NOTEBOOKLM_AUTH_JSON` | Auth tokens as JSON string (CI/CD use) |
| `NOTEBOOKLM_LOG_LEVEL` | `DEBUG`, `INFO`, `WARNING` (default), `ERROR` |
| `NOTEBOOKLM_DEBUG_RPC` | Set to `1` for RPC-level debug logging (legacy alias for DEBUG) |

## Testing Strategy

- **Unit tests** (`tests/unit/`): Test encoding/decoding, no network
- **Integration tests** (`tests/integration/`): Mock HTTP via VCR.py cassettes
- **CLI integration tests** (`tests/integration/cli_vcr/`): VCR-recorded CLI tests
- **E2E tests** (`tests/e2e/`): Real API, require auth, marked `@pytest.mark.e2e`

**Coverage**: 90% minimum enforced. **Timeout**: 60s per test (global default).

### E2E Test Status

- ✅ Notebook operations (list, create, rename, delete)
- ✅ Source operations (add URL/text/YouTube, rename)
- ✅ Download operations (audio, video, infographic, slides)
- ⚠️ Artifact generation may fail due to rate limiting

## Common Pitfalls

1. **RPC method IDs change**: Check network traffic and update `rpc/types.py`
2. **Nested list structures**: Params are position-sensitive. Check existing implementations.
3. **Source ID nesting**: Different methods need `[id]`, `[[id]]`, `[[[id]]]`, or `[[[[id]]]]`
4. **CSRF tokens expire**: Use `client.refresh_auth()` or re-run `notebooklm login`
5. **Rate limiting**: Add delays between bulk operations
6. **Auth in CI/CD**: Use `NOTEBOOKLM_AUTH_JSON` env var instead of storage file

## Agent Integration

The library includes first-class support for LLM agent workflows:

- **`SKILL.md`** (repo root): Claude Code skill definition — install into a project with `notebooklm skill install claude`
- **`AGENTS.md`** (repo root): Instructions for Gemini/Codex agent environments — shown via `notebooklm agent show codex`
- **Skill install paths**: `~/.claude/skills/notebooklm/SKILL.md` (user scope) or `.claude/skills/notebooklm/SKILL.md` (project scope)
- **Target environments**: "claude" (→ SKILL.md) and "codex" (→ AGENTS.md/CODEX.md)

## Documentation

All docs use lowercase-kebab naming in `docs/`:
- `docs/cli-reference.md` — CLI commands reference
- `docs/python-api.md` — Python API reference
- `docs/configuration.md` — Storage, env vars, settings
- `docs/troubleshooting.md` — Known issues and workarounds
- `docs/development.md` — Architecture, testing, debugging
- `docs/rpc-development.md` — RPC capture and reverse engineering
- `docs/rpc-reference.md` — RPC payload structures
- `docs/stability.md` — API versioning policy
- `docs/releasing.md` — Release checklist
- `docs/examples/` — Runnable example scripts

Scratch/investigation work goes in `docs/scratch/` with `YYYY-MM-DD-<context>.md` naming (see CONTRIBUTING.md).

## When to Suggest CLI vs API

- **CLI**: Quick tasks, shell scripts, LLM agent automation
- **Python API**: Application integration, complex workflows, async operations

## Pull Request Workflow (REQUIRED)

After creating a PR, you MUST monitor and address feedback.

This repository uses the **GitHub MCP server tools** (prefixed `mcp__github__`) for all GitHub interactions. Do **not** rely on the `gh` CLI unless it is confirmed available in the session.

### 1. Monitor CI Status

Use `mcp__github__pull_request_read` to check PR status and CI checks. Repeat until all checks pass. If any fail, investigate and fix.

### 2. Check for Review Comments

Use `mcp__github__pull_request_read` or `mcp__github__add_reply_to_pull_request_comment` to read and respond to review comments.

### 3. Address Feedback

For each review comment (especially from `gemini-code-assist`):
1. Read and understand the feedback
2. Make the suggested fix if it improves the code
3. Commit with a descriptive message referencing the feedback
4. Push and re-check CI
5. Reply to the review thread confirming the fix using `mcp__github__add_reply_to_pull_request_comment`

### 4. Verify Final State

Use `mcp__github__pull_request_read` to confirm the PR is mergeable.

**Important**: Do NOT consider a PR complete until:
- All CI checks pass
- All review comments are addressed
- PR is in a mergeable/clean state
