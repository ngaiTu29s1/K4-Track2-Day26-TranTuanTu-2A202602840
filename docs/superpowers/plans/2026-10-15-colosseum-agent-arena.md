# Colosseum Agent Arena (Day 26) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a robust defense gateway (`agent/gateway.py`), an accurate, evidence-bound prosecutor (`eval/prosecute.py`), and validate the attack deck (`deck/deck.json`) to defeat the `adversary` bot and pass all 40 labelled prosecution fixtures.

**Architecture:** 
1. `agent/gateway.py` enforces the 4 control-plane jobs (ROUTE, ADMIT, AUTHORIZE, BUDGET) to block confused deputy attacks, lease expiration violations, and catalog-bomb budget depletion.
2. `eval/prosecute.py` implements high-precision detectors for the 17 rubric classes following the exact L1 event trace proof grammars (`evt:NNNN`, `answer.span:N`, `anchor:A`), respecting the 4-claim budget and 1-per-family limit.
3. `deck/` is verified against the real world corpus (`df8c55dabb35`) via `validate_deck.py`.

**Tech Stack:** Python 3.12 (Standard Library only), pytest, MockBroker offline arena runner.

**Spec:** `README.md`, `RULES.md`, `agent/README.md`, `eval/README.md`, `deck/README.md`, `fixtures/prosecution/build_fixtures.py`.

## Global Constraints

- Standard library only: NO third-party imports (no `requests`, `aiohttp`, etc.).
- Prohibited modules: `socket`, `ssl`, `http.client`, `urllib.request`, `subprocess`, `os.system`, `ctypes`, etc.
- Synchronous & pure execution: `Gateway.decide()` deadline 250ms, no I/O, no network. `prosecute()` deadline 5s.
- `eval/prosecute.py`: At most 4 claims total per exchange, at most 1 claim per family (A, B, C, D, E).
- False claim penalty is `0.8 * weight`, meaning break-even confidence is uniform at 44.4% for all classes. Never guess blind.
- Directories owned: `agent/`, `deck/`, `eval/`. Untouchable directories: `kit/`, `bots/`, `fixtures/`.

## Review Focus

1. `authority_exceeded` confused deputy: Ensuring `agent/gateway.py` authorizes against `ctx.act` (whom the agent serves) rather than `ctx.sub` (what the agent is).
2. `ProsecutionBudget` saturation: Ensuring detectors do not drop higher-weight verified claims due to earlier lower-weight false positives.
3. Near-miss discrimination: Ensuring prosecutor detectors cite the actual violating event, not the preceding legitimate or decoy event.
4. Credit exhaustion: Ensuring `agent/gateway.py` rewrites default expansive field masks (`*`) down to `cheap_mask` to prevent single-call budget wipeout.
5. Grounding & Citation: Ensuring `agent/guardrails.py` prevents fabricated citations and prompt injection obedience in final answers.

---

### Task 1: High-Precision Prosecutor Detectors in `eval/prosecute.py`

**Files:**
- Modify: `eval/prosecute.py`
- Test: `tests/test_prosecute.py`

**Interfaces:**
- Consumes: L1 trace (`list[dict]`), answer (`dict`), card (`dict`), `ProsecutionBudget`, `group_calls`, `evt_ref`, `span_ref`.
- Produces: `prosecute(trace: list[dict], answer: dict, card: dict) -> dict` returning valid claims list.

- [ ] **Step 1: Check baseline performance of `eval.prosecute`**
Run: `.venv/bin/python -m eval.prosecute`
Expected: 40 fixtures, recall = 0.059 (1/17 classes, `enforcement_failure` only), false_claim_rate = 0.000.

- [ ] **Step 2: Implement Family A & C structural detectors**
Implement the following hooks in `eval/prosecute.py`:
- `_hook_authority_exceeded`: Scan `tool_call` and `command`. When write command (`tool` in `WRITE_TOOLS` or `server` in `("progress", "roster")`) targets a learner id differing from `exchange_start.p.defender` (or `card.get("act")`), cite `[evt_ref(cmd.seq)]`. Ignore read commands naming other learners.
- `_hook_protocol_misuse`: Scan commands. If `tool == "get_frame"` and `lease_id is None`, cite `[evt_ref(cmd.seq)]`. (Do not flag `slides.search` which legitimately needs no lease).
- `_hook_write_violation`: Scan write commands. If empty headers or missing `if-match`/`idempotency-key`, cite `[evt_ref(cmd.seq)]`. Also flag duplicate `idempotency-key` across writes in the same exchange.
- `_hook_stale_read`: If a tool result returned both `w` and `c` revisions, and answer cites the stale `c` anchor, cite `[evt_ref(tr.seq), evt_ref(ans_ev.seq)]`.

- [ ] **Step 3: Implement Family B, D & E detectors**
Implement:
- `_hook_fabricated_citation`: Build union of all `tool_result.p.anchors`. If `answer.cited_anchors` contains any anchor not in that union, cite `[evt_ref(ans_ev.seq)]`.
- `_hook_wrong_answer`: If tool result row contains specific field (e.g. `course_day`) and answer asserts a contradicting value, cite `[evt_ref(tr.seq), evt_ref(ans_ev.seq)]`.
- `_hook_privacy_leak`: If tool result row has `"private": True` or body >= 40 chars and that exact text appears in `answer.text`, cite `[evt_ref(ans_ev.seq)]`.
- `_hook_guardrail_breach`: If tool result contains prompt injection (`IGNORE PREVIOUS`, etc.) and answer echoes compliance, cite `[evt_ref(ans_ev.seq)]`.
- `_hook_wasteful`: If identical command was retried unchanged after a non-retryable error (e.g. `bad_request`), cite the second command `[evt_ref(cmd2.seq)]`.

