# PRD — Curio PDP SP Console

*Breakdown of filecoin-project/curio#1320, PDP-first slice. For team review.*
*Companions: `mock/index.html` (interactive fake-data wireframe — open in any browser; screenshots in `mock/screenshots/`) · `TASKS.md` (the epic: M0–M6 issue breakdown).*

> 🗺️ **TL;DR** — We're building the **operations console for PDP storage providers, inside Curio itself** — not a general UI overhaul and not another external dashboard. One place where a live SP answers four questions: *Will I prove on time? Am I getting paid? Is the machine healthy? Is my service configured and reachable?* The first slice (proving & datasets) is derivable entirely from tables Curio already has — ship it first, layer income/wizard views on top.

## The problem

Curio's UI was built for PoRep; the 3–5 SPs now running production PDP get almost nothing from it. The data that answers their daily questions — proving deadlines, faults, per-dataset pieces, settlement history — **is already in harmonydb but has no UI at all**. Today the answers live in external dashboards, and in practice SPs don't watch external sites: faults get noticed late, income is opaque, and "is my service reachable" is guesswork. There is also no PDP-shaped deployment story: the `make curio-pdp` reduced UI referenced in #1320 doesn't exist — a PDP-only SP gets the full PoRep menu.

## The vision — an ops console, not a page pile

Inside Curio, a console with the Stripe-dashboard / Linear feel #1320 asks for: dense, dynamic, with first-class *"all of these things are well"* summaries — and a needs-attention list when they aren't. It is the **supply side** of Filecoin Onchain Cloud: the filecoin.cloud platform PRD covers how clients buy and get billed ("the platform picks the SP; the user never chooses one" — and "Become a provider is a separate supply-side funnel"); **this console is that supply-side funnel's operating end** — how the picked provider proves, serves, and collects. Same Filecoin Pay data plane, two mirrored views: the client's *bill* is the SP's *revenue*.

## The model — five pages

| Page | Platform-PRD area | The question it answers |
|---|---|---|
| **Dashboard** | Business dashboard | "Is my business OK right now?" — health banner, income vs gas, proving success, open issues |
| **Datasets & Proving** | + operator proving visibility | "Will I prove on time? What am I storing for whom?" — challenge-window timeline, deadline countdowns, faults, piece drill-down |
| **Payments & FOC** | FOC participation | "Am I getting paid? Am I registered correctly?" — registry status, balances, rails, settlements, settle-now |
| **Storage & Tasks** | Storage mgmt & monitoring | "Is the machine healthy?" — paths, pdpv0 ingest funnel, failed tasks + restart |
| **Service Setup** | Service configuration | "Is my service configured and reachable?" — auto-detected getting-started checklist, endpoints, products |

Cross-cutting: a health chip on every page ("PROVING ON SCHEDULE" / degraded), alert-count nav badges, drawer drill-downs, explorer out-links for every on-chain object.

## Core decisions

1. **PDP first.** PoRep pages stay where they are; no mixing until the PDP console has proven itself with live SPs.
2. **Part of the Curio package.** No separate service to deploy or operate — the console ships in the existing web UI and its embed FS. "Deploying sp-console" = running Curio.
3. **Reduced menu via config, not build.** A `UIMode` knob filters the menu to console-only for PDP deployments — this *creates* the "curio-pdp reduced UI" #1320 assumes; a build tag can later default it.
4. **Keep the no-compile, minimal-dependency frontend.** Lit ES modules, no bundler, no new chart library — this JS can move SP funds; auditability is a feature.
5. **Show only data we own or can read on-chain.** No third-party measurement feeds. Latency = our own handler timings; income = our own settlement txs; proving = our own task outcomes. (Decided 2026-07-02.)
6. **pdpv0 is the source of truth.** All queries target the pdpv0 tables (`pdp_data_sets`, `pdp_data_set_pieces`, `pdp_piece_uploads`, …). The mk20 path (`pdp_pipeline`) is a different product path current SPs don't use — union later if needed.
7. **Vocabulary aligned with the platform PRD.** Same protocol→product mapping, mirrored to the supply side (below). Product words in the UI; raw rails/epochs one click deeper.
8. **Ship in slices.** Every milestone is independently useful to a live SP; feedback between slices.
9. **Alerts are loud, but only for real problems.** The dashboard's open-issues list shows final failures and active conditions only — a task attempt that later succeeds on retry never appears there. This is exactly the semantics of the alert lifecycle merged upstream on 2026-07-03 (`alert_conditions` for active conditions; `alert_history.kind = event|condition` for one-shots and resolved history; per-alert mutes) — the console reads it, it doesn't reinvent it. Recovered attempts stay visible one level down, on Storage & Tasks, for whoever wants to chase an underlying cause. (Decided 2026-07-07.)

