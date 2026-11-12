# Self-Governing Queue — Deployment Blueprint

Directive: Kam, 2026-11-12 15:19 PT — "remove the limitations and just deploy this how it looks, rebuild the system in this way, provide it to the chief of staff to implement, and then just have it run."

Implementer: **Chief of Staff (you, if you are reading this as your task)**. Verifier: OpenClaw (board operator). Authority: this file is the charter.

## 1. What was decided (do not re-litigate)

- The coordinator-dispatch role is **removed**. The DAG has no coordinator node.
- The six-cap global admission limit is **removed**. WIP self-limits via per-seat concurrency (1 task per staff agent, native).
- A deterministic **queue-mapper script** is the only dispatcher: reads the Edit Queue + live Paperclip state on a 30-min cron; creates/assigns tasks only where missing; skips when nothing to do; exits non-zero into the inbox on anything it cannot reconcile.
- Reviewer seat routes `~` (operator). Formal per-seat GitHub reviews are deferred until identities exist (not this build).
- Budget ceilings per agent: raise to 50000 cents (the subscription ceiling stance; these are shadow-metered and not a real spend limit).
- Everything else in the pipeline is unchanged: T7 media storage, encode lock, privacy masking, source preservation, publication boundaries (final-gate-only), Obsidian Content Calendar as the read surface.

## 2. Target architecture (verified against the live instance)

```
Obsidian Edit Queue (Kam's topic approvals land here)
        │  every 30 min
        ▼
queue_mapper.mjs  ── deterministic · token-free · idempotent ──►  Paperclip task created + assigned
        ▼                                            (assignment wake fires the staff agent)
Staff seat runs → closes task with artifact path → next sweep assigns the next stage
```

Stage → staff map (the column schema):

| Stage (Edit Queue column) | Staff agent | Produces |
|---|---|---|
| transcript-selection | Script Writer | exact script + word-timestamp beat map |
| rough-render | Video Editor | splice map → rough render → machine checks |
| ROUGH-REVIEW (gate) | Kam (local-board card) | verdict: approve / feedback / archive |
| covers-captions | Graphic Designer | caption plan + 9:16 covers |
| platform-copy | Script Writer | platform copy ×6 |
| finishing-render | Video Editor | finishing render (captions + SFX) |
| motion (flagged only) | Motion Graphics | motion pieces |
| finishing-qa | Video Editor (check run) | independent finishing QA report |
| FINAL-REVIEW (gate) | Kam (local-board card) | verdict: approve / feedback |
| scheduled | — (publish automation) | gg-daily-content-publish |

Parallel rule: after ROUGH-REVIEW approve, covers-captions and platform-copy may dispatch in the same sweep (different agents, no conflict). finishing-render waits for covers-captions close. Everything serializes through the single encode lock (`encode_lock.py`) — no new locking logic needed, the lock self-guards.

Hardware facts (verified): M2, 8 cores, 24 GB. Agents are CLI/API-bound (no GUI apps, no DaVinci — grep-verified). Four staff agents + one encode fit with headroom. If thermals ever throttle, the encode lock already serializes the only heavy workload.

## 3. Build list (in order)

1. **`shorts-playbook/queue_mapper.mjs`** — Node stdlib only. Inputs: Content Calendar EDITS QUEUE rows (read the same cells `obsidian_update.py` writes), Paperclip REST (`GET /api/companies/{id}/issues`, `POST .../issues` with `assigneeAgentId`). Rules: row in stage X owned by staff S with no open task → create + assign; row with open task → skip; gate stage (ROUGH-REVIEW/FINAL-REVIEW) → assign `local-board` (a card arrives to Kam); nothing to do → exit 0 silently; unreconcilable row → exit 1 with the reason on stderr (inbox alert).
2. **Instruction replacement** — replace each staff agent's layered instruction block with the thin contract: role summary + SOP pointer + one-wake semantics ("you wake on assignment; close with the artifact path; never park; escalate by comment") + escalation rule (breaking problem → comment + mark task broken; never silently idle). Preserve the SOP files themselves — they are unchanged.
3. **Reviewer seat → `~`** and remove the six-cap prose wherever it is still active (queue-flow.md amendment V3 noting the Nov 12 directive; older cap text stays as history).
4. **Mirror duty** — after any task state change the mapper caused, invoke `obsidian_update.py` so the Content Calendar row reflects it.
5. **Schedule the mapper** — OpenClaw automation, script kind, cron `*/30 * * * *`, accountId default, failureAlert after 1 → Telegram.
6. **Disable the old coordination layer** — remove/disable the monitors and dispatch duties on GRI-163 (keep the issue + history; do not delete). Keep GRI-163 as an archive pointer.

## 4. Acceptance criteria (OpenClaw verifies each before "done")

1. Empty-queue sweep exits 0, creates nothing.
2. Test row in a staff column → task created + assigned → staff run fires within one sweep.
3. A full fake piece passes: script → rough → gate card to Kam → covers/copy → finishing → QA → gate card → scheduled. Kam participates at both gates.
4. A deliberately broken task → mapper exit 1 + inbox alert + task marked broken.
5. Zero coordinator-dispatch runs during the test window (all wakes are assignment wakes).
6. Obsidian rows update after every transition.
7. No staff agent exceeds 1 concurrent task; encode lock respected (never two encodes).
8. Kam performs both gate reviews without opening the Paperclip UI.

## 5. Rollback

Disable the mapper automation (one toggle). The old coordination layer stays on disk and in git history; the GRI-163 root and monitors can be re-enabled. Staff instruction contracts revert via git.

## 6. Out of scope (later, after stability)

Phase-A simplification (auto-transcribe on import, auto-Jey on transcript) — separate build. Per-seat GitHub identities — separate decision. CoS retirement decision — after 2 weeks of exception-rate data.

## 7. Guardrails (unchanged, binding)

No publishing outside an approved final gate. No external contact. Media stays on T7. Source media never modified. Privacy masking rules before any export leaves the machine. Do not exceed the encode lock. Do not clear human gates.
