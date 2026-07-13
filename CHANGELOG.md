# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-07-13

Initial release as a standalone skill repository.

### Added

- `skills/chaos-tdd-fault-injection/SKILL.md` — chaos-TDD for LLM agent pipelines: fault catalog mined from operational history, deterministic injection at existing seams (interface-seam `ChaosBackend` fake + HTTP-layer mock), determinism discipline (seeded schedules, derandomized fuzz with `@example` pins, no real sleeps), steady-state assertion on telemetry/reason-code channels, and the test-first contract where the minimal guard ships in the same PR.

### Notes

- This repository is manually curated: it generalizes the operational variant running inside the [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) project, piloted there on a memory-distillation pipeline (the paired decision record [ADR-0077](https://github.com/shimo4228/contemplative-agent/blob/main/docs/adr/0077-chaos-tdd-fault-injection.md) stays in the research repository). There is intentionally no automated sync script.