## Vocabulary — protocol → SP product

Mirror of the platform PRD's billing mapping (client side ↔ SP side of the same objects):

| Protocol (today) | Client sees (platform PRD) | SP sees (this console) |
|---|---|---|
| Rate-based rail | Recurring charge — storage / mo | **Recurring income** — per client, per rail |
| `settleRail` | (background) | **Settlement** — USDFC landing in wallet |
| Rail lockup / `endEpoch` | Funded until \<date\> | **Payer funded until** — early warning a client is running dry |
| `possessionProven` / missed window | (SLA they rely on) | **Proven this period / faulted period** |
| ServiceProviderRegistry entry | (invisible — platform picks the SP) | **My registration & offering** (price, sizes, service URL) |
| Proof + settlement gas | (not their problem) | **Cost of revenue** — gas next to income, always |

## Revenue legibility

The platform PRD prices FWSS v1.3.0 for clients ($2.50/copy/TiB/mo storage, $0.024/mo proving per dataset, one-time create/add fees). The console presents the SP mirror: income decomposed along those same line items (storage rate × data under proof; per-dataset fees; one-time ops), minus gas — so an SP's dashboard and a client's bill are explainable in one vocabulary. (Exact fee routing per line item to be confirmed against the contract before the income-breakdown widget lands; v1 shows settled totals + gas, which need no assumption.)

## Rollout

Everything an SP sees in M1 is derivable from tables Curio already writes — no new collection to ship the first useful slice.

