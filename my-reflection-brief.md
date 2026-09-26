# Reflection Brief — Harness Engineering Capstone

**Name: Buddannagari Deepthi reddy
**Date:24/09/26

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):claude-3-haiku-20240307 /claude-3-5-sonnet-20241022
- OS / Python:Linux / Python 3.11+
- Approx. API spend:$1.50

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In claims_intake/loop.py, the execution engine inspects the response's stop_reason attribute on every turn. The trace output in runs/traces/ records a sequence of stop_reason: "tool_use" during tool execution phases, which transitions to stop_reason: "end_turn" when the model concludes its reasoning and routes or escalates the claim. The loop continues iterating while stop_reason == "tool_use" and terminates immediately upon hitting end_turn.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → tests/test_antipatterns.py explicitly checks for unhandled recursive tool execution loops that lack a proper stop_reason termination guard. If the system employed this anti-pattern, an unconstrained model could fall into an infinite tool-calling loop, rapidly burning through API tokens and failing to return a final routing decision.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → Tools like lookup_policy and check_claim_status feature explicit, non-overlapping input schemas and rich docstrings that guide the model on when to use each. When execution fails, returning a structured JSON error dictionary rather than a generic text string allows the agent to inspect the failure parameters and dynamically self-correct its arguments on the subsequent turn.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → Examining the generated summary.md for claim CUST-9999, the process completed in 5 turns at an approximate cost of $0.012. This differs from the static README sample due to dynamic branching logic where tool responses and customer history nuances alter the precise conversation trajectory and turn count.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → According to budget.json in the run artifact folder, the baseline context size was 4,850 tokens, while the assembled context dropped to 2,120 tokens, achieving a 56.3% reduction (comfortably beating the 50% target). The active conversation block dominates the token footprint because it is preserved verbatim to maintain dialogue continuity and contextual nuance.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved issues are compressed into compact structured summaries averaging ~150 tokens to strip redundant dialogue, whereas active conversation turns are kept byte-exact to protect verbatim user expressions. This dual strategy balances high token efficiency with strict conversational fidelity.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → Comparing eval.jsonl with eval_control.jsonl (where the persistent case facts block was omitted), question 4 regarding customer account IDs and active subscription tiers experienced a clear regression. This proves that anchoring critical metadata in a persistent case block is essential for factual accuracy.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → Quoting the frontmatter paths: ["src/**/*.py"] from rule files, path-scoped rules inject conventions selectively into matching source files. This design is vastly superior to a monolithic root-level CLAUDE.md because it targets specific directory scopes, avoiding context window bloat in unrelated parts of the repository.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → Quoting context: fork and allowed-tools: ["View*", "Glob*"] from the skill definition, running in a forked context with read-only permissions guarantees that experimental code exploration takes place in an isolated sandbox. Without this, exploratory modifications could pollute the main conversation context or inadvertently alter production files.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → Validator output distinguishes project-level rules (defined locally in the workspace configuration files) from user-level settings (global developer preferences governed in the user's environment).

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The indexed warm-tier query (defects_since) returned only 12 relevant rows out of thousands of historical records. By delegating filtering to the database layer, the model receives a highly condensed slice of recent data and never has to process the full historical raw logs.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → recovery.py incorporates a STALE_RESUME_THRESHOLD_MINUTES = 30 threshold. If a shift crashes and the checkpoint age exceeds 30 minutes, the orchestrator safely triggers a fresh start with an injected summary rather than attempting to resume an obsolete or out-of-sync session state.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → The generated hot_state.json file measures roughly 1.2 KB, remaining safely below the ~5 KB budget limit. Maintaining a minimal state footprint ensures that reading and writing operational metadata during shift handoffs remains lightweight and reliable indefinitely.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model:Anthropic API client calls executed inside System 1 (claims_intake/loop.py) and System 4.
    → Harness:CLAUDE.md hierarchies, path-scoped rules, and prompt templates in System 2 and System 3.
    → Orchestration:Shift execution pipelines (shift_monitor/pipeline.py) and atomic state writers in System 4.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Deterministic enforcement is handled strictly in code via atomic file writing (os.replace + fsync) and SQL filtering, whereas behavioral guidelines like tone and formatting are governed by prompt instructions. Code-level enforcement is critical for safety, atomicity, and security, while prompts excel at flexible linguistic reasoning.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 compresses intra-session conversation down from ~4,850 tokens to ~2,120 tokens to optimize the active window. System 4 compresses cross-session state into a 1.2 KB hot_state.json manifest across shift transitions. Both solve context bloat through selective pruning—one across conversation turns, the other across operational shifts.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → Automated test suites (such as AST verification checks ensuring no Python-side filtering occurs outside SQL queries) guarantee compliance across hidden edge cases and rare failure modes that a single successful manual run cannot possibly validate.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In System 4, if the orchestration pipeline encounters a runtime error, the blast radius is tightly restricted through atomic file replacements (os.replace), strict API call limits (call_count == 1), and byte-budget trims on hot_state.json, effectively preventing cascading state corruption across shifts.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → Initially, path resolution errors occurred when running pytest due to virtual environment mismatch across directories. This was resolved by activating the dedicated virtual environment in each specific system root prior to invoking test suites.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would implement an in-memory caching layer for frequent database lookups in the orchestrator warm tier to further decrease latency during high-frequency shift monitoring transitions.
