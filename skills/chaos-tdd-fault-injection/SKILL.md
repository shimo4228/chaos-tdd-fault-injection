---
name: chaos-tdd-fault-injection
description: Chaos-TDD for LLM agent pipelines — deterministic fault injection at existing seams, where the fault-injection test states the desired guarded behavior FIRST and the minimal production guard lands in the same PR. Use when hardening a pipeline against LLM/external-I/O fault families (truncated output, timeouts, 429s, wrong-shaped-but-parseable JSON, flapping backends), when building a fault catalog from operational bug history, when adding property-based fuzz over LLM output shapes, or when reviewing whether a pipeline's failure paths abstain with reason codes instead of degrading silently. NOT for infra-level chaos on distributed systems (chaostoolkit/toxiproxy are the right altitude there, not here), NOT for single-bug regression pinning after the fact (this skill is that discipline's front-loaded, catalog-driven counterpart), and NOT for audit-log schema design (a sibling concern this skill only asserts against).
compatibility: Developed and tested on Claude Code; portable to other Agent Skills-compatible agents. Code examples are Python (pytest / hypothesis / responses); the pattern is language-agnostic.
user-invocable: true
origin: shimo4228
---

# Chaos-TDD Fault Injection

**Seeded fault schedules as test-first specification.**

LLM agent pipelines fail in one recurring family: an LLM or external I/O
returns something unexpected — truncated text, a 429, valid JSON of the wrong
shape — and the pipeline degrades *silently*. These bugs get diagnosed after
they burn a production run, one at a time. Chaos-TDD front-loads them:

> operational fault catalog → fault-injection test asserting the DESIRED
> guarded behavior (RED) → minimal production guard in the same PR (GREEN).

This is not Netflix-style random production chaos. A single-process agent
needs no daemon to inject faults into itself, its stored data is often
irreplaceable, and randomness makes CI flaky. Everything here is
deterministic, in-process, and pytest-native.

## 1. Build the fault catalog from operational history

Do not invent faults from imagination — mine them from incidents you already
had:

1. List the incidents. Classify each by *where the unexpected response
   entered*: LLM text? HTTP status? embedding rows? the parse layer?
2. Diff that list against existing test coverage — grep for the **fault**,
   not the feature (`finish_reason`, `429`, `JSONDecodeError`, ...).
3. The uncovered remainder is the catalog. A typical first catalog:
   - mid-generation read-timeout (only connection-refused was covered)
   - direct HTTP faults on the embedding endpoint (429 / timeout / ragged
     rows / fewer rows than requested)
   - syntactically-valid JSON violating the structured-output schema
     (wrong top-level type, wrong key, non-string list items)
   - 429 from the LLM endpoint itself (rate-limit handling often exists
     only for the app-facing API client, not the model backend)
   - flapping backends: alternating success/failure sequences beyond the
     single-recovery case
4. For each fault, write the *asserted behavior* column **before writing any
   test**. If you cannot say what the pipeline SHOULD do, that is a design
   gap — the usual answer is "abstain with a machine-readable reason code" —
   and the chaos test is about to become the specification for a small
   production change. That is the point, not a detour.

## 2. Choose injection seams — never add a production hook

Inject only at seams that already exist:

- **Interface seam** — a test-side fake implementing the same
  Protocol/interface the production backend does. Faults are a
  `schedule: list[str]` consumed one entry per call — an explicit list, or
  materialized from a seed at construction (`from_seed(seed, n)`) so a
  failing run can print the schedule verbatim for replay:

  ```python
  OK, NONE, EMPTY, EXC_TIMEOUT, TRUNCATED, SHAPE_VIOLATION = (
      "ok", "none", "empty", "exc_timeout", "truncated", "shape_violation")

  @dataclass
  class ChaosBackend:            # implements the same Protocol as production
      schedule: list[str] = field(default_factory=list)
      calls: list[dict] = field(default_factory=list)

      def generate(self, prompt, **kw):
          idx = len(self.calls); self.calls.append({"prompt": prompt, **kw})
          fault = self.schedule[idx] if idx < len(self.schedule) else OK
          if fault == NONE: return None
          if fault == EXC_TIMEOUT: raise ReadTimeout("chaos: read timed out")
          if fault == TRUNCATED: return Result(text=ok_json(idx), finish_reason="length")
          if fault == SHAPE_VIOLATION: return Result(text='["wrong", "shape"]')
          return Result(text=ok_json(idx))   # valid, deterministic per index
  ```

- **HTTP layer** — a request-mocking library (Python: `responses`)
  registering fault responses on the URL the code *actually resolves* (read
  the same env var / config the production code reads). Status codes,
  `body=ReadTimeout(...)` exception injection, and malformed payloads — all
  without touching production code.

