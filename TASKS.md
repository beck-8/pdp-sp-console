# EPIC: Curio PDP SP Console — milestones

Delivers `PRD.md`. Each milestone below files as one issue; the engineer who takes it owns the finer breakdown. Ordering: M0 first, then M1→M2→M4 build on each other; M3, M5, M6 are independent of that chain.

## M0 — Console shell + reduced menu
A navigable "SP Console" group in the existing web UI, and a config-driven reduced menu for PDP-only deployments.

New `pages/console/` routes and the shared components (KPI tile, status pill, health banner, drawer, table, small SVG charts) on the existing no-compile Lit stack. Menu filtering goes into `web/static/ux/curio-ux.mjs` — `renderMenu()` is a single hard-coded list today, so the reduced menu doesn't exist until we build it. No backend changes.

## M1 — Datasets & Proving, read-only
Proving visibility inside Curio: dataset list with deadline countdowns, a challenge-window timeline, dataset detail with pieces.

The data is already written by `tasks/pdpv0`: `pdp_data_sets` (`proving_period`, `challenge_window`, `prove_at_epoch`), `pdp_data_set_pieces`, `pdp_piecerefs`. What's missing is read-only webrpc methods over those tables. Strictly read-only — no prove-now / terminate buttons in this slice; those need their own confirm UX and belong to a later milestone.

Depends on M0.

## M2 — Proving history & faults
Proving success rate, weekly proofs-vs-faults, per-dataset fault counts, and alert rules for entered-window / faulted-period.

Curio keeps no record of past proving periods (`pdp_data_sets` rows are overwritten each period), so this needs a new `pdp_proving_history` table written from `tasks/pdpv0` — `ProveTask`, `NextProvingPeriodTask`, and the failure path in `failure_handling.go`. History starts accruing at deploy; the UI has to say so in its empty state.

Depends on M1.

## M3 — Payments & FOC
Registry status, balances, rails, and settlement history on one page.

Most of the machinery exists: registry cards reuse `FSRegistryStatus` / `FSUpdatePDP` / `FSDeregister` (`web/api/webrpc/pdp.go`); settlement history reads `filecoin_payment_transactions` + `message_waits_eth`, which `lib/filecoinpayment` and `tasks/pay` already write; rail discovery and state live in `lib/filecoinpayment/utils.go`. The new work is exposing these through webrpc and the page itself.

One action only in this milestone: **Settle now**, which kicks the existing `tasks/pay` SettleTask. Send-to-treasury (moving USDFC out) is explicitly a follow-up, not part of M3.

Depends on M0; independent of M1/M2.

## M4 — Business dashboard
The health banner, the open-issues list, and the income / gas / proving KPIs and charts.

The open-issues list reads the alert lifecycle from curio PR #1332 (open at time of writing): active conditions from `alert_conditions`, recent one-shot events from `alert_history` (`kind='event'`), with the existing mute mechanism. It shows final failures and active conditions only — attempts that recover on retry never appear (they live in M6's task table instead). Two things to mind: this slice depends on that PR merging, and the PR's task-failure query still counts every failed attempt, so the final-failure filter (retries exhausted — task row deleted from `harmony_task`) needs to land there or in our read RPC. The banner collapses to "All clear" when the list is empty.

KPIs and charts need M2 (proving numbers) and M3 (settlement reads). Probably wants a small daily rollup rather than scanning tx receipts on every page load — the engineer taking this decides.

## M5 — Service Setup
The auto-detected getting-started checklist plus endpoint/products cards.

Most checks read existing state: chain connection, `eth_keys role='pdp'`, storage paths, `FSRegistryStatus`, `[HTTP]` config including `DelegateTLS`. Genuinely new: an outbound reachability self-probe of the public `/pdp/ping`, and a PDP self-test flow (create → prove → clean up) — that one touches `tasks/pdpv0` and should be scoped with whoever owns that code.

Depends on M0.

## M6 — Storage & Tasks
Paths, task failures, and node views re-composed in console style, plus the pdpv0 ingest funnel (`pdp_piece_uploads` / `pdp_piece_pulls` → `parked_pieces` → `pdp_piecerefs` → `pdp_data_set_piece_adds` → save-cache → IPNI), with per-stage failure counts.

The task table splits **Failed** (retries exhausted — alerts) from **Recovered** (attempt failed, retry succeeded — no alert, but repeated recoveries point at an underlying issue). `harmony_task_history` already keeps one row per attempt with `task_id` and `result`, so this is a query change, not new collection.

Depends on M0.

## Out of scope for this epic
Serve-latency instrumentation · USDFC send-to-treasury · prove-now / terminate actions · mk20 pipeline union · PoRep console pages · the new-SP onboarding funnel from #1320.
