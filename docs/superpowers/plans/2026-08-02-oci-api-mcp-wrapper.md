# OCI API MCP Wrapper Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a loopback-only MCP server that gives local agentic systems complete OCI API access through allowlisted local OCI profiles, with typed discovery, generic signed requests, tiered approvals, redacted audit records, and Free Tier verification.

**Architecture:** Use the official Python MCP SDK with stateless FastMCP Streamable HTTP mounted in Starlette. Route canonical operations through a shared validation, policy, approval, execution, redaction, and audit pipeline; prefer OCI SDK clients and fall back to signed OCI REST requests. Persist approvals and work requests in owner-only SQLite and append tamper-evident audit events to JSON Lines.

**Tech Stack:** Python 3.13, uv, `mcp`, Starlette, Uvicorn, OCI Python SDK, Pydantic, httpx, pytest, Ruff, mypy, SQLite, systemd user services.

## Global Constraints

- Bind MCP and health endpoints only to `127.0.0.1`; proposed port `8800` must be checked and reserved through local coordination before first bind.
- Accept only explicitly configured profile aliases loaded from the operator-selected OCI config file; never expose raw profile values.
- Every state-changing operation requires separate approval. Ordinary mutations require supervisor authority; destructive, IAM, security-sensitive, or potentially costly operations require the local human approval CLI.
- Bind single-use, expiring approval to the canonical profile, tenancy, region, realm, endpoint, method, safe headers, query, body, and policy class digest.
- Reject arbitrary destinations, redirects, caller-controlled signing headers, caller-supplied profile paths, unsafe payloads, and oversized requests.
- Never commit credentials, private keys, authority secrets, runtime databases, audit logs, tenancy exports, or generated live-test data.
- Live mutations require a separately approved Free Tier-safe test action, unique test tags, reconciliation, and verified cleanup.
- Use test-driven development and run the focused failing test before each implementation step.
- Each task must invoke Superpowers with its task context, dependencies, interfaces, and acceptance criteria before implementation.

## Planned file map

```text
pyproject.toml                         Packaging, dependencies, commands, lint/type/test configuration
.gitignore                            Credential, state, audit, build, cache, and environment exclusions
src/oci_api_mcp/config.py             TOML settings and secure path/permission validation
src/oci_api_mcp/models.py             Shared request, result, policy, approval, and error models
src/oci_api_mcp/profiles.py           Safe OCI profile alias loader
src/oci_api_mcp/endpoints.py          OCI hostname, DNS, redirect, header, and size validation
src/oci_api_mcp/catalog.py            SDK/CLI operation catalog and search
src/oci_api_mcp/policy.py             Conservative operation risk classification
src/oci_api_mcp/digests.py            Canonical request serialization and SHA-256 digest
src/oci_api_mcp/state.py              SQLite approval and work-request repository
src/oci_api_mcp/audit.py              Redacted hash-linked JSONL audit writer and verifier
src/oci_api_mcp/redaction.py          Recursive request, response, log, and exception redaction
src/oci_api_mcp/adapters/sdk.py        Typed OCI SDK execution adapter
src/oci_api_mcp/adapters/rest.py       Generic signed OCI REST adapter
src/oci_api_mcp/executor.py            Shared validation/policy/approval/execution pipeline
src/oci_api_mcp/work_requests.py       OCI asynchronous operation tracking
src/oci_api_mcp/tools.py               Stable MCP tool schemas and handlers
src/oci_api_mcp/app.py                 FastMCP plus Starlette health application
src/oci_api_mcp/cli.py                 Serve, profile, human approval, audit, and catalog commands
tests/                                 Mirrored unit, contract, security, MCP, and live tests
assets/config.example.toml             Non-secret operator configuration example
assets/systemd/oci-api-mcp-wrapper.service  User-service template
scripts/install_user_service.py        Idempotent user-service installer
plugin/oci-api-mcp/                    Reusable Codex plugin, skill, and narrow hooks
docs/                                  Installation, operation, security, testing, and API reference
```

---

### Task 1: Bootstrap the tested Python package

**Files:**
- Create: `pyproject.toml`, `.gitignore`, `src/oci_api_mcp/__init__.py`, `tests/test_package.py`
- Generate: `uv.lock`

**Interfaces:**
- Produces: importable package `oci_api_mcp`; console entry point `oci-api-mcp = oci_api_mcp.cli:main` reserved for Task 8.

- [ ] **Step 1: Write the failing package test**