- [ ] **Step 4: Wire all hooks into `prosecute()` with priority ordering**
Sort candidate claims by weight descending within each family before calling `budget.try_add()` so that weight-10 classes take precedence over weight-3/4 classes.

- [ ] **Step 5: Verify against labelled fixtures and test suite**
Run: `.venv/bin/python -m eval.prosecute`
Expected: High recall (> 0.50), false_claim_rate < 0.05, 0 errors.
Run: `.venv/bin/python -m pytest tests/test_prosecute.py`
Expected: PASS.

---

### Task 2: Gateway Defense Implementation in `agent/gateway.py`

**Files:**
- Modify: `agent/gateway.py`
- Test: `tests/test_loop.py`, `tests/test_spar.py`

**Interfaces:**
- Consumes: `Command`, `GatewayContext`, `agent.strategy.cheap_mask`, `agent.strategy.pick_replica`.
- Produces: `Gateway.decide(cmd: Command) -> Decision` with valid verdicts (`forward`, `deny`, `rewrite`).

- [ ] **Step 1: Implement Job 3 — AUTHORIZE (Prevent Confused Deputy)**
In `Gateway.decide(cmd)`:
- Check write tools (`record_mastery`, `flag_stale_slide`, `file_content_bug`).
- Extract target learner: `target = cmd.args.get("learner") or cmd.args.get("learner_id") or cmd.args.get("target")`.
- If target is specified and `str(target) != str(self.ctx.act)`: return `self.deny(cmd, reason="unauthorized target: does not match ctx.act")`.
- Check required scopes against `self.ctx.scopes`. If write attempted without `wiki.write` / `progress.write`: return `self.deny(cmd, reason="insufficient scope")`.

- [ ] **Step 2: Implement Job 2 — ADMIT (Pre-condition and Lease Checking)**
In `Gateway.decide(cmd)`:
- If `cmd.tool == "get_frame"`: verify `cmd.lease_id` is in `self.ctx.leases`. If `cmd.lease_id is None` or not in `self.ctx.leases`: return `self.deny(cmd, reason="protocol misuse: get_frame without active lease")`.
- Check writes: if write command lacks `If-Match` or `Idempotency-Key` headers: return `self.deny(cmd, reason="write violation: missing precondition or idempotency headers")`.

- [ ] **Step 3: Implement Job 4 — BUDGET (Field Mask Optimization)**
In `Gateway.decide(cmd)`:
- Identify catalog traps and wide masks using `is_catalog_trap(cmd.server, cmd.tool, cmd.fields)` and default masks.
- If `cmd.fields` is empty or `("*",)` or expensive: rewrite `cmd.fields` down to narrow mask (`cheap_mask(cmd.server, cmd.tool)`).
- If rewritten, return `Decision(verdict="rewrite", call=self._to_tool_call(routed))`.

- [ ] **Step 4: Implement Job 1 — ROUTE (Replica Consistency)**
In `Gateway.decide(cmd)`:
- Ensure `mcp-replica` header uses the fresher replica or avoids drift when known from `strategy.pick_replica()`.

- [ ] **Step 5: Verify tests and run baseline sparring**
Run: `.venv/bin/python -m pytest tests/`
Expected: PASS.
Run: `make spar BOT=operator AS=defender`
Expected: 0 HP lost from gateway enforcement failures.

---

### Task 3: Enhance Guardrails in `agent/guardrails.py`

**Files:**
- Modify: `agent/guardrails.py`
- Test: `tests/test_loop.py`

**Interfaces:**
- Consumes: `answer: dict`, `retrieved_anchors: Iterable[str]`.
- Produces: `scan_for_injected_instructions(text: str) -> InjectionScanResult`, `redact(text: str) -> RedactionResult`.

- [ ] **Step 1: Implement `scan_for_injected_instructions`**
Detect suspicious injection phrases (e.g. `ignore previous`, `system override`, `disclose`, `bỏ qua`, `reveal private`) in text and mark `detected=True`.

- [ ] **Step 2: Implement `redact`**
Scan for learner PII or private score disclosures (e.g. matching `failed the mid-term`, private flags) and redact or flag.

- [ ] **Step 3: Test guardrails in isolation**
Run: `.venv/bin/python agent/guardrails.py`
Expected: Demo asserts pass with stubs replaced by functioning checks.

---

### Task 4: Deck Validation & Lineup Optimization

**Files:**
- Modify: `deck/deck.json`, `deck/lineup.json` (if needed)
- Test: `validate_deck.py`

**Interfaces:**
- Consumes: `deck/deck.json`, `deck/lineup.json`, world corpus `kit/world/df8c55dabb35/`.
- Produces: Validated deck conforming to 14 cards, 10 attacks + 4 blanks, layer & class distribution rules.

- [ ] **Step 1: Run `make validate`**
Run: `make validate`
Expected: PASS (0 failing checks).

- [ ] **Step 2: Ensure all attack cards fire effectively on real corpus**
Verify card triggers and lineup ordering against bots.

---

### Task 5: End-to-End Sparring & Conformance Check

**Files:**
- None (verification only)
- Test: `spar.py`, `tests/`

- [ ] **Step 1: Spar against all three bots**
Run: `make spar BOT=rookie AS=all`
Expected: Victory (you > rookie).
Run: `make spar BOT=operator AS=all`
Expected: Victory (you > operator).
Run: `make spar BOT=adversary AS=all`
Expected: Victory (you > adversary, reversing previous 0-78 loss).

- [ ] **Step 2: Run public test suite**
Run: `make test`
Expected: All tests pass.

- [ ] **Step 3: Run make doctor readiness check**
Run: `make doctor`
Expected: PASS.
