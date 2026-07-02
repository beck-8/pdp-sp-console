# EPIC: Curio PDP SP Console — Issue Breakdown

*Delivers `PRD.md`. Each milestone below files as one GitHub issue under the epic; checkboxes are its PR-sized subtasks.*

Sliced so each milestone ships alone and delivers something an SP can use immediately. Order = value for the 3–5 live PDP SPs. Estimates are engineering-days, single engineer, incl. review churn.

## M0 — Console shell + reduced menu  *(~3 d, unblocks everything)*
- [ ] `web/static/pages/console/` route group + "SP Console" menu group in `curio-ux.mjs`
- [ ] Shared components: KPI tile, status pill, health banner, SVG chart helper, drawer wrapper, sortable table (`web/static/ux/console/`)
- [ ] Design tokens added to `ux/main.css` (surfaces, status colors, validated chart palette)
- [ ] `UIMode` config knob + `renderMenu()` filtering → the de-facto `curio-pdp` reduced UI
- Deliverable: empty-but-navigable console; PDP-only menu mode. No new backend.

## M1 — Datasets & Proving (read-only)  *(~5 d — the #1 SP ask)*
- [ ] webrpc `PDPDataSets` / `PDPDataSet(id)` / `PDPDataSetPieces(id, page)` over the pdpv0 tables (`pdp_data_sets`, `pdp_data_set_pieces`, `pdp_piecerefs`) + chain head
- [ ] Datasets page: KPI row, datasets table w/ proving-status pill + deadline countdown, challenge-window timeline strip
- [ ] Dataset drawer: identity, proving params, pieces (paged)
- Deliverable: SPs see "will I prove on time" inside Curio for the first time.
- Depends: M0.

## M2 — Proving history & faults  *(~4 d, backend-heavy)*
- [ ] Migration `pdp_proving_history` + writes from ProveTask / NextProvingPeriodTask / fault path (upstream `pdp/` package so downstream PDP builds inherit it)
- [ ] webrpc `PDPProvingHistory(range)`; faults 7d/30d roll into `PDPDataSets`
- [ ] Dashboard proofs-vs-faults weekly chart + proving-success KPI; per-dataset "last proven" + fault counts in M1 views
- [ ] Alert rule: dataset entered challenge window / period faulted → alerts framework
- Depends: M1. (History accrues from deploy day — say so in the UI empty state.)

## M3 — Payments & FOC page  *(~6 d)*
- [ ] Move existing registry card (`FSRegistryStatus` + update/deregister) onto the page
- [ ] webrpc chain-reads: `PDPRails` (`getRailsForPayeeAndToken` + `getRail`), `PDPBalances` (FIL + USDFC `balanceOf` + accrued-unsettled), short TTL cache
- [ ] webrpc `PDPSettlements` over `filecoin_payment_transactions` + receipts; settlement-health card
- [ ] "Settle all now" (kick settle task) — **read-only otherwise**; USDFC send-to-treasury deferred (open question #2; change-payee ruled out, contract doesn't support it)
- Depends: M0 only — can run in parallel with M1/M2.

## M4 — Business dashboard  *(~4 d)*
- [ ] Daily rollup poller: settled USDFC + gas (proof/settle txs via `message_waits_eth`)
- [ ] Dashboard KPIs (income, gas, data under proof) + earnings-vs-gas chart + needs-attention list + `PDPHealthSummary` top-bar chip
- Depends: M2 (proving KPIs), M3 (settlement reads).

## M5 — Service Setup wizard  *(~5 d)*
- [ ] Read-only checklist w/ auto-detection (chain, wallet, funding, storage, registry, domain) — all existing data
- [ ] Reachability self-probe of `https://<domain>/pdp/ping`
- [ ] PDP flow self-test task (create dataset → prove → cleanup) + "Re-run test"
- [ ] Endpoints/products summary cards
- Depends: M0. Self-test task is its own PR.

## M6 — Storage & Tasks  *(~3 d, parallel)*
- [ ] Compose existing RPCs (paths, failed tasks + restart, nodes single-row/cluster) into the console page style
- [ ] New `PDPv0Pipeline` RPC: ingest funnel from pdpv0 tables (`pdp_piece_uploads`/`pdp_piece_pulls` → `parked_pieces` → `pdp_piecerefs` → `pdp_data_set_piece_adds` → saveCache → IPNI), failure counts joined vs `harmony_task`
- Depends: M0.

## Later / explicitly out of v1
- Serve-latency instrumentation (`/piece`, `/ipfs` handlers + addPieces confirmation time — own data only, decided) · USDFC send-to-treasury tx · mk20 pipeline union (if that path gets used) · PoRep console pages · CF-tunnel onboarding & the rest of #1320's new-SP funnel · i18n pass (keep strings translation-friendly from M0).

**Critical path:** M0 → M1 → M2 → M4 (≈16 d). M3, M5, M6 parallelize — two engineers land everything in ~3 weeks.

**Suggested split (two engineers):** one takes M1+M2 (harmonydb + proving domain), the other M3+M5 (chain-read/payments + wizard); M0 pair-designed first since components + tokens set the visual bar.