```python
def test_package_exposes_version() -> None:
    import oci_api_mcp
    assert oci_api_mcp.__version__ == "0.1.0"
```

- [ ] **Step 2: Verify the test fails**

Run: `python3 -m pytest tests/test_package.py -q`
Expected: FAIL because pytest or `oci_api_mcp` is unavailable.

- [ ] **Step 3: Create `pyproject.toml`, package metadata, and exclusions**

```toml
[project]
name = "oci-api-mcp-wrapper"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = ["mcp", "oci", "starlette", "uvicorn", "pydantic>=2", "httpx"]

[project.optional-dependencies]
dev = ["pytest", "pytest-asyncio", "pytest-cov", "ruff", "mypy"]

[project.scripts]
oci-api-mcp = "oci_api_mcp.cli:main"
```

Set `__version__ = "0.1.0"`. Ignore `.venv/`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `*.sqlite*`, `*.jsonl`, `.oci/`, `config.local.toml`, authority secrets, coverage output, and build artifacts.

- [ ] **Step 4: Install and verify**

Run: `python3 -m pip install --user uv && uv sync --extra dev && uv run pytest tests/test_package.py -q && uv run ruff check . && uv run mypy src`
Expected: all checks PASS and `uv.lock` exists. Package installation is local project bootstrap; coordinate before changing any shared package installation.

- [ ] **Step 5: Commit**

```bash
git add pyproject.toml uv.lock .gitignore src/oci_api_mcp/__init__.py tests/test_package.py
git commit -m "Bootstrap tested OCI MCP package"
```

### Task 2: Define canonical models and secure configuration

**Files:**
- Create: `src/oci_api_mcp/models.py`, `src/oci_api_mcp/config.py`, `assets/config.example.toml`
- Test: `tests/test_config.py`, `tests/test_models.py`

**Interfaces:**
- Produces: `RiskClass`, `AuthorityClass`, `OperationRequest`, `CanonicalRequest`, `OperationResult`, `McpError`; `load_settings(path: Path) -> Settings`.

- [ ] **Step 1: Write failing tests** for rejecting wildcard hosts, unknown keys, relative state paths, permissive authority-secret files, and caller-provided OCI config paths; verify example config loads with alias `free-tier` and default port `8800`.

```python
def test_settings_reject_wildcard_bind(tmp_path: Path) -> None:
    path = write_config(tmp_path, host="0.0.0.0")
    with pytest.raises(SettingsError, match="loopback"):
        load_settings(path)
```

- [ ] **Step 2: Run focused tests and confirm failure**

Run: `uv run pytest tests/test_config.py tests/test_models.py -q`
Expected: FAIL because models and loader do not exist.

- [ ] **Step 3: Implement strict Pydantic models and TOML loader**

```python
class RiskClass(StrEnum):
    READ_ONLY = "read_only"
    ORDINARY_MUTATION = "ordinary_mutation"
    HIGH_RISK = "high_risk"

class OperationRequest(BaseModel):
    profile: str | None = None
    method: Literal["GET", "HEAD", "POST", "PUT", "PATCH", "DELETE"]
    target_uri: AnyHttpUrl
    query: dict[str, str | list[str]] = Field(default_factory=dict)
    headers: dict[str, str] = Field(default_factory=dict)
    body: JsonValue | None = None
```

Use `tomllib`, forbid extra keys, resolve only operator-configured absolute paths, enforce loopback host, and validate owner-only authority-secret permissions.

- [ ] **Step 4: Run focused and static checks**

Run: `uv run pytest tests/test_config.py tests/test_models.py -q && uv run mypy src/oci_api_mcp/config.py src/oci_api_mcp/models.py`
Expected: PASS.

- [ ] **Step 5: Commit** with `git commit -m "Define secure OCI MCP configuration"`.

### Task 3: Implement profiles, endpoint validation, redaction, and policy

**Files:**
- Create: `src/oci_api_mcp/profiles.py`, `endpoints.py`, `redaction.py`, `policy.py`
- Test: `tests/test_profiles.py`, `test_endpoints.py`, `test_redaction.py`, `test_policy.py`

**Interfaces:**
- Produces: `ProfileRegistry.list_aliases()`, `ProfileRegistry.load(alias) -> dict[str, str]`; `validate_target(uri, resolved_addresses) -> ValidatedTarget`; `redact(value) -> JsonValue`; `classify(request, catalog_hint) -> PolicyDecision`.

