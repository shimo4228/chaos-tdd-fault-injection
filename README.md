# chaos-tdd-fault-injection

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/chaos-tdd-fault-injection)

An [Agent Skill](https://agentskills.io/specification) for **chaos-TDD on LLM agent pipelines**: build a fault catalog from your own operational bug history, inject each fault deterministically at seams that already exist, and let the fault-injection test state the desired guarded behavior *first* — so the minimal production guard (usually a reason-coded abstain) lands in the same PR. Deterministic by construction: seeded fault schedules, derandomized property fuzz, no real sleeps, no production hooks.

## Install

### Claude Code

```bash
# Copy into your global skills directory
cp -r skills/chaos-tdd-fault-injection ~/.claude/skills/chaos-tdd-fault-injection
```

### SkillsMP

```bash
/skills add shimo4228/chaos-tdd-fault-injection
```

## The Five Disciplines

| # | Discipline | One-line contract |
|---|------------|-------------------|
| 1 | Fault catalog from operational history | Mine incidents, diff against test coverage; the uncovered remainder is the catalog |
| 2 | Existing seams only | Interface-seam fake (schedule-driven `ChaosBackend`) + HTTP-layer mock; never a production hook |
| 3 | Determinism | Seeded schedules, derandomized fuzz with `@example` pins, latency = injected timeout exception, run-twice-identical as a PASS criterion |
| 4 | Steady state on observable channels | Telemetry `outcome`/`error_kind` + greppable `reason=<code>` log tokens, not implementation internals |
| 5 | TDD contract | The chaos test asserts behavior that does not exist yet; the guard ships in the same PR |

On its first deployment this pattern surfaced three previously-silent failures — non-string schema violations promoted via `str()`, a JSON `null` body mis-routed by a `None` sentinel, and fault kinds indistinguishable in telemetry. See [SKILL.md](skills/chaos-tdd-fault-injection/SKILL.md) §6.

## Why not chaostoolkit / toxiproxy?

Wrong altitude for a single-process local agent. Those tools inject faults between distributed services via daemons and proxies; an agent pipeline's faults are all injectable in-process at its own Protocol and HTTP seams — deterministically, with zero extra infrastructure.

## Origin & curation

This repository is manually curated: it generalizes the operational variant running inside the [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) project, where the pattern was piloted on a memory-distillation pipeline ([ADR-0077](https://github.com/shimo4228/contemplative-agent/blob/main/docs/adr/0077-chaos-tdd-fault-injection.md)). There is intentionally no automated sync script.

## License

[MIT](LICENSE)
