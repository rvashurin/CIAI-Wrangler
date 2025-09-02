# AGENTS.md

Guidelines for agents and contributors working on CIAI‑Wrangler in the Codex CLI environment. This document focuses on safe, minimal, and effective iteration for a subprocess‑driven CLI that interacts with SLURM.

## Overview
- Purpose: Orchestrate and monitor submission of batch job scripts to an HPC SLURM cluster with a bounded concurrency queue.
- Entry point: `./ciai_wrangler` (Python 3 shebang script; no `.py` extension).
- Key constraint: In development environments without SLURM, avoid executing real cluster commands.

## Repository Layout
- `ciai_wrangler`: Python CLI. Main components:
  - `parse_args()` → CLI argument parsing (`config_file` YAML).
  - `load_config()` → parses minimal YAML (keys: `jobs` list, `max_queue`, `log_file`).
  - `Logger` → logs to stdout and optional file (opened in write mode).
  - `submit_job(script)` → runs `sbatch`, parses job ID.
  - `check_job_status(jobid)` → polls `squeue`; returns `queued|running|completed` (no failure distinction).
  - `cluster_health_check(log)` → verifies `sbatch` and `squeue` exist.
  - `main()` → bounded‑concurrency loop with 5s polling.
- `README.md`: Features, usage, and known limitations.
- `AGENTS.md`: This guide.

## How We Work (Codex CLI)
- Planning first: Start with a clear plan (pseudocode only). Implement only when explicitly requested (e.g., “implement”, “write the code”).
- Preambles: Before running grouped commands or edits, send a brief 1–2 sentence preamble explaining the next actions.
- Tools:
  - `apply_patch` → make precise, minimal file edits. Do not add license headers unless asked.
  - `shell` → read‑only exploration preferred; avoid destructive commands. Request escalated permissions only when strictly necessary (e.g., writing outside workspace or network access).
  - `update_plan` → use for multi‑step, non‑trivial tasks to track progress; keep exactly one `in_progress` step.
- Safety defaults (current setup): workspace‑write filesystem sandbox, network restricted, approvals on‑request. Prefer approaches that don’t require escalation.

## Coding Guidelines
- Keep changes minimal and focused on the requested scope; don’t refactor broadly or fix unrelated issues.
- Follow existing style; prefer readability and PEP 8 conventions.
- Avoid one‑letter variable names; avoid adding inline comments unless requested.
- External dependencies require explicit approval (PyYAML is approved for YAML parsing).
- Update documentation (README, this file) when CLI behavior, flags, or outputs change.
- Never commit or create branches unless the user asks for it.

## SLURM‑Specific Notes
- `check_job_status` relies on `squeue` only; if a job disappears from `squeue`, the script marks it as `completed` (success vs failure is not determined). `sacct` usage is intentionally disabled in the code.
- `cluster_health_check` must find `sbatch` and `squeue`; the CLI exits if they’re missing. Avoid running the full CLI in non‑SLURM environments during development.
- Log file is opened in write mode; running multiple sessions with the same `-l` will overwrite previous logs.

## Testing and Validation (No SLURM Assumed)
- Prefer static validation:
  - Inspect `parse_args()` and `load_config()` to ensure YAML keys and defaults match README.
  - Keep output format changes deliberate and minimal; users may parse logs downstream.
- Avoid executing the CLI end‑to‑end locally if SLURM tools aren’t present (it will fail the health check).
- If adding features that need runtime checks, consider implementing a guarded `--dry-run` (no `sbatch`/`squeue` calls) to enable local verification before merging. Propose it first if not requested.
- Do not add a test framework unless requested; follow existing repo patterns.

## Common Change Patterns
1) Add a new CLI flag
   - Update `parse_args()` with the flag and default.
   - Wire it through the minimal necessary logic.
   - Update `README.md` usage/examples and this file if behavior assumptions change.
   - Validate by re‑reading code paths; avoid running SLURM commands locally.

2) Adjust concurrency or polling
   - Tweak `-q` semantics or polling interval thoughtfully; keep defaults unless a change is requested.
   - Consider backoff strategies only if explicitly requested; avoid complexity creep.

3) Improve status detection
   - If re‑enabling `sacct`, gate it behind a flag and document failure modes.
   - Preserve current simple behavior unless a more accurate flow is explicitly requested.

4) Logging changes
   - Keep timestamps and concise messages stable; if adding fields, opt‑in via a flag.
   - Consider switching to append mode (`a`) only if requested; document the change.

## Review Checklist
Before editing
- Confirm the exact request and scope; decide if a plan is needed.
- Read relevant files and note assumptions (especially CLI flags and defaults).

While editing
- Make the smallest viable change; keep names and structure consistent.
- Don’t introduce new dependencies or refactor unrelated areas.
- Keep changes localized; prefer adding guarded code paths over global changes.

Before hand‑off
- Ensure README usage matches `parse_args()`.
- Sanity‑check log messages and behavior descriptions.
- Summarize changes, risks, and any follow‑ups in the final message.

## Approvals and Sandbox
- Request escalated permissions only when necessary (e.g., installing tools, network access, or writing outside workspace). Provide a one‑sentence justification when you do.
- Avoid destructive shell actions unless expressly requested (e.g., `rm -rf`).

## Quick Reference
- Run help locally (safe): `sed -n '1,200p' ciai_wrangler` to inspect CLI.
- Entrypoint: `./ciai_wrangler` with args `CONFIG_YAML`.
- Known limitation: Missing from `squeue` → logged as `completed` (success unknown).

## Hand‑Off Style
- Provide a concise summary of what changed, why, and any risks.
- Offer next steps or options (e.g., add `--dry-run`, append‑mode logging) without over‑scoping.

---
This guide aims to keep contributions predictable, safe, and fast for a subprocess‑driven CLI that interacts with SLURM. When in doubt, propose the plan first and wait for explicit approval to implement.