- [ ] **Step 1: Write failing security tests** covering profile content non-disclosure, unknown aliases, unsafe headers, non-OCI and private destinations, redirect escape, credential-shaped values, GET/HEAD reads, all other methods as mutations, and IAM/delete/cost keywords as high risk.

```python
@pytest.mark.parametrize("uri", ["http://127.0.0.1/x", "https://example.com/x", "https://169.254.169.254/opc/v2/"])
def test_target_rejects_non_oci_destinations(uri: str) -> None:
    with pytest.raises(TargetDenied):
        validate_target(uri, ["127.0.0.1"])
```

- [ ] **Step 2: Confirm failures** with `uv run pytest tests/test_profiles.py tests/test_endpoints.py tests/test_redaction.py tests/test_policy.py -q`.
- [ ] **Step 3: Implement minimal modules** using `oci.config.from_file()` only after alias lookup, recognized OCI realm suffixes, connection-time IP validation, redirects disabled, case-insensitive forbidden headers, recursive key/value redaction, and conservative risk rules.
- [ ] **Step 4: Run focused tests and security lint** with `uv run pytest tests/test_profiles.py tests/test_endpoints.py tests/test_redaction.py tests/test_policy.py -q && uv run ruff check src tests`.
- [ ] **Step 5: Commit** with `git commit -m "Enforce OCI request security boundaries"`.

### Task 4: Build canonical digests, approval state, and tamper-evident audit

**Files:**
- Create: `src/oci_api_mcp/digests.py`, `state.py`, `audit.py`
- Test: `tests/test_digests.py`, `test_state.py`, `test_audit.py`

**Interfaces:**
- Produces: `canonicalize(request, profile_context, decision) -> CanonicalRequest`; `digest_request(canonical) -> str`; `StateStore.prepare(...)`, `approve(...)`, `consume(...)`; `AuditWriter.append(event)`, `verify() -> AuditVerification`.

- [ ] **Step 1: Write failing tests** proving key-order-independent digests, digest changes for any execution field, WAL mode, single-use approvals, expiry, authority mismatch, originator/supervisor separation, and broken audit-chain detection.

```python
def test_approval_is_single_use(store: StateStore, pending: PendingOperation) -> None:
    store.approve(pending.id, AuthorityClass.SUPERVISOR, "supervisor-a")
    assert store.consume(pending.id, pending.digest).status == "consumed"
    with pytest.raises(ApprovalConsumed):
        store.consume(pending.id, pending.digest)
```

- [ ] **Step 2: Confirm failures** with `uv run pytest tests/test_digests.py tests/test_state.py tests/test_audit.py -q`.
- [ ] **Step 3: Implement canonical JSON serialization, SHA-256 digests, transactional SQLite state, and hash-linked JSONL audit**. Open files with owner-only modes and call redaction before serialization.
- [ ] **Step 4: Run focused tests plus concurrent consume test**; expected one consumer succeeds and all others fail closed.
- [ ] **Step 5: Commit** with `git commit -m "Add immutable approvals and audit chain"`.

### Task 5: Build the OCI catalog and execution adapters

**Files:**
- Create: `src/oci_api_mcp/catalog.py`, `src/oci_api_mcp/adapters/__init__.py`, `sdk.py`, `rest.py`
- Test: `tests/test_catalog.py`, `tests/test_sdk_adapter.py`, `tests/test_rest_adapter.py`, `tests/fixtures/catalog.json`

**Interfaces:**
- Produces: `OperationCatalog.search(query, service)`, `get(operation_id)`; `SdkAdapter.supports(canonical)`, `execute(canonical)`; `RestAdapter.execute(canonical)` returning `AdapterResult`.

- [ ] **Step 1: Write failing contract tests** for catalog search, deterministic operation IDs, SDK preference, OCI request signing, disabled redirects, JSON bodies, pagination headers, idempotency headers, timeout mapping, and refusal to send before connection-time address validation.
- [ ] **Step 2: Confirm failures** with `uv run pytest tests/test_catalog.py tests/test_sdk_adapter.py tests/test_rest_adapter.py -q`.
- [ ] **Step 3: Implement catalog generation from installed OCI SDK/CLI metadata and a checked-in sanitized fixture**. Implement the SDK adapter through an explicit registry, not dynamic `eval`; implement REST signing with OCI SDK signers and an httpx transport hook that validates the connected destination.

```python
class ExecutionAdapter(Protocol):
    async def execute(self, request: CanonicalRequest) -> AdapterResult: ...
```

