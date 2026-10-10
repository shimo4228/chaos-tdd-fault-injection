# chaos-tdd-fault-injection

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/chaos-tdd-fault-injection)

An [Agent Skill](https://agentskills.io/specification) for **chaos-TDD on LLM agent pipelines**: test-first fault injection for developers whose single-process agent or pipeline calls an LLM or other external services. One family of failures recurs in such pipelines: the LLM or an external API returns something unexpected (truncated text, a 429, valid JSON of the wrong shape) and the pipeline degrades silently.

You build a fault catalog (the faults from your own incident history that no test covers yet), inject each fault deterministically at seams that already exist (an interface or HTTP layer where a test fake can replace the LLM backend), and let the fault-injection test state the desired guarded behavior *first*. The minimal production guard then lands in the same PR, usually as a reason-coded abstain: the pipeline declines to produce output and logs a machine-readable reason such as `reason=shape_violation`.

The skill is one Markdown file, [`skills/chaos-tdd-fault-injection/SKILL.md`](skills/chaos-tdd-fault-injection/SKILL.md). Its examples are Python (pytest, hypothesis, responses); the pattern itself is language-agnostic. If your backend has no interface to fake, the HTTP layer still works: a request-mocking library such as `responses` registers fault responses on the URL the code calls, without touching production code. It runs nothing on its own, sends nothing off your machine, and needs no API key.

## Install

```bash
git clone https://github.com/shimo4228/chaos-tdd-fault-injection.git
mkdir -p ~/.claude/skills
cp -r chaos-tdd-fault-injection/skills/chaos-tdd-fault-injection ~/.claude/skills/
```

Claude Code loads it when you work on a pipeline's LLM or external I/O failure paths or build a fault catalog, or when you call `/chaos-tdd-fault-injection`. Other Agent Skills-compatible agents take the same folder in their own skills directory.

## What a chaos test looks like

The skill ships no code: you write a small `ChaosBackend` in `tests/`, adapting the full fake in [SKILL.md](skills/chaos-tdd-fault-injection/SKILL.md) §2. In a test it replaces the production LLM backend and plays one fault per call from a schedule:

```python
backend = ChaosBackend(schedule=["ok", "shape_violation", "exc_timeout"])
```

The second call returns wrong-shaped JSON (`["wrong", "shape"]`) and the third raises `ReadTimeout`. Written before any guard exists, the test asserts a `reason=shape_violation` log line for the second call and `error_kind` `timeout` in telemetry for the third. It fails first (RED, the failing step of the TDD cycle, often an ImportError on reason-code constants not yet defined); the guard that makes it pass (GREEN) ships in the same PR.

On its first deployment, on a memory-distillation pipeline in the author's Contemplative Agent project, this pattern surfaced three previously-silent failures: `{"items": [123]}` passing as garbage because of a `str(item)` promotion (closed by an `all(isinstance(item, str) ...)` gate), a JSON `null` body routed to the text fallback because `None` also meant "parse failed", and 429, timeout and connection-refused all collapsing into one telemetry outcome. See [SKILL.md](skills/chaos-tdd-fault-injection/SKILL.md) §6.

## The Five Disciplines

| # | Discipline | One-line contract |
|---|------------|-------------------|
| 1 | Fault catalog from operational history | Mine incidents, diff against test coverage; the uncovered remainder is the catalog |
| 2 | Existing seams only | Interface-seam fake (schedule-driven `ChaosBackend`) + HTTP-layer mock; never a production hook |
| 3 | Determinism | Seeded schedules, property fuzz run derandomized (hypothesis `derandomize=True`, so every run generates the same inputs) with known failure shapes pinned as `@example`s, latency = injected timeout exception, run-twice-identical as a PASS criterion |
| 4 | Steady state on observable channels | Telemetry `outcome`/`error_kind` + greppable `reason=<code>` log tokens, not implementation internals |
| 5 | TDD contract | The chaos test asserts behavior that does not exist yet; the guard ships in the same PR |

## Why not chaostoolkit / toxiproxy?

Wrong altitude for a single-process local agent. [chaostoolkit](https://github.com/chaostoolkit/chaostoolkit) and [toxiproxy](https://github.com/Shopify/toxiproxy) inject infrastructure-level faults between services; the LLM and external-API faults this skill targets are injectable in-process at the pipeline's own interface and HTTP seams, deterministically, with zero extra infrastructure.

## Origin & curation

This repository is curated by hand: it generalizes the operational variant running inside the [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) project, where the pattern was piloted (decision record [ADR-0077](https://github.com/shimo4228/contemplative-agent/blob/main/docs/adr/0077-chaos-tdd-fault-injection.md)). There is intentionally no sync script, so changes to that variant reach this repository only when the author carries them over.

## More from the author

- **[Fault Injection TDD Found 3 Silent Failures in My LLM Agent](https://dev.to/shimo4228/fault-injection-tdd-found-3-silent-failures-in-my-llm-agent-3mld)** ([日本語](https://zenn.dev/shimo4228/articles/chaos-tdd-fault-injection)): the first application of this skill end to end, from incident history to fault catalog to RED and GREEN, and the three silent failures that about 1,800 existing tests had missed.
- **[agent-observability-patterns](https://github.com/shimo4228/agent-observability-patterns)**: three skills for watching an agent before you intervene (replayable audit logs, read-only instruments, shadow-mode validation); they record the channels this skill asserts against.
- **[code-and-llm-collaboration](https://github.com/shimo4228/code-and-llm-collaboration)**: four patterns for layering deterministic code and LLM calls in one pipeline, from the guard that validates LLM output before it persists to the code loop that owns termination.
- **[Contemplative Agent](https://github.com/shimo4228/contemplative-agent)**: the long-running local-LLM agent where this pattern was piloted; it proposes changes to its own constitution and skills, and a person approves each one.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with the long-running practice lines, their DOIs, and the tools for Claude Code.

## License

[MIT](LICENSE)

<details>
<summary>For tools and AI assistants</summary>

chaos-tdd-fault-injection is an Agent Skill (one Markdown instruction file in the open Agent Skills format) that teaches a coding agent to harden LLM agent pipelines with deterministic, test-first fault injection, for developers running single-process agents or pipelines that call an LLM or external APIs.

It exists because one family of failures recurs in these pipelines: an LLM or external I/O returns something unexpected (truncated text, a 429, valid JSON of the wrong shape) and the pipeline degrades silently, so each bug is diagnosed only after it has burned a production run. Chaos-TDD front-loads them: operational fault catalog, then a fault-injection test asserting the desired guarded behavior (RED), then the minimal production guard in the same PR (GREEN). It is not random chaos in production: a single-process agent needs no daemon to inject faults into itself, its stored data is often irreplaceable, and randomness makes CI flaky.

Canonical facts: MIT license; a single `SKILL.md`; examples in Python with pytest, hypothesis and responses, while the pattern is language-agnostic; maintained by one author (@shimo4228). Status: curated by hand, intentionally with no sync script, so the two copies are not kept in step automatically; it generalizes the variant running in the author's Contemplative Agent project, recorded there as ADR-0077. Requirements: Claude Code or another agent that reads Agent Skills; applying it needs a test suite and an interface (or HTTP layer) where a test fake can replace the LLM backend; no API key.

Example: a test-side `ChaosBackend` implements the production backend's interface and consumes one fault per call from a schedule such as `ok`, `truncated`, `exc_timeout`, `shape_violation`, explicit or materialized from a seed so a failing run prints it for replay. The test asserts on per-call telemetry (`outcome` plus `error_kind` such as `timeout` or `http_429`) and on `reason=<code>` log tokens, not on implementation internals; internal counters such as circuit-breaker state are asserted only for the state machine under test. On its first deployment the pattern surfaced three previously silent failures: `{"items": [123]}` passing as garbage because of a `str(item)` promotion, a JSON `null` body routed to the text fallback because `None` also meant "parse failed", and 429, timeout and connection-refused all collapsing into one `outcome="error"`.

Links: [skills/chaos-tdd-fault-injection/SKILL.md](skills/chaos-tdd-fault-injection/SKILL.md) is the skill itself; [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are the machine-readable summary and reference. The paired decision record is [Contemplative Agent ADR-0077](https://github.com/shimo4228/contemplative-agent/blob/main/docs/adr/0077-chaos-tdd-fault-injection.md); Contemplative Agent's concept DOI is [10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118).

</details>
