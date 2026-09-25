# Reflection Brief — Harness Engineering Capstone

Name: Venkata Veda Vivek Boggavarapu
Date: 24th September, 2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): claude-haiku-4-5-20251001
- OS / Python: Python 3.13.0
- Approx. API spend: $0.13USD (from System 1 batch run)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In runs/20260924_145239/traces/claim_01_kitchen_fire.jsonl, the turn-by-turn sequence of stop_reason values is: Turn 1: "tool_use", Turn 2: "tool_use", Turn 3: "end_turn". The continue-vs-stop decision is executed in claims_intake/loop.py inside the function run_loop(). The function inspects the API return field response.stop_reason; if stop_reason == "tool_use", it appends tool calls to working_messages, executes the tool dispatch, appends the tool_result block, and loops; if stop_reason == "end_turn", the loop exits/terminates and parses the final classification.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → tests/test_antipatterns.py checks for the String Matching Termination Anti-Pattern. If the run relied on string parsing, an intermediate model thought or tool parameter containing the word "ROUTED" would trigger premature loop exit even before compulsory verification tools were invoked. This would break test_loop.py and cause intake cases like claim_01_kitchen_fire to exit without producing valid routing_slips.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → lookup_policy and verify_coverage both accept policy_id. lookup_policy's schema description restricts usage to initial metadata retrieval, while verify_coverage mandates passing both policy_id and incident_type to evaluate line-item limits. When an invalid ID was passed, the tool returned a structured JSON error {"status": "error", "error_code": "POLICY_NOT_FOUND", "recoverable": true} rather than a plain string "Error". This detailed and structured payload enabled the agent to branch conditionally, querying the alternative policy index, and self-correct instead of crashing or hallucinating an escalation.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → For claim_02_stolen_bike in runs/20260924_145239/summary.md, the run took 5 turns and cost $0.0206. In the reference README sample, the same claim resolved in 4 turns at $0.0162. The extra turn occurred because Haiku made a single-item check for police report verification prior to executing the routing tool call, showing dynamic multi-step decomposition under live sampling variability.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → From runs/20260924-150211/budget.json: Baseline tokens = 38,708, Assembled tokens = 16,886, resulting in a 56.38% reduction. The Active Turn Segment dominates the assembled context at 15,789 tokens. Keeping it exactly same is mandatory because recent turns contain active disambiguation, immediate customer tone, and uncommitted transaction state, compressing the active window would introduce semantic loss on questions that are not resolved.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Inactive/resolved conversational branches are compressed via LLM summarization, whereas critical case facts and the active conversational turn are preserved byte-exact. In budget.json, the resolved refund dialogue is compressed from 12,334 tokens down to 360 tokens, and the subscription issue is compressed from 11,475 tokens down to 516 tokens. In contrast, the immutable Case Facts block is pinned byte-exact at 204 tokens, and the Active Turn is retained byte-exact at 15,789 tokens.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In eval.jsonl, all 6 evaluation questions passed (6/6). This proves that lossy summarization drops machine-readable transaction codes (AVS_MISMATCH), and that pinning critical case facts at the top of context is necessary to prevent "lost-in-the-middle" memory degradation.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → In .claude/rules/react.md, the frontmatter declares: paths: ["src/components/**/*.tsx", "src/pages/**/*.tsx"]
   Path-scoped rules are higher to directory-level CLAUDE.md files because cross-cutting architectural standards span multiple disparate folders across the monorepo. Path-scoped rules load conditionally only when relevant globs are matched, avoiding prompt bloat while preventing rules duplication across various subdirectories.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → From .claude/skills/deploy-check/SKILL.md: context: fork
    allowed-tools: [Read, Grep, Glob"]
    Running forked and read-only isolates extensive pre-deployment audits in a temporary sub-agent session. Without context: fork, hundreds of lines of compiler output and tool logs would fill the primary developer session, rapidly utilizing the token context window and reducing prompt quality.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator outputs OK (exit code 0) verifying both scopes. A project-level component is .claude/commands/review.md, checked into version control to enforce shared team PR review standards. A user-level scope is ~/.claude/settings.json, preserved outside version control for individual preferences like local IDE shortcuts and personal tool telemetry.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → In data/warm.sqlite, the total warm-tier database contained 142 defects; the indexed query defects_since returned 8 defects for the active shift slice. The model never sees the entire database history because injecting 142 records into the prompt wastes tokens and brings noise; pushing filtering down into SQLite indexes ensuring the agent receives only shift-specific, actionable anomaly data.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → In recovery.py, the system evaluates crash recovery using a 30-minute staleness threshold on the unfinalized shift manifest. If an unfinalized shift crashed $\le 30$ minutes ago, the orchestrator continues using existing part state. If $> 30$ minutes have elapsed, operational physical conditions on the production floor have shifted significantly, meaning part in-flight state is stale; starting fresh with a merged summary prevents the model from acting on invalid real-time assumptions.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → ls -l data/hot_state.json yielded 643 bytes. Keeping hot state strictly budgeted is important because the system runs once per shift indefinitely; unconstrained state accumulation would cause memory leak, slowing down serialization, and followed by exceed context boundaries when reloaded across shifts.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: claims_intake/agent.py in System 1 directly manages the raw LLM client (anthropic.Client), model parameters, and raw API message generation.
    → Harness: CLAUDE.md and .claude/rules/*.md in System 3 define path permissions, @import standards, and tool execution constraints governing how the LLM interacts with codebases.
    → Orchestration: shift_monitor/pipeline.py and shift_monitor/recovery.py in System 4 orchestrate multi-shift execution cycles, crash recovery policies, SQL pre-filtering, and tiered disk persistence.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → In System 4 (tiered_state.py), atomic state writing and the 20-hash cap on hot state are guaranteed in code via POSIX file replacement and Python validation; if violated, the system raises an exception. Conversely, in System 3 (react.md), preferring functional components and descriptive prop types is instructed by prompt instructions. Deterministic code execution is necessary for data integrity and security boundaries, prompt guidance is suitable for conversational nuances.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → Both systems enforce context minimization to prevent degradation, even though they apply different architectural mechanisms. System 2 manages intra-session context dynamically in-memory, shrinking 38,708 transcript tokens down to 16,886 tokens via persistent case facts and hierarchical summarization. System 4 manages cross-session context externally by pushing history entirely out of the prompt into tiered storage, querying only 8 defects via SQL, and capping hot state to 643 bytes.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → In System 1, test_antipatterns.py and test_loop.py by programming injects mock tool failures and API exceptions to verify that the loop triggers self-correction rather than crashing or infinite looping. A single manual execution only follows the "happy path" on a valid fixture. Automated tests guarantee that boundary conditions and malformed inputs are handled reliably before deployment.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In System 4, an unconstrained agent could corrupt factory line monitoring or overwrite historical inspection logs. The blast radius is contained because WarmStore allows read-only indexed SQL queries during shift monitoring, and the hot state has an enforced 5 KB / 20-hash limit validated in code. The kill switch is the orchestrator's crash recovery staleness gate in recovery.py. If anomaly counts exceed thresholds or the run fails to finalize, state mutations are halted and reverted without touching cold storage.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → During the initial run of System 1, the script threw an anthropic.BadRequestError: Error code: 400 - {'message': 'This key was not found. Please check key was inputed correctly.'}. Investigation revealed the shell environment had an incomplete key due to not completely inputing the key. Re-writing both export ANTHROPIC_BASE_URL="https://claude.vocareum.com" and the complete key solved the issue immediately.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → In System 1, run.py runs all claims sequentially in a single synchronous loop. I would restructure the runner to process claims asynchronously using asyncio.gather() or a worker queue. Since intake evaluations are independent across claim files, parallel execution would reduce the overall intake time.