- [ ] **Step 4: Run focused tests with all outbound traffic mocked** and verify no test reaches the network.
- [ ] **Step 5: Commit** with `git commit -m "Add OCI catalog and execution adapters"`.

### Task 6: Implement the shared executor and work-request tracker

**Files:**
- Create: `src/oci_api_mcp/executor.py`, `work_requests.py`
- Test: `tests/test_executor.py`, `tests/test_work_requests.py`

**Interfaces:**
- Produces: `Executor.execute_read(request, originator)`, `prepare_mutation(...)`, `execute_approved(pending_id, authority_context)`; `WorkRequestTracker.register(adapter_result)`, `status(local_id)`.

- [ ] **Step 1: Write failing orchestration tests** proving reads execute directly, mutations never execute during prepare, SDK fallback routing, policy revalidation, exact-digest consumption, idempotency handling, bounded read retries, no blind mutation retry, `outcome_unknown`, and asynchronous handle return.
- [ ] **Step 2: Confirm failures** with `uv run pytest tests/test_executor.py tests/test_work_requests.py -q`.
- [ ] **Step 3: Implement the executor as the only path to adapters** and map all exceptions to stable `McpError` codes with OCI request IDs and safe next actions.
- [ ] **Step 4: Run focused tests and branch coverage** with `uv run pytest tests/test_executor.py tests/test_work_requests.py --cov=oci_api_mcp.executor --cov=oci_api_mcp.work_requests --cov-branch`.
- [ ] **Step 5: Commit** with `git commit -m "Orchestrate safe OCI operation execution"`.

### Task 7: Expose the compact MCP tool surface and health app

**Files:**
- Create: `src/oci_api_mcp/tools.py`, `app.py`
- Test: `tests/test_tools.py`, `tests/test_mcp_protocol.py`, `tests/test_health.py`

**Interfaces:**
- Produces MCP tools: `list_profiles`, `search_operations`, `get_operation`, `execute_read`, `prepare_mutation`, `get_pending_operation`, `cancel_pending_operation`, `approve_ordinary_mutation`, `execute_approved`, `get_work_request`, `continue_page`, `verify_audit`; ASGI routes `/mcp` and `/health`.

- [ ] **Step 1: Write failing MCP client tests** for initialize, tools/list, harmless tool calls, structured errors, originator identity, supervisor credential separation, absent high-risk approval tool, response schemas, concurrency limits, and health redaction.
- [ ] **Step 2: Confirm failures** with `uv run pytest tests/test_tools.py tests/test_mcp_protocol.py tests/test_health.py -q`.
- [ ] **Step 3: Implement FastMCP with stateless JSON Streamable HTTP and mount it in Starlette**. Add correlation IDs, request-size and concurrency limits, and a loopback-host startup assertion.

```python
mcp = FastMCP("OCI API", stateless_http=True, json_response=True, streamable_http_path="/")
app = Starlette(routes=[Mount("/mcp", app=mcp.streamable_http_app()), Route("/health", health)])
```

- [ ] **Step 4: Run protocol tests and start a temporary loopback server**; verify initialize, list-tools, call-tool, health, and `ss -ltnp` show no wildcard listener.
- [ ] **Step 5: Commit** with `git commit -m "Expose loopback OCI MCP tools"`.

### Task 8: Add operator CLI and user-service packaging

**Files:**
- Create: `src/oci_api_mcp/cli.py`, `assets/systemd/oci-api-mcp-wrapper.service`, `scripts/install_user_service.py`
- Test: `tests/test_cli.py`, `tests/test_service_installer.py`

**Interfaces:**
- Produces commands: `serve`, `profiles`, `approve`, `deny`, `pending`, `audit-verify`, `catalog-refresh`, `doctor`; installer `install --config PATH --port 8800`.

- [ ] **Step 1: Write failing CLI tests** proving human approval displays safe canonical details and digest, requires interactive confirmation, rejects redirected stdin/non-TTY by default, and never prints secrets. Test idempotent service rendering with explicit absolute paths and loopback arguments.
- [ ] **Step 2: Confirm failures** with `uv run pytest tests/test_cli.py tests/test_service_installer.py -q`.
- [ ] **Step 3: Implement commands and installer**. The installer must check/reserve the port through local coordination before activation and must not start or enable the service without the user-approved execution stage.
- [ ] **Step 4: Run focused tests and `oci-api-mcp doctor` against fixture configuration**; expected safe readiness report with no credential values.
- [ ] **Step 5: Commit** with `git commit -m "Package OCI MCP operator workflows"`.

