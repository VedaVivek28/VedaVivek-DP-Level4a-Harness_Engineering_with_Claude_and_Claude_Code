# Evidence Verification Index

| System | Required Log / Count | Observed Result | Key Run Artifact(s) |
| :--- | :--- | :--- | :--- |
| **System 1: Agentic Loop** | `pytest_S1.log` (29 passed) | 29 passed | `summary.md` (all 8 routed or escalated), trace JSONL |
| **System 2: Context Strategy** | `pytest_S2.log` (17 passed) | 17 passed | `budget.json` (56.38% reduction), `eval.jsonl`, `eval_control.jsonl` |
| **System 3: Claude Code Config** | `pytest_S3.log` (35 passed) | 35 passed | `validator_output.txt` (OK), `CLAUDE.md`, `.claude/` rules/skills |
| **System 4: Orchestration** | `pytest_S4.log` (28 passed) | 28 passed | `shift_run_output.txt` (SQL filter slice), `hot_state_size.txt` (643 bytes) |