Cover **both sides of a dispatch branch**: if generation routes to either an
injected backend or a built-in HTTP path, point some faults at each. Keep
the injector in `tests/`, not `src/` — test-only code must not enter
production import paths — and give the injector its own self-tests
(interface compliance, schedule determinism, OK-payload validity).

## 3. Determinism discipline

- Every schedule explicit or seed-derived; inspectable before the run.
- Property-based fuzz runs derandomized. With hypothesis:
  `settings.register_profile("ci", derandomize=True, database=None,
  deadline=None)` in `conftest.py` — and, the part `database=None` does NOT
  cover, relocate `HYPOTHESIS_STORAGE_DIRECTORY` into the test sandbox
  tempdir *before importing hypothesis*, or its constants/unicode caches
  recreate an untracked `.hypothesis/` in the repo.
- **No real sleeps.** A latency fault is the *observable outcome* of a
  timeout — inject `ReadTimeout` instead of sleeping past `timeout=`. A
  `no_sleep` fixture that monkeypatches `time.sleep` to raise doubles as the
  fail-fast assertion (e.g. "a 429 from a local daemon must not honor
  Retry-After — the circuit breaker is the recovery mechanism").
- Pin every known failure shape with `@example(...)` on top of `@given(...)`
  — the property explores; the examples are permanent regressions.
- Verify determinism mechanically: run the chaos test files **twice** in the
  verify step; identical output is a PASS criterion.

## 4. Steady-state assertion channels

Assert on observable channels, not implementation internals:

- **Per-call telemetry** — e.g. a JSONL log with an `outcome` field plus a
  sparse `error_kind` (`timeout` / `connection` / `http_<status>` /
  `bad_json` / `backend_exception`), present on failure rows only. If your
  telemetry cannot distinguish the fault kinds you are injecting, **that
  indistinguishability is itself a finding** — the field exists to be
  asserted against, and its absence means the audit trail cannot answer
  "which fault actually occurred?" offline.
- **Reason-coded log tokens** — machine-greppable `reason=<code>` lines
  (`llm_none` / `empty_render` / `shape_violation` / `embed_failed`),
  tallied per reason in a summary line so a backend fault burst is
  distinguishable from a parse-layer problem and from a clean low-yield run.
- Internal counters (circuit-breaker state) only for the state machine under
  test itself, nowhere else.

Schedule-driven pipeline tests compute the expected tally *from the schedule
alone* (a fixed fault→reason mapping). Corollary: exclude circuit-opening
schedules from exact-count properties (a `trips_circuit()` filter) — once a
breaker opens, later OK entries short-circuit and the prediction breaks.
Keep one separate never-crashes property that allows the breaker to open.

## 5. The TDD contract

The chaos test asserts the DESIRED behavior, which usually does not exist
yet — that is the point. The RED for "wrong-shaped JSON must abstain with a
reason code" is often an ImportError on the reason-code constants. Two rules
keep this honest:

- Inverting an existing test's expectation (e.g. "top-level array returns
  `[]`" → "classifies `shape_violation`") is part of RED — record the
  inversion in a decision record, never as a silent test diff.
- Keep legitimate degradations. A bullet-list fallback for *genuinely
  non-JSON* bodies can stay — tagged (`parse=bullet_fallback`) for
  observability. Chaos-TDD tightens the silent paths; it does not delete
  graceful ones.

## 6. Failure shapes this pattern reliably catches

Found on the pattern's first deployment, all previously silent:

- **`{"items": [123]}`** — a `str(item)` promotion let non-string
  schema violations through as garbage entries; an
  `all(isinstance(item, str) ...)` gate plus an `@example` pin closed it.
- **JSON `null` body** — `json.loads("null")` returns `None`, which a
  `parsed = None`-as-failure sentinel silently routes to the text-scanning
  fallback; a distinct `_PARSE_FAILED = object()` sentinel separates "not
  JSON" from "JSON null" (a shape violation).
- **Telemetry indistinguishability** — 429 vs timeout vs connection-refused
  all collapsed into one `outcome="error"`; surfaced by writing the
  fault-kind assertions before the field existed.

## When to reach for this skill

- A new pipeline touches an LLM or external API → build its fault column
  before shipping, in the same PR.
- A production incident closes → after pinning the regression, ask "which
  catalog family was this, and which OTHER pipelines share the seam?"
- Reviewing code that parses LLM output → the shape questions: what happens
  on valid-but-wrong-shape? Is the failure distinguishable from legitimate
  emptiness? Is there a reason code?