### Task 9: Add reusable Codex skill, hooks, and documentation

**Files:**
- Create: `plugin/oci-api-mcp/.codex-plugin/plugin.json`, `plugin/oci-api-mcp/skills/oci-api-mcp/SKILL.md`, `plugin/oci-api-mcp/skills/oci-api-mcp/agents/openai.yaml`, `plugin/oci-api-mcp/skills/oci-api-mcp/references/server.md`, `plugin/oci-api-mcp/hooks/hooks.json`, `plugin/oci-api-mcp/hooks/oci-api-mcp-preflight.py`, `plugin/oci-api-mcp/hooks/oci-api-mcp-evidence-guard.py`
- Create: `README.md`, `docs/installation.md`, `docs/operations.md`, `docs/security.md`, `docs/testing.md`, `docs/mcp-tools.md`
- Test: `tests/test_plugin.py`, `tests/test_docs_commands.py`

**Interfaces:**
- Produces portable skill and narrow hooks that route OCI tasks to server `oci_api` and require brief tool-result evidence without blocking unrelated prompts.

- [ ] **Step 1: Invoke `codex-scope-manager`, `skill-creator`, and Superpowers writing-skills before authoring durable behavior.** Confirm target scope and managed refresh path; do not install unreviewed hooks.
- [ ] **Step 2: Write failing manifest and hook fixture tests** for OCI prompt match, unrelated prompt bypass, unavailable server remediation, secret-free output, and accepted MCP evidence.
- [ ] **Step 3: Implement plugin artifacts and documentation** with exact install, configure, register, initialize, test, audit, upgrade, and uninstall commands.
- [ ] **Step 4: Validate plugin schema, run hook fixtures, secret scan, and documentation command checks**. After installation in a non-managed home, explicitly instruct the user to review and trust through `/hooks`.
- [ ] **Step 5: Commit** with `git commit -m "Document and package OCI MCP integration"`.

### Task 10: Complete automated and live OCI acceptance

**Files:**
- Create: `tests/security/test_boundary_matrix.py`, `tests/contract/test_oci_metadata.py`, `tests/live/test_free_tier_reads.py`, `tests/live/test_free_tier_mutation.py`, `scripts/smoke_mcp.py`, `docs/acceptance-results.md`
- Modify: `README.md`, `docs/testing.md`

**Interfaces:**
- Consumes all prior interfaces.
- Produces reproducible acceptance evidence and a live-test cleanup ledger containing only non-secret resource identifiers.

- [ ] **Step 1: Add failing acceptance tests** for every completion gate and mark live mutations with an explicit opt-in marker plus approved test manifest.
- [ ] **Step 2: Run all non-live checks**:

```bash
uv run ruff format --check .
uv run ruff check .
uv run mypy src
uv run pytest -m "not live" --cov=oci_api_mcp --cov-branch
uv run python scripts/smoke_mcp.py
```

Expected: all pass; smoke output confirms initialize, tool listing, call-tool, health, and loopback listener.

- [ ] **Step 3: Run read-only live acceptance with the designated Free Tier profile**. Verify authentication, tenancy/region/compartment reads, broad available-service reads, and one generic signed request. Record only redacted evidence.
- [ ] **Step 4: Stop for separate explicit approval before any live mutation.** After approval, preflight quota/cost/region, create uniquely tagged Free Tier-safe resource, reconcile, delete, verify absence, and record cleanup. Do not substitute a different mutation if blocked.
- [ ] **Step 5: Audit tracked files and active Git history** for credentials, OCI config material, private paths, agent metadata, runtime data, logs, and personal exports. Use a public-safe Git identity before publication.
- [ ] **Step 6: Commit** with `git commit -m "Verify OCI MCP acceptance criteria"`.

## Final verification and handoff

- [ ] Run the complete non-live suite again after any rebase or integration.
- [ ] Verify live MCP readiness using an SDK client, not only process or listener state.
- [ ] Verify runtime files are outside Git and owner-only.
- [ ] Verify the service is loopback-only and no gateway/firewall route was added.
- [ ] Update `docs/acceptance-results.md` with exact commands, outcomes, known Free Tier coverage limits, and any separately approved live mutation evidence.
- [ ] Use Superpowers verification-before-completion and requesting-code-review before declaring implementation complete.