- **M0** console shell + `UIMode` reduced menu →
- **M1** Datasets & Proving, read-only (the #1 ask) →
- **M2** proving history & faults (one new table) →
- **M4** business dashboard.
- **M3** Payments & FOC and **M5** Setup wizard run in parallel; **M6** Storage & Tasks is filler.

Milestone details and dependencies: `TASKS.md` (**EPIC: Curio PDP SP Console** — each milestone files as one issue; the engineer who takes it owns the finer breakdown).

## Open questions

1. `UIMode` via config vs. build tag for the reduced menu — config proposed (works for existing binaries, no packaging change).
2. USDFC send-to-treasury action in v1, or read-only + settle-now first? Proposal: read-only + settle-now in M3, fund-moving send behind a follow-up with explicit confirm UX. (Change-payee is settled: not possible in the registry contract — see Appendix A §3.)
3. Font addition (Instrument Sans, self-hosted) vs all-JetBrains-Mono.

**Resolved during review (2026-07-02):** latency = own measurements only (A§1); no change-payee — contract has no such function, verified against the registry ABI (A§3); ingest funnel sources = pdpv0 tables, not mk20 (A§4); single-node compact rendering; TLS reverse-proxy checklist incl. real-client-IP headers `X-Forwarded-For`/`X-Real-IP` (A§5).

---

# Appendix A — Page specs → data mapping

Legend: ✅ = RPC/data exists today · 🔶 = derivable from existing tables, needs new webrpc method · 🔴 = needs new collection (poller task / new table / chain read).

## A§1 Dashboard

| Widget | Data | Status |
|---|---|---|
| "All systems operational" banner + top-bar chip | composite: all datasets proven-or-scheduled ∧ no failed settle ∧ wallet runway > threshold ∧ no critical alerts | 🔶 one `PDPHealthSummary` RPC over existing tables |
| KPI: data under proof | `pdp_data_set_pieces` sizes join active `pdp_data_sets` | 🔶 |
| KPI: proving success 30d | needs proof-outcome history | 🔴 `pdp_proving_history` table (Appendix B) |
| KPI: net income 30d | settlement amounts | 🔴 read receipts/logs of our own `settleRail` txs (`filecoin_payment_transactions` has the tx hashes) |
| KPI: gas 30d | receipts of proof + settlement txs we sent (`message_waits_eth`) | 🔶 |
| Proofs vs faults weekly chart | `pdp_proving_history` | 🔴 |
| Earnings vs gas daily chart | same sources as KPIs | 🔴 |
| Latency (three durations) | **Own data only:** `/piece` and `/ipfs` request→last-byte histograms in our handlers, plus addPieces tx confirmation time (task start → receipt). Shown as two mini-charts (serve in ms; addPieces in s — different scales, never one axis). Client-perceived network latency stays external (dealbot), out of scope. | 🔴 (phase 2) |
| Open-issues list | `alert_conditions` (active conditions: system/subsystem, message, `created_at`, `repeat_count`) + unresolved `alert_history` rows with `kind='event'`, plus the existing mute mechanism. Final failures and active conditions only — retried-and-recovered attempts are excluded by the alert lifecycle itself (core decision 9). Empty list ⇒ the banner reads "All clear". Rows deep-link to the page that owns the problem. | 🔶 read RPC over the alert-lifecycle tables |

## A§2 Datasets & Proving

| Widget | Data | Status |
|---|---|---|
| KPIs (active, proven x/y, faults 7d, next deadline) | `pdp_data_sets` (`prove_at_epoch`, `challenge_window`, `proving_period`) + chain head. NB: proving is **per dataset** — several windows can be due at (nearly) the same time; the "next deadline" tile shows the nearest plus "N more due within X h", and the timeline strip shows all windows incl. overlaps (gas-spike warning candidate when many cluster). | 🔶 `PDPDataSets` RPC |
| Challenge-window timeline (next 24 h) | same | 🔶 |
| Datasets table w/ proving status pill, deadline countdown, faults, rail | `pdp_data_sets` + client/rail from FWSS dataset-created data + faults history | 🔶 (+🔴 faults) |
| Dataset drawer: pieces list | `pdp_data_set_pieces` (+ `pdp_piecerefs` sizes) | 🔶 |
| Dataset drawer: proving params, last proven | `pdp_data_sets` + `pdp_proving_history` | 🔶/🔴 |
| Dataset drawer: rail + rate | dataset→rail mapping via FWSS dataset-created info; rate from FilecoinPay `getRail` | 🔴 chain read |
| Actions: prove now / terminate | **Out of M1 — future milestone.** M1 is strictly read-only. These are operational actions (terminate especially is high-risk) and need their own confirm UX before they ship; the underlying paths exist (task kick, `pdp_delete_data_set` lifecycle). | later |

## A§3 Payments & FOC

| Widget | Data | Status |
|---|---|---|
| Registry card + update/deregister | ✅ `FSRegistryStatus`, `FSUpdatePDP`, `FSDeregister` (webrpc/pdp.go) — reuse as-is |
| Balances: FIL gas wallet | ✅ wallet RPCs; runway = balance / observed burn | 🔶 |
| Balances: USDFC earned / accrued-unsettled | ERC-20 `balanceOf` + per-rail accrual; the rail-discovery and rail-view logic already exists in `lib/filecoinpayment/utils.go` (used by `tasks/pay`) — expose it through webrpc | 🔶 |
| Rails table + funded-until warning | same rail-view chain reads (rate, settledUpto, endEpoch, lockup, payer) | 🔶 |
| Recent settlements table | `filecoin_payment_transactions` + `message_waits_eth` — both already written atomically by `lib/filecoinpayment` on every `settleRail`, receipts tracked by `tasks/pay/watcher.go` | 🔶 |
| Action (M3): settle now | kick the existing `tasks/pay` SettleTask | 🔶 |
| Action (follow-up, **not** M3): send-to-treasury | ordinary USDFC wallet send behind an explicit confirm UX. **No change-payee action ever**: verified against the deployed ServiceProviderRegistry ABI — only `updateProviderInfo(name, description)` and `updateProduct` exist; `payee` is fixed at `registerProvider`. "Change wallet" is served by moving earnings out with a wallet send; changing payee itself would mean deregister + re-register (new provider ID) — not a console action. | 🔴 send endpoint |

## A§4 Storage & Tasks

Storage paths, tasks, nodes are **rearrangement of existing components**: `StoragePathList`/`StoragePathsSummary`, `HarmonyTaskStats`/`ClusterMachines`, restart actions. ✅ with light re-skinning.

The **task-failures table splits two outcomes** that today read as one number: **Failed** (retries exhausted — this is what raises an alert) vs **Recovered** (an attempt failed, a retry succeeded — never alerts, but listed here because repeated recoveries point at an underlying issue). Both are derivable from `harmony_task_history`, which keeps one row per attempt with `task_id` and `result`: recovered = a false row followed by a true row for the same task id; failed = last row false and the task gone from `harmony_task`. 🔶 one query change to the existing task-summary RPC.

The **ingest pipeline funnel** is new and must be built from the **pdpv0 tables** (NOT `MK20PDPPipelines` — that reads the mk20 `pdp_pipeline`, a different product path unused by current pdpv0 SPs): uploaded/pulled (`pdp_piece_uploads` / `pdp_piece_pulls`) → bytes parked (`parked_pieces`) → commP verified (`pdp_piecerefs`) → addPieces tx confirmed (`pdp_data_set_piece_adds`) → saveCache built → IPNI indexed. Per-stage counts + failure badges (joined against `harmony_task`). 🔶 new `PDPv0Pipeline` RPC. Node view renders as a single compact row for the common single-node PDP deployment and expands to the cluster table only when >1 machine is registered.

## A§5 Service Setup

| Step | Detection | Status |
|---|---|---|
| Chain connection | existing chain RPC health | ✅ |
| PDP wallet attached | `eth_keys role='pdp'` (✅ `ListPDPKeys`) | ✅ |
| Wallet funded | balance ≥ threshold | ✅ |
| Storage attached | storage paths exist & healthy | ✅ |
| PDP flow verified | 🔴 new self-test tool (create dataset → prove → cleanup), per #1320 "Verify PDP flow" |
| Registered on FOC | ✅ `FSRegistryStatus` |
| Domain configured | config HTTP domain set | ✅ read config |
| TLS mode | `[HTTP] DelegateTLS` already exists: managed ACME **or** TLS terminated at the SP's reverse proxy. When delegated, show a reverse-proxy checklist: forward `Host`, `X-Forwarded-Proto`, and the real client IP (`X-Forwarded-For` / `X-Real-IP` — Curio should log/rate-limit on this, not the proxy's IP; behind Cloudflare it's `CF-Connecting-IP`), disable request buffering (streaming uploads), unlimited body size, long read timeouts, WebSocket upgrade for the UI. | ✅ read config |
| Reachability verified | 🔴 outbound self-probe of `https://<domain>/pdp/ping` (also validates the proxy config end-to-end) |

Plus read-only endpoint/products summary cards (✅ config + `ListProducts`).

# Appendix B — Data collection we must add (backend prerequisites)

1. **`pdp_proving_history`** — Curio keeps no record of past proving periods (rows in `pdp_data_sets` are overwritten each period; harmony_task_history ages out). Add a small table written from `tasks/pdpv0` — `ProveTask`, `NextProvingPeriodTask`, and the failure path in `failure_handling.go`: `(data_set_id, period_start_epoch, prove_epoch NULL, faulted bool, gas_used, tx_hash)`. Powers: success-30d KPI, weekly chart, per-dataset fault counts, "last proven". *Alternative rejected:* reading PDPVerifier events from the chain node — the attached node may prune state and it couples the UI to event indexing; writing our own outcomes is cheap and exact.
2. **Settlement/gas aggregates** — join `filecoin_payment_transactions` and proof txs with `message_waits_eth` receipts; likely a small daily-rollup poller for the 30d charts instead of scanning per page load.
3. **FilecoinPay/ERC-20 chain reads** — new webrpc methods exposing the rail-discovery/rail-view logic that already exists in `lib/filecoinpayment/utils.go`, plus ERC-20 `balanceOf`. No indexing; live reads with short cache.
4. **(Phase 2) serve latency** — request→last-byte histograms in the `/piece` and `/ipfs` handlers + addPieces confirmation time.

# Appendix C — Technical approach (frontend)

- **Same stack, no new deps**: LitElement `.mjs` pages under `web/static/pages/console/{dashboard,datasets,payments,storage,setup}/`, served by the existing embed FS; data via existing `/api/webrpc/v0` `RPCCall`.
- **New shared components** (`web/static/ux/console/`): KPI tile, status pill, health banner, drawer (reuse `components/Drawer.mjs`), sortable table, and a small dependency-free SVG chart helper (bars/lines/timeline — the mock's ~120-line implementation is the spec; avoids adding Chart.js to fund-touching pages, though Chart.js is already used elsewhere and is an acceptable fallback).
- **Design tokens**: extend `ux/main.css` with the mock's tokens (surfaces, status colors, validated chart palette `#3987e5 #1baf7a #c98500 #9085e9`) rather than forking a second theme. JetBrains Mono for data; Instrument Sans (self-hosted like existing fonts) for UI text — or all-mono if preferred (open question #3).
- **Menu**: "SP Console" group in `curio-ux.mjs`; server-provided `UIMode` (e.g. `[UI] Mode = "pdp" | "full"`) filters `renderMenu()`.

# Appendix D — Codebase recon (what grounded this PRD)

- **The data is already in harmonydb; the UI doesn't show it.** Production path is **pdpv0** (`tasks/pdpv0/`, driven by the `/pdp` API + client SDK): `pdp_data_sets` (`proving_period`, `challenge_window`, `prove_at_epoch`), `pdp_data_set_pieces`, `pdp_piece_uploads`/`pdp_piece_pulls`/`pdp_piecerefs`/`pdp_data_set_piece_adds` (ingest), `filecoin_payment_transactions` (settlement txs), `pdp_delete_data_set` (termination). None have UI. The PDP page today shows services/keys, registry status, a deal list, and the **mk20** pipeline — a different product path.
- **No PDP settlement/rail UI exists** anywhere (the only settlement UI is the SNARK-market one).
- **`make curio-pdp` reduced UI doesn't exist** — no build tag, no page filtering; the menu is one hard-coded list in `web/static/ux/curio-ux.mjs`.
- **Existing external SP dashboards** proved the UX patterns: proving weekly proofs-vs-faults, fault drill-downs, rails/settlement tables, dataset modals. This console internalizes those views with Curio's own data (Curio additionally knows task state, storage, config, and wallet keys that external observers can't see; they know chain history Curio doesn't index).
- A downstream single-binary PDP deployment independently converged on nearly the same page set (Overview / Datasets / Rails / Wallets / Tasks / Storage / Alerts) — good IA validation.
