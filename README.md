# HermeStrata-2xV100

> **Successor of [HermeStrata](https://github.com/urawazakun/HermeStrata)** (frozen, the one-V100 edition). This repo starts from its final
> state (`f9137fd`, 2026-10-03) and continues on a machine with **two Tesla V100 16 GB** (Windows, driver 472.12, CUDA 11.8, sm_70).
>
> **Honest status (2026-10-03):** the second card was installed today. Production still runs on one card (GPU 0, FlashNext at
> 262K context); GPU 1 is idle. Nothing in the patches below uses two cards yet, and every measurement below was taken on one V100.
> Planned use of the second card: (1) a fast stateless *decision model* (Cloudflare Clef, Jev-compatible) for the roguelike player,
> swapped in/out of the first card by freezing Strata's VRAM in host RAM (PARK/UNPARK), or running on the second card; (2) engine
> work on upstream Strata 0.1.36 (rebase of this series in progress, GPU acceptance pending). See the "Two-card plan" section.

## Two-card plan (not done yet)
| item | state |
|---|---|
| Strata 0.1.36 rebase of patches 0001-0021 (CUDA 11.8 / Windows port kept) | source ready, CUDA build + GPU acceptance pending |
| PARK/UNPARK: copy the engine's VRAM to host RAM, free it, run another model, restore exactly (saved states stay in RAM) | design only (feasibility of CUDA VMM on driver 472.12 unknown) |
| Clef 27B decision model (NF4 / offload on torch 2.7.1+cu118) as the game player, FlashNext writes plans only | feasibility study done (briefs not published); full 27B load not measured |
| Which engine uses which card (config has no GPU index today) | not decided |

The text below is the one-V100 edition, kept as the baseline.

---

# HermeStrata

A local agent stack for **one user, one GPU, one serialized engine slot with multiple cached conversation lines**:
[Hermes Agent](https://github.com/NousResearch/hermes-agent) (planner/implementer agents, `delegate_task`) driving
[Strata](https://github.com/Niko1221/Strata) (hybrid-attention MoE inference engine) serving Qwen3.8-Flash-Next
IQ3_S at 262K context on a single **Tesla V100 16 GB** (Windows, CUDA 11.8, sm_70).

This repo is not a fork. It contains the patches, the cache design that ties the two together, the launch/acceptance
scripts, and the measurements. Both upstreams are MIT-licensed; the patches keep their licenses.

> Status: experimental, personal setup. Nothing here is upstreamed.

## Why

Hermes is a multi-agent harness: a parent turn, then a child task, then the parent again, plus small side requests
(session titles). Upstream Strata keeps **one live conversation** of saved state; any unrelated request prunes it.
With a 12K-token Hermes system prompt + tools, and conversations that grow to 100K tokens, every switch between
parent, child and title meant re-reading the whole prompt (34 s for 12K, minutes for 100K on this GPU).

Why upstream keeps a single line (our reading): the model's recurrent state (GDN + PLE + QSA tails) cannot be
truncated like a KV cache — you can only return to points where a ~113 MiB snapshot was taken — and each extra
line costs GBs of host RAM and adds ways to restore the wrong state silently. For a single chat on a 32-64 GB box,
one line is the right trade-off. With 128 GB host RAM and an agent harness, several lines pay off.

## What is in it

| Layer | Change | Where |
|---|---|---|
| Strata | V100 (sm_70) / CUDA 11.8 / Windows port, with docs | `patches/strata/0001` |
| Strata server | auxiliary requests (titles: no tools + `response_format`) wait for the main turn and never evict its state | `0002` |
| Strata | adaptive MTP draft window (`--spec-adapt`, off; no gain measured on V100) | `0003` |
| Strata | **saved states**: pinned *roots* (fixed prefix = tools + system prompt) and LRU *tips* (end of each conversation line) in host RAM, restore of the longest validated prefix, per-request resume/snapshot stats | `0004` |
| Strata | roots persisted to disk (identity header, lazy load, checksum, size cap, `--root-dir`, `--no-root-disk`) | `0005` |
| Strata | aux composition tests, authenticated `/v1/cache` inventory + DELETE of non-pinned tips | `0006` |
| Strata | fixes from GPU acceptance: fall back to the next valid restore point instead of cold (F1); tips restorable at the last turn boundary of the prompt (F2/F2b) | `0007` |
| Strata server | **prefix keeper** (`--prefix-keeper`): keeps Hermes requests append-only for the cache — volatile system lines become a trailing `[context update]`, pruned history is restored; idea by the owner | `0008` |
| Strata | upstream 0.1.29/0.1.30 follow-ups: 0.1.30's `dead`/`block_pos` recurrent state carried through capture/restore/disk (disk schema v3), K8V4 KV explicitly unsupported for saved states, single MTP prefill, mutual exclusion with upstream's opt-in conversation cache | `0009` |
| Strata + server | **readability refactor** (behaviour-preserving, from a ChatGPT review): cache-protocol client class (`serve/cache_protocol.py`), restore candidate selection as a typed list/result, two-phase disk restore (validate without GPU mutation, then mount), `saved_state` split into RAM store / disk codec / disk store, fix-history labels moved to `docs/CACHE-HISTORY.md`. GPU-accepted and in production | `0010`-`0014` |
| Strata server | fix a `KeyError` when two requests overlap (upstream 0.1.30 #212 pops the status tail at a request's end while another request is still writing it); found by running an agent and a second client at the same time | `0015` |
| Strata | **tail resume**: a 40 KiB MTP boundary row on every conversation checkpoint and at each request's end, so a conversation resumes at its end. Live agent runs re-read 5-18K tokens per request before (once 149K after a mid-history edit); GPU acceptance: follow-up turns read only their 26 new tokens, a mutated middle message resumes from the newest checkpoint below it, an identical repeated 11.7K request 0.09 s (was 0.5 s) | `0016`-`0017` |
| Strata server | prefix keeper diagnostics: timestamps, and for an unrepaired history mutation the first differing offset, a cause hint and short excerpts (JSONL only) | `0018` |
| Strata server | vision embedding cache defaults to 400 files, configurable with `STRATA_VISION_CACHE`; prevents a many-image request from deleting embeddings it still needs | `0019` |
| Strata | retain stable prompt-only tips alongside generation tips for alternating conversations; retention diagnostics and CPU regressions | `0020` |
| Strata tests | alternating two-line GPU acceptance with an independent tokenizer/template oracle and a suffix + 256-token read bound | `0021` |
| Hermes | fixed prefix discipline: git workspace snapshot moved to the volatile tail (`HERMES_WORKSPACE_LATE=1`), title generated after the turn (`HERMES_TITLE_AFTER_TURN=1`) | `patches/hermes-agent/0001` |
| Hermes | `delegate_task` **fork mode**: a child starts from the parent's exact prompt (+ its tool call + a tool result carrying the task), so the engine resumes it from the parent's saved state; blocked tools and depth limit are refused at call time instead of being removed from `tools[]` | `patches/hermes-agent/0002` |
| Hermes | **work deadline** tools (`set_work_deadline` / `finish_work`): "work until 18:00" keeps the agent going until the deadline; run budget stays the ceiling | `patches/hermes-agent/0003` (experimental) |
| Hermes | fix a delegation **deadlock** (a finished child blocked in a process-wide telemetry flush while the parent waited for it); **`ask_document`** tool = cache-augmented reading: the document is a byte-stable system prefix, so the engine keeps it as a root on disk and the 2nd question on the same document skips re-reading it (live: 165 s -> 43 s end to end, 26K-token document) | `patches/hermes-agent/0004` |
| Hermes | fork mode refactor: a `ForkContext` dataclass replaces the private attributes a fork child carried (behaviour-preserving; recorded first-request parity test); from a ChatGPT readability review | `patches/hermes-agent/0005` |
| Hermes | `HERMES_KEEP_TOOL_IMAGES=int\|all` (default 3): Hermes rewrites all but the newest 3 screenshot tool results to a placeholder on every request, which changes old history bytes and breaks a local engine's cache (live: 149K tokens re-read after the 6th vision call). `all` keeps them byte-stable until compression; set in the flashnext launcher | `patches/hermes-agent/0006` |
| Hermes | opt-in `HERMES_THINK_MODE=adaptive`: loopback custom/local Chat Completions requests think OFF by default; `think_harder` enables 1–5 subsequent calls with configured effort. `HERMES_MAX_OUTPUT_TOKENS` bounds output (default 16384); lower explicit caps survive, invalid/nonpositive competing caps are bounded | `patches/hermes-agent/0007` |

Design: [docs/DESIGN.md](docs/DESIGN.md). How the fixes were found: [docs/KNOWN-ISSUES.md](docs/KNOWN-ISSUES.md).
Acceptance ledger: [docs/ACCEPTANCE.md](docs/ACCEPTANCE.md). Live Hermes runs and what they exposed: [docs/LIVE-HERMES.md](docs/LIVE-HERMES.md).

Memory cost per saved line (measured unit costs): KV int8 12.7 KB/token, MTP rows 1.06 KB/token, recurrent state
113 MiB per snapshot. A 12K root ≈ 0.27 GB; a 100K conversation tip ≈ 1.5 GB. Default tip budget 8 GiB.

## Results (2026-09-30, V100 16 GB, ctx 262144, `--kv int8 --kv-resident 20480`, expert cache 3127, 3 repeats)

Hermes system prompt + tools = 11.7K tokens. Cold prefill of that on this GPU: **34 s**.

| situation | before (upstream behaviour) | HermeStrata |
|---|---|---|
| new session after a title (aux) request | 34 s | **0.50-0.66 s** (root) |
| parent turn after an isolated child ran | full re-read | **0.72-0.77 s** (tip at the turn boundary, 26 tokens read) |
| forked child | full re-read | **1.7 s** (parent's chain point, 64 tokens read) |
| isolated child, 2nd time | full re-read | **0.17 s** (its own tip) |
| identical request repeated | reuse | 0.45-0.53 s |
| first session after an engine restart | 34 s | **2.5 s** (root from disk) |

Restore correctness: a fact planted in the first user message is recalled by the forked child 3/3. 20 mixed
Hermes-shaped requests cached vs truly cold: all sane. Token parity is deliberately not a criterion (quantized
kernels + KV paging drift run to run). Decode speed unchanged: 26.7 vs 26.5 tok/s (8K doc). `--spec-adapt`
showed no measurable gain here and stays off.

Not yet measured on GPU: tip-budget eviction under pressure (item 6 of the ledger).
## 2026-10-01: rebased onto upstream Strata 0.1.30

Production now runs the 0.1.30-based series (GPU acceptance on V100, same settings as above): saved states, root-disk, parent/child/fork and planted-fact recall all pass; Japanese answers +12-15% vs the 0.1.21 build (0.1.27 CJK draft head); a 39K-token prompt reads 12% faster; 8K-doc decode 25.1 vs 23.2 tok/s (0.1.29).  The 0.1.21 series and its history stay in `patches/strata-0.1.21/`.

Later the same day the readability refactor (`0010`-`0014`) passed the same GPU acceptance (root/aux, parent/child/fork with planted-fact recall, Japanese, a fact planted in 39K tokens 3/3) and replaced production. An identical repeated 11.7K request now resumes in 0.48-0.56 s (the 0.1.30 port had 1.3 s); 8K-doc decode 28.9-29.2 tok/s.


## Layout

```
patches/strata/         21 patches on Niko1221/Strata v0.1.30 (30ec18ec...), one per concern
patches/strata-0.1.21/  the earlier 17-patch series on 0.1.21 (history)
patches/hermes-agent/   git format-patch against the Hermes Agent snapshot used by this project
docs/                   design, fix history, acceptance ledger
scripts/                server launcher (PowerShell), Hermes launcher (.cmd), GPU acceptance driver
```

Scripts resolve the project root from `HERMESTRATA_ROOT` (default: the parent of `scripts/`); model and pack paths come from the launcher's config section. Paths only matter at startup, not for speed.

## Reading the final code

`pwsh scripts/materialize.ps1` clones both upstreams at the pinned bases and applies the patch series (one commit per
patch) into `materialized/strata` and `materialized/hermes-agent`, so the current state can be read as plain source
instead of composing the patches in your head. Checked: the Strata result has the same tree hash as the source commit
the patches were generated from.

## Patch bases

The Strata series is pinned to upstream tag v0.1.30 (`30ec18ec7094550fcc594fd948220d511d80464e`); scripts/materialize.ps1 rebuilds exactly the tree production runs (tree hash checked). Do not assume the series applies
cleanly to current Strata `main`; upstream has moved since this snapshot.

The Hermes series is pinned to `d5aaaa4a`, as recorded in `scripts/materialize.ps1`. Both reconstructed source
trees are checked against the reviewed source commits. Treat a rebase onto current Hermes Agent as a separate
compatibility task rather than silently applying with rejects.

## 2026-10-02 maintenance

The combined vision-cache and two-lines candidate passed one isolated GPU acceptance repetition: 12/12 measured
requests, ten warm follow-ups within the same-line suffix + 256-token bound. It is deployed locally. This measures
cache retention, not general answer quality or token parity. CPU: 14 C++ tests passed, one AVX-512-only test skipped;
51 frontend tail tests passed. See [the acceptance ledger](docs/ACCEPTANCE.md) for the lifecycle failure and recovery.

Adaptive thinking passed 86 policy/integration tests plus 74 prompt/provider/tool-cache regressions, all offline.
It is opt-in source functionality; no profile or launcher switch was changed. GPU performance for this mode is
unmeasured. Patch preparation stops before pushing.

## Credits

Strata: Niko1221 and contributors. Hermes Agent: Nous Research. Design and review in this repo: the owner with
Claude (Anthropic); implementation of the cache patches: Muse (via OpenCode) orchestrated and reviewed by Sol (Codex).
