# Minimal ReasoningBank adaptation to pi + CyberGym-E2E

Updated 2026-09-18. The user approved implementation; the native extension and harness integration are now implemented and verified with offline tests and a local-service Docker system test. See the [implementation README](../cybergym-e2e-extended/baselines/reasoning-bank/README.md) for current usage and validation limits. The goal remains generic coding-agent self-evolution research, with CyberGym-E2E as the evaluation harness.

**The detailed, current proposal is [baselines/reasoning-bank/PLAN.md](../cybergym-e2e-extended/baselines/reasoning-bank/PLAN.md).** The planning directory has moved into the extended harness repository, alongside its other baselines.

## Verified starting point

The supplied upstream archive was inspected at revision `b46456c46838b2b090d7e6ded5bfdf1ff583dba7`. The actual implementation target is the user's [cybergym-e2e-extended submodule](../cybergym-e2e-extended/README.md), now at `2ae5fdbe36474a1c44a033c6527e2cc63e24bd54`.

The extended harness already supports pi. It provides model/endpoint profiles, native session export, streamed JSON logs, sequential coding tools, task containers, timeouts, and independent evaluation. Its installer pins pi 0.84.1. That version's published package was inspected and supports the extension interfaces required here; the desktop's 0.85.1 is not the benchmark target.

The latest two commits add [the Pi Live-SWE baseline](../cybergym-e2e-extended/baselines/pi-live-swe/README.md) and its runner integration. The shared `pi.execute()` now accepts `extension_path`, which replaces the default sequential-tools extension for callers such as Live-SWE. ReasoningBank should preserve this interface and the existing baseline behavior; the proposed configuration enables memory only for standard `agent=pi`.

| Existing component | Reuse |
|---|---|
| [agents/pi_reasoning_bank.py](../cybergym-e2e-extended/scripts/agents/pi_reasoning_bank.py) | Memory-enabled execution using the existing pi setup/model/session helpers and two explicit extensions; standard pi and its override interface remain unchanged. |
| [run_config.py](../cybergym-e2e-extended/scripts/run_config.py) | Extend the existing resolved configuration with an optional ReasoningBank config. |
| [pi-sequential-tools.ts](../cybergym-e2e-extended/scripts/pi-sequential-tools.ts) | Retain the normal tools and their execution order. |
| [agents/pi_live_swe.py](../cybergym-e2e-extended/scripts/agents/pi_live_swe.py) and [its tests](../cybergym-e2e-extended/tests/test_pi_live_swe.py) | Preserve the newly added baseline and verify that optional ReasoningBank wiring does not change its selected extension. |
| [run_agent.py](../cybergym-e2e-extended/scripts/run_agent.py) | Reuse task orchestration, collection, cleanup, and scoring. |
| [run_e2e.sh](../cybergym-e2e-extended/run_e2e.sh) | Keep the existing launcher/model profiles; run memory-enabled tasks sequentially. |

## Minimal adaptation

Build one native pi package under `cybergym-e2e-extended/baselines/reasoning-bank/`, with a JSONL experience bank and small harness integration edits. The package is part of the submodule and resolves from that repository's root. Use the standard pi workflow, one fresh session per task, and one attempt.

Preserve ReasoningBank's algorithm:

1. Embed the existing task query and retrieve the top one prior experience by cosine similarity.
2. Append all of that experience's lessons to pi's system prompt.
3. Let the existing pi agent execute the task.
4. Judge success/failure from the observed trajectory with the actor's model.
5. Extract at most three reusable title/description/content lessons using the appropriate success/failure prompt, then append the experience for the next task.

The user selected **gemini-embedding-001 by default**, with an **OpenAI-compatible embedding service** supported as an alternative. Keep the embedding setting independent of the actor profile and avoid mixing incompatible vector spaces in a bank.

Implementation must include frequent, concise source comments at the algorithmic steps. Cite the paper section and the original ReasoningBank function or prompt, explain the preserved behavior, and label intentional adaptation choices. The detailed plan identifies references for retrieval, injection, judging, extraction, and append-only consolidation. These comments complement behavioral tests and license attribution.

## Lifecycle choice

The implementation retrieves in pi's `input` hook and injects prepared lessons in `before_agent_start`. This lets a retrieval failure stop the actor; pi catches exceptions in the latter hook and would otherwise continue. After the actor exits, the memory adapter invokes a tool-disabled `/reasoningbank-learn` command using the saved native session and the same model configuration. It exports the new experience before container cleanup and appends it on the host.

This explicit finish step handles ordinary completion and timeouts through one path. It supersedes the earlier automatic `agent_settled` extraction idea, which would also have needed recovery when the process was interrupted. It is a memory-processing call, not another task attempt.

The original task prompt is the retrieval query. Its similarity across tasks is a limitation to measure initially, without adding a profile-generation model call or a specialized investigation workflow.

## Evaluation

Retain the harness's official evaluation. End-to-end S1–S3 establish successful task completion; S4 diagnoses whether the original target vulnerability was fixed. These scores are not the memory learner's input: the first version retains ReasoningBank's LLM judgment and learns from both successes and failures.

Use separate banks for separate experiments/modes. Preserve the harness's existing input preparation and network configuration. Report memory-processing overhead and errors alongside actor results. Start with a small ordered comparison of pi without memory and pi with an initially empty bank.

MaTTS, verifier-supervised learning, alternative retrieval, and more elaborate memory management are **fully optional research possibilities**, not deferred commitments. They may never be implemented, and the initial adaptation does not need placeholders or infrastructure for them.

See [the implementation plan](../cybergym-e2e-extended/baselines/reasoning-bank/PLAN.md) for file-level changes, configuration, test cases, and acceptance criteria; see [the original algorithm analysis](reasoningbank-implementation.md) for the paper-to-code mapping and upstream mini-swe-agent comparison.
