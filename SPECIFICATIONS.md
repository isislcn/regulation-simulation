# SPECIFICATIONS: Proposed Regulation Simulation ("Reg-Intel Inject")

**Companion to:** CONCEPT-IDEA.md
**Target platform:** Claude Artifacts (self-contained HTML page published as an artifact)
**Version:** v1.0 draft, 2026-11-23

---

## 0. Assumptions and open questions

| # | Assumption | Confirm with team |
|---|---|---|
| A1 | **Decided: ships as a tab** in the existing simulation demo. Tab order: Overview · Simulation · Inject library · BOM import · Decision memo. The Overview tab is specified in §13 and built in `overview-tab.html`. | What is the current sim's state model, so the Overview scenario controls can share `appState`? |
| A2 | The artifact has **no live network access**. Sayari, FCC and Congress data come in as files (CSV/JSON) that students export and import. | Do we want a Claude-in-artifact capability to parse pasted intel text into an inject? (Optional, see §9.) |
| A3 | The demo BOM is illustrative. Real parts come from the student Sayari work. | Which Sayari export fields will students actually have? (§5.3 proposes a mapping.) |
| A4 | Persistence: in-memory by default. Use an Artifact data capability if one is available, so authored injects and imported BOMs persist across viewers. Otherwise use `localStorage` (wrapped in try/catch) for per-viewer drafts only. | Is a shared store needed (class-wide injects), or are per-student copies enough? |

---

## 1. Architecture

```
┌──────────────────────────── Artifact page (single HTML) ────────────────────────────┐
│                                                                                      │
│  [Inject Library] ─┐                                                                 │
│  [Inject Author]  ─┼─► Inject[] ──┐                                                  │
│                    │              ▼                                                  │
│  [BOM Import] ───► Part[] ──► RULE ENGINE (pure fn) ──► Exposure[] ──► Views         │
│  [Design Model] ─► Zones/Phases ─┘   evaluate(parts, injects, date, levers)           │
│                                         ▲                                            │
│  [Mitigation Levers] ───────────────────┘                                            │
│                                                                                      │
│  Views: Timeline · Heatmap (zone × component class) · Part table · Trace panel ·     │
│         Design-Considerations list · Decision-memo export                            │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

- **Single file:** markup, CSS and JS in one HTML document. No build step and no CDN dependencies. Canned data is embedded as `<script type="application/json">` blocks.
- **Rule engine is a pure function.** It must be deterministic and unit-testable, with no DOM access.
- **State:** one `appState` object: `{ injects, parts, design, levers, simDate, selectedId }`. Each change triggers a re-render.

---

## 2. Data model

### 2.1 Inject

```json
{
  "id": "INJ-OPTICS-3_2T",
  "title": "Reported FCC curb on new Chinese optical transceiver models",
  "actor": "FCC",
  "instrument": "draft_rule",
  "evidence_level": "REPORTED",
  "evidence_note": "Reuters exclusives Aug 4, Nov 10, Nov 17 2026; 3.2T phase-in and 65% figure from Morgan Stanley readout. No FCC text.",
  "sources": [
    {"label": "Reuters via secondary coverage, Nov 17 2026", "url": "https://jpozo.substack.com/p/pozos-photonics-intelligence-briefing-047", "primary": false}
  ],
  "effective": { "type": "date_range_estimate", "start": "2028-01-01", "end": "2029-12-31" },
  "applies_to": "new_authorizations_only",
  "scope": {
    "all": [
      { "field": "component_class", "op": "in", "value": ["optical_transceiver"] },
      { "field": "manufacturer_hq_country", "op": "in", "value": ["CN"] },
      { "field": "speed_gbps", "op": ">=", "value": 3200 }
    ]
  },
  "exemptions": [
    { "id": "US_BOM_65", "label": ">=65% US BOM value",
      "when": { "field": "us_bom_value_pct", "op": ">=", "value": 65 } }
  ],
  "consequence": "NEW_AUTH_BLOCKED",
  "zones": "all",
  "design_considerations": ["DC-OPTICS-GEN", "DC-SKU-LOCK", "DC-BOM-CONTENT", "DC-ARCH-ALT"]
}
```

**Enums**
- `evidence_level`: `ENACTED | ADOPTED | PROPOSED | REPORTED | ANALYST`
- `instrument`: `final_rule | covered_list_entry | conditional_approval | bill | entity_list | draft_rule | report`
- `applies_to`: `new_authorizations_only | procurement | installed_base | import`
- `consequence`: `NEW_AUTH_BLOCKED | PROCUREMENT_BLOCKED | WATCH | APPROVAL_EXPIRES`
- `effective.type`: `date | date_range_estimate | relative` (for example `{ "type":"relative","anchor":"enactment","offset_years":5 }`, where enactment date is a user-set assumption)
- Scope ops: `in, not_in, ==, !=, >=, <=, exists, intersects` (`intersects` is for list fields such as `entity_lists`)
- Scope combinators: `all`, `any`, `not` (nestable)

### 2.2 Part (BOM line)

| Field | Type | Notes / Sayari mapping |
|---|---|---|
| `part_id` | string | internal |
| `mpn` | string | manufacturer part number |
| `description` | string | |
| `component_class` | enum | `optical_transceiver, aoc, dac, switch, server, nic, dpu, bmc, oob_router, console_server, ups, inverter, bess_pcs, pdu, cooling_controller, other` |
| `speed_gbps` | number? | 400 / 800 / 1600 / 3200 |
| `form_factor` | string? | OSFP, QSFP-DD, LPO, CPO… |
| `brand` | string | name on the box |
| `manufacturer` | string | actual producer |
| `manufacturer_hq_country` | ISO2 | Sayari: entity country |
| `production_country` | ISO2 | Sayari: trade/shipment origin (e.g. TH for InnoLight/Eoptolink Thai plants) |
| `ultimate_parent` | string | Sayari: ownership chain / UBO |
| `entity_lists` | string[] | `FCC_COVERED`, `DOD_1260H`, `BIS_ENTITY`, `NS_CMIC`, `UFLPA`… (Sayari risk flags) |
| `logic_bearing` | bool | per FCC 26-50 definition |
| `sub_components` | Part-ref[] | logic-bearing subcomponents (e.g. mgmt SoC, DSP), used for the component rule |
| `us_bom_value_pct` | number? | needed for the 65% exemption |
| `dsp_vendor`, `laser_vendor` | string? | drives the 65% lever |
| `fcc_id`, `fcc_auth_date` | string/date? | authorized before rule effective date = installed-base safe |
| `conditional_approval_expiry` | date? | for routers/inverters |
| `qty`, `unit_cost_usd`, `lead_time_wks` | number | illustrative in demo |
| `zone_id`, `phase_id` | string | where and when deployed |
| `sayari_entity_id` | string? | traceability back to student research |
| `source_note` | string | student's evidence note |

### 2.3 Design model

```json
{
  "site": "Demo AI Hall – 20 MW",
  "zones": [
    {"id":"Z1","name":"Commercial Hall A","federal_nss":false},
    {"id":"Z2","name":"Federal Enclave","federal_nss":true}
  ],
  "phases": [
    {"id":"P1","name":"Phase 1 – 800G/1.6T fabric","order_date":"2027-03-01","deploy_date":"2027-09-01"},
    {"id":"P2","name":"Phase 2 – 3.2T upgrade","order_date":"2028-09-01","deploy_date":"2029-03-01"},
    {"id":"P3","name":"Refresh – mgmt & power","order_date":"2028-01-15","deploy_date":"2028-06-01"}
  ],
  "assumptions": {"s5548_enactment_date":"2027-12-31"}
}
```

### 2.4 Exposure (engine output)

```json
{ "part_id":"...", "inject_id":"...", "state":"WATCH",
  "effective_on":"2028-01-01", "confidence":0.4,
  "reason":"component_class=optical_transceiver ∧ hq=CN ∧ speed≥3200",
  "exempted_by":null, "trace":[ /* predicate results */ ] }
```

---

## 3. Rule engine

`evaluate(parts, injects, design, levers, simDate) → Exposure[]`

1. **Apply levers** to a cloned parts list (§6). Original data is never mutated.
2. For each inject and each part:
   1. Resolve `effective` to a date. For `relative`, use `design.assumptions`. For a range, use the start date for the state calculation and show the range in the UI.
   2. Skip if `simDate < effective` (state = `CLEAR` for this inject; the timeline still shows the upcoming date).
   3. Evaluate `scope` against the part. For **component-rule injects** (`scope_mode: "contains"`), evaluate against the part **and its `sub_components`**. A match on any subcomponent flags the parent and records which subcomponent caused it.
   4. Zone filter: `zones: "federal_nss"` → only parts in zones with `federal_nss: true`.
   5. Exemptions: if any exemption matches, state = `EXEMPT`, with the exemption recorded.
   6. **Installed-base logic:** if `applies_to = new_authorizations_only`, the part has `fcc_auth_date < effective`, **and** the part's phase is not a *new model*, then state = `WATCH (installed base safe; next revision exposed)` instead of blocked.
   7. Map `consequence` → state, then **cap by evidence level**:

| Evidence | Max state shown | Confidence weight |
|---|---|---|
| ENACTED / ADOPTED | Red (BLOCKED) | 1.0 |
| PROPOSED | Red-hatched ("if enacted") | 0.6 |
| REPORTED | Amber (WATCH) | 0.4 |
| ANALYST | Amber-dotted | 0.2 |

3. **Per-part roll-up:** the worst state across all injects wins. Keep every trace.
4. **Metrics:**
   - `exposed_units`, `exposed_spend_usd` (qty × cost × confidence)
   - `% of fabric optics exposed` by phase
   - `days of runway` = effective date − phase order date
   - `mitigation_cost_delta` and `lead_time_delta` from lever deltas

**Unit tests (inline, run from a hidden `?test` mode):** at least 12 cases, including: a subcomponent match flags the parent; an authorized-before-effective part is WATCH and not blocked; 65% exemption at exactly 65.0 passes; a federal-zone-only inject ignores commercial zones; a REPORTED inject never renders red; a relative date resolves from the enactment assumption.

---

## 4. Canned demo injects (ship in v1)

| ID | Title | Evidence | Effective | Scope (summary) | Consequence |
|---|---|---|---|---|---|
| `INJ-FCC-26-50` | Logic-bearing component rule | ENACTED | 2026-10-13 | any device `contains` a sub-component where `entity_lists ∩ {FCC_COVERED}` | NEW_AUTH_BLOCKED |
| `INJ-FCC-26-50-FNPRM-FW` | Possible firmware/software extension + HBOM/SBOM | PROPOSED (FNPRM) | TBD (user-set) | parts with firmware from Covered List producer | WATCH |
| `INJ-OPTICS-3_2T` | Reported curb on new Chinese transceiver models | REPORTED | ~2028–2029 | `optical_transceiver ∧ hq=CN ∧ speed≥3200`, exemption `us_bom_value_pct≥65` | NEW_AUTH_BLOCKED (capped amber) |
| `INJ-S5548` | Federal NSS procurement ban (InnoLight, Eoptolink + affiliates) | PROPOSED (bill) | enactment + 5 yrs | `optical_transceiver ∧ ultimate_parent ∈ {Zhongji InnoLight, Eoptolink}`, zone `federal_nss` | PROCUREMENT_BLOCKED |
| `INJ-1260H-INNOLIGHT` | InnoLight on DoD 1260H list | ENACTED | 2026-06-30 (DoD procurement) | `entity_lists ∋ DOD_1260H`, zone `federal_nss` | PROCUREMENT_BLOCKED |
| `INJ-INVERTERS` | Foreign-produced power inverters on Covered List | ENACTED | 2026-07-28 | `component_class ∈ {inverter, bess_pcs} ∧ production_country ≠ US` unless Conditional Approval | NEW_AUTH_BLOCKED |
| `INJ-ROUTERS` | Foreign-produced **consumer-grade** routers | ENACTED | 2026-03-23 | `component_class = oob_router ∧ consumer_grade = true ∧ production_country ≠ US` | NEW_AUTH_BLOCKED |
| `INJ-ROUTER-CA-EXPIRY` | Conditional Approvals expire | ENACTED | per part (2027-10-31 … 2028-03-06) | `conditional_approval_expiry exists` | APPROVAL_EXPIRES → WATCH after date |

Each canned inject embeds its source list (URLs in §10) and a one-line "what this does *not* cover" note.

---

## 5. Illustrative demo BOM and student import

### 5.1 Demo BOM (about 25 lines, labeled ILLUSTRATIVE)
It must include at least:
- 800G and 1.6T OSFP modules: InnoLight (CN, plus a TH-produced variant), Eoptolink (CN/TH), Coherent (US), Lumentum (US)
- **3.2T modules for Phase 2**: one InnoLight variant with a US DSP + US laser at `us_bom_value_pct = 68`, one at 40
- A white-label leaf switch whose **management SoC** sub-component comes from a Covered List producer, plus one without
- GPU server with BMC, NICs/DPUs
- OOB management router (one consumer-grade with a Conditional Approval expiring 2027-10-31, one enterprise)
- UPS, BESS PCS and inverters (foreign-produced with and without Conditional Approval)
- LPO and CPO alternatives (used by levers)

### 5.2 Import
- Accept **CSV or JSON** by drag-drop or file picker. Show a column-mapping dialog if headers differ.
- Validate: required fields `mpn, component_class, manufacturer, qty, zone_id, phase_id`. Show row-level errors and do not reject the whole file.
- Show the imported BOM side by side with the demo BOM ("Compare runs").

### 5.3 Sayari field mapping (student worksheet)
| Sim field | Where students get it in Sayari |
|---|---|
| `manufacturer`, `manufacturer_hq_country` | Entity profile |
| `ultimate_parent` | Ownership / UBO graph |
| `entity_lists` | Risk factors / sanctions & watchlist flags |
| `production_country` | Trade records (shipper origin, facility) |
| `sub_components` (DSP, laser, SoC vendors) | Trade/BOM research + supplier relationships |
| `sayari_entity_id` | Entity ID / URL |
Provide a downloadable `bom_template.csv` from the UI. A Sayari connector is available on this account for student research outside the artifact. The artifact itself only consumes the exported file.

---

## 6. Mitigation levers

| Lever | Effect on cloned parts | Cost / lead-time model (illustrative) |
|---|---|---|
| **Lock authorized SKUs** | Phase parts keep current `fcc_auth_date`, so they are not "new model" | +0% cost, risk of later obsolescence noted |
| **Last-time buy** | Moves selected phase order dates ahead of an inject effective date | +carrying cost % |
| **US DSP + US laser swap** | Sets `dsp_vendor/laser_vendor` to US and recomputes `us_bom_value_pct` (slider 0–100) | +X% unit cost |
| **Vendor switch (US optics)** | Replaces CN modules with Coherent/Lumentum equivalents | +cost, +lead time (capacity-constrained) |
| **LPO / CPO architecture** | Changes optics count or class for selected tiers | changes qty and unit cost |
| **Copper/AEC in-rack** | Removes short-reach optics | −optics qty |
| **Federal enclave segregation** | Moves federal workloads to a zone with a vetted BOM | duplicates part of the fabric |
| **Power chain swap** | Domestic or approved inverter/PCS | +cost |
| **HBOM/SBOM + firmware governance** | Sets an evidence flag that lowers FNPRM-related WATCH to "documented" | process cost |

Lever settings show as chips in the scenario header so that every result is reproducible.

---

## 7. UI specification

**Layout:** three columns on desktop, stacked on mobile.

1. **Left: Inject Library.** Cards with an evidence badge (color + label + icon; color is never the only signal), an effective date, and a toggle on/off. A "+ New inject" button opens the author form.
2. **Center: Simulation.**
   - **Timeline** (2026-01 → 2032-12): regulatory markers above, build phases below, a draggable `simDate` cursor, and Play (steps by quarter).
   - **Heatmap:** rows = component class, cols = zones × phases. Each cell shows the worst state and exposed spend. Click to filter the part table.
   - **Part table:** sortable. State pill, inject(s), runway days, exemption.
3. **Right: Trace & Considerations.**
   - **Trace panel:** for the selected part, which predicates matched or failed, the subcomponent responsible, the source links, and the evidence note.
   - **Design Considerations:** de-duplicated list of `DC-*` items triggered by active exposures. Each has a short rationale and the levers that address it.

**Inject author form:** fields mirror §2.1. The scope builder has dropdowns for field/op/value plus nested any/all. It shows a live "matches N parts" preview and a JSON tab for paste or edit. Saving requires an evidence level and at least one source.

**Exports:** "Decision memo" (printable HTML view, with print-to-PDF): scenario, active injects with evidence, exposure summary, chosen levers, remaining risk, and sources. "Export scenario JSON" (injects + levers + design) so runs can be shared or graded.

**Persistent banners:** "Educational simulation, not legal advice." When the demo BOM is loaded, also show "Illustrative BOM."

**Accessibility:** keyboard navigable, ARIA labels on pills and heatmap cells, contrast ≥ 4.5:1, works in light and dark mode.

---

## 8. Design Considerations catalog (`DC-*`)

| ID | Consideration | Triggered by |
|---|---|---|
| DC-OPTICS-GEN | Timing of 1.6T vs 3.2T standardization against phase-in dates | INJ-OPTICS-3_2T |
| DC-SKU-LOCK | Value of already-authorized models; freeze qualified SKUs | any `new_authorizations_only` |
| DC-BOM-CONTENT | DSP/laser sourcing to meet a US-content threshold | exemptions with `us_bom_value_pct` |
| DC-ARCH-ALT | LPO / CPO / copper to reduce exposed optics count | optics injects |
| DC-TENANT-ZONE | Segregated federal/NSS zones with a vetted supply chain | INJ-S5548, INJ-1260H |
| DC-INSIDE-BOX | Logic-bearing subcomponent audit of white-label gear | INJ-FCC-26-50 |
| DC-MGMT-PLANE | OOB routers, BMCs, console servers in scope review | INJ-ROUTERS, INJ-FCC-26-50 |
| DC-POWER-CHAIN | Inverter/PCS/UPS sourcing and approval expiries | INJ-INVERTERS, expiry |
| DC-APPROVAL-CALENDAR | Track Conditional Approval termination dates vs refresh plans | INJ-ROUTER-CA-EXPIRY |
| DC-EVIDENCE-PACK | HBOM/SBOM, FCC IDs, production-site records per part | FNPRM, any exemption |
| DC-FIRMWARE | Firmware provenance tracking ahead of possible extension | INJ-FCC-26-50-FNPRM-FW |
| DC-PRODUCTION-SITE | Third-country production (e.g. Thailand) and whether it changes anything; unresolved in reporting | optics injects + `production_country ≠ hq` |

---

## 9. Optional: intel-to-inject assist

If the Artifact page can use a Claude capability (confirm with `describe_capabilities` at build time), add a "Paste intel" box. The user pastes a news paragraph or rule excerpt, and Claude returns a **draft** inject JSON with an evidence level and a list of unknowns. The draft always opens in the author form for human review. It is never auto-activated. If the capability is not available, hide the box.

---

## 10. Source register (embed in canned injects)

**Primary**
- FCC 26-50 Third Report & Order + Third FNPRM (adopted July 22, 2026): https://docs.fcc.gov/public/attachments/FCC-26-50A1.pdf
- Federal Register final rule (effective Oct 13, 2026): https://public-inspection.federalregister.gov/2026-18535.pdf
- Federal Register FNPRM (firmware/software, HBOM/SBOM, Covered List bifurcation): https://public-inspection.federalregister.gov/2026-16197.pdf
- FCC news release on FCC 26-50: https://docs.fcc.gov/public/attachments/DOC-423291A1.pdf
- FCC Covered List (live): https://www.fcc.gov/supplychain/coveredlist
- Routers PN DA 26-278 (Mar 23, 2026): https://docs.fcc.gov/public/attachments/DA-26-278A1.pdf
- Router Conditional Approvals: DA 26-390 (eero, to Oct 31, 2027) https://docs.fcc.gov/public/attachments/DA-26-390A1.txt ; DA 26-548 (Sagemcom, to Dec 5, 2027) https://docs.fcc.gov/public/attachments/DA-26-548A1.pdf ; DA 26-768 (Humax/Askey, to Jan 17, 2028) https://docs.fcc.gov/public/attachments/DA-26-768A1.pdf ; DA 26-957 (ASUS, to Mar 6, 2028) https://docs.fcc.gov/public/attachments/DA-26-957A1.pdf
- Power inverters PN DA 26-786 (Jul 28, 2026): https://docs.fcc.gov/public/attachments/DA-26-786A1.pdf ; modification DA 26-870 (Aug 20, 2026): https://docs.fcc.gov/public/attachments/DA-26-870A1.pdf
- DoD 1260H list notice (FR, June 10, 2026): https://public-inspection.federalregister.gov/2026-11571.pdf
- S.5548 sponsor release (Sept 2026): https://www.mccormick.senate.gov/news/press-releases/senators-mccormick-gallego-cornyn-fetterman-introduce-bill-to-keep-chinese-transceivers-out-of-u-s-national-security-systems/ (bill text: congress.gov, S.5548, 119th Congress)

**Reported / secondary (use for REPORTED and ANALYST injects only)**
- Reuters exclusives on the transceiver curb (Aug 4, Nov 10, Nov 17, 2026), e.g. as relayed at https://www.trendforce.com/news/2026/11/11/news-microsoft-reportedly-urges-u-s-to-drop-proposed-chinese-optical-transceiver-ban-amid-ai-supply-concerns/
- 3.2T phase-in and 65% US-BOM figure (Morgan Stanley readout), summarized at https://roctelecom.com/insights/fcc-optical-module-rules-update/
- Law-firm analysis of FCC 26-50: Squire Patton Boggs https://www.squirepattonboggs.com/insights/publications/fcc-logic-bearing-hardware-component-rules-take-effect/ ; Cooley https://www.cooley.com/news/insight/2026/2026-07-29-fcc-expands-restrictions-on-covered-list-equipment-and-supply-chains

**Re-verify before every live demo:** the Covered List, the S.5548 status on congress.gov, and any new FCC transceiver item on the open-meeting agenda.

---

## 11. Build plan (Artifacts)

| Step | Deliverable | Done when |
|---|---|---|
| 1 | Confirm A1–A4 and get the current sim's state model | answers recorded |
| 2 | Data: `injects.json` (8 canned), `demo_bom.json` (~25 lines), `design.json`, `bom_template.csv` | validates against schema |
| 3 | Rule engine + `?test` harness | all tests pass |
| 4 | UI shell: library, timeline, heatmap, part table, trace | storyline steps 1–5 run |
| 5 | Levers + metrics + considerations panel | storyline step 6 runs |
| 6 | Import + column mapping + compare runs | a 500-line CSV scores in < 5 s |
| 7 | Inject author form + JSON tab (+ optional §9 assist) | new inject in < 5 min |
| 8 | Memo export, banners, accessibility pass | checklist passes |
| 9 | Publish as an artifact (private), run the dry-run demo, then share | presenter sign-off |
| 10 | Student hand-off pack: worksheet (§5.3), template CSV, grading rubric | delivered |

## 12. Acceptance criteria

- [ ] REPORTED/ANALYST injects never render a red (blocked) state.
- [ ] Every non-CLEAR state shows a trace with predicate results and at least one source.
- [ ] A component-rule match on a subcomponent is attributed to that subcomponent.
- [ ] Parts authorized before an effective date show "installed base safe" for new-authorization rules.
- [ ] The federal-zone-only injects do not affect commercial zones.
- [ ] Scenario JSON round-trips (export, then import, gives identical results).
- [ ] Works offline as one HTML file. No console errors. Storage calls are wrapped in try/catch.

---

## 13. Overview tab (for compliance officers, lawyers and demo audiences)

**Built:** `overview-tab.html` (self-contained, published privately as an artifact for review). Drop it into the demo as the first tab. Its scenario controls should read from and write to the shared `appState` so that the Simulation tab and the Overview agree.

### 13.1 Purpose
Before anyone touches the simulator, answer three questions: *What is law today versus only signaled? Which simulation inputs decide the outcome? Which decisions matter most, and how much?*

### 13.2 Sections (in order)
1. **Disclaimer + as-of date** (educational, not legal advice; check the live Covered List).
2. **Bottom line**: in force = the component rule plus the router and inverter categories. Transceiver curb = reported only. The highest weights fall on decisions that are hard to reverse.
3. **Decision-weight board (the picture)**: 11 decisions ranked 0–100 with tiers (Decide now ≥60 / Plan 40–59 / Monitor <40) and no-regret tags. Selecting a row shows its drivers, probabilities, impact, lock-in, owner, act-by date and mitigating moves.
4. **Scenario controls**: transceiver curb evidence (analyst / reported / draft / adopted / shelved), how a curb would be written (production location vs. named producer), S.5548 stage, FNPRM outcome, and a "serves federal/NSS tenants" toggle. Plus Reset.
5. **Instruments table**: 7 instruments, each with status mark, who is bound, trigger, *what it does not reach*, and key date.
6. **Timeline**: Feb 2025 (InP controls) through enactment + 5 years for S.5548, in hanging-date style. Estimated dates are greyed.
7. **Ten critical elements**: evidence level, legal hook, producer vs. location listing, who produced it, logic-bearing sub-components, authorization date and grandfathering, tenant type, US-content measurement, Conditional Approval terms, and runway. Each one maps to the sim field that carries it.
8. **Decision register table**: owner, weight, act-by, what would change it.
9. **Weighting method**: formula, evidence-to-probability defaults, and caveats.
10. **Questions for counsel**: 8 questions, each tagged with the decisions it moves. Questions 3, 4 and 6 are the Sayari student tasks.
11. **Glossary** (11 terms) and **Sources** (primary vs. reported).

### 13.3 Weighting model
```
P(decision)  = 1 − Π (1 − p_driver × relevance_driver)
Weight       = round(100 × P × (0.6 × Impact + 0.4 × LockIn) / 5)
Tier         = ≥60 Decide now | 40–59 Plan this quarter | <40 Monitor
No-regret    = LockIn ≤ 1 and P ≥ 0.8 (flagged regardless of tier)
```
Evidence defaults: Enacted 1.00 · Adopted 0.90 · Proposed agency 0.60 · Proposed bill 0.35 · Reported 0.40 · Analyst 0.20 · Shelved 0. If "serves federal tenants" is off, federal-only drivers are multiplied by 0.1.

| Decision | Owner | I | L | Drivers (p × r at today's evidence) | Weight today |
|---|---|---|---|---|---|
| Federal/NSS tenant segregation | Legal + Design | 4 | 5 | 805 indirect 1.0×0.8 · 5949 analog 1.0×0.5 · S.5548 0.35×1.0 | **82** |
| Optics generation timing | Design + Procurement | 5 | 5 | Curb 0.4×1.0 · InP 1.0×0.3 | 58 |
| Transceiver supplier mix | Procurement | 5 | 3 | Curb 0.4×1.0 · S.5548 0.35×0.6 · 1260H 1.0×0.3 | 56 |
| Power chain sourcing | Procurement + Eng | 3 | 4 | Inverter entry 1.0×0.8 | 54 |
| Architecture shift (LPO/CPO/copper) | Design | 4 | 5 | Curb 0.4×0.8 · InP 1.0×0.3 · producer-cascade 0.05×1.0 | 48 |
| Contract terms & flow-downs (no-regret) | Legal | 4 | 1 | Component rule 1.0×0.5 · Curb 0.4×0.7 · certifications 1.0×0.6 | 43 |
| Inside-the-box audit | Compliance | 3 | 2 | Component rule 1.0×0.7 · FNPRM 0.6×0.5 | 41 |
| HBOM/SBOM evidence program (no-regret) | Compliance | 3 | 1 | FNPRM 0.6×1.0 · 65% proof 0.4×0.8 · CA filings 1.0×0.5 | 38 |
| SKU lock / last-time buy | Procurement | 4 | 3 | Curb 0.4×1.0 · grandfather-withdrawal 1.0×0.2 | 37 |
| Management plane | Engineering | 2 | 1 | Routers 1.0×0.3 · CA expiry 1.0×0.4 · Component rule 1.0×0.3 | 23 |
| Regulatory engagement | Legal | 2 | 1 | FNPRM 0.6×1.0 | 19 |

**Sensitivities shown in the demo:**
- Curb → "Adopted": optics generation and supplier mix move into Decide now.
- Curb written as a named-producer listing: the architecture-shift weight rises, because co-packaged optics from a listed maker would make the host switch subject to the component rule.
- Federal toggle off: segregation drops to Monitor.

These are teaching defaults, and counsel should replace them with their own values and record the rationale.

### 13.4 Legal points the Overview must state (each verified against primary sources)
- FCC 26-50 applies to **new** authorizations. Pending applications are exempt unless amended to change a logic-bearing part. It applies **only to producer-based entries**, not to location-based categories (routers, inverters, drones).
- Named Covered List producers include their **subsidiaries and affiliates**. "Produced by" is judged on the **totality of the circumstances**.
- Grandfathering is not permanent. In DA 26-635 (June 26, 2026) the FCC barred continued import and marketing of previously authorized covered equipment that had been listed in 2024 or earlier.
- The router entry covers **consumer-grade** routers. Conditional Approvals last up to 18 months. Inverter approvals continue only while the onshoring plan is followed, and both end if the application contained misstatements.
- Section 805: direct DoD ban from June 30, 2026, indirect ban from June 30, 2027. Section 5949: effective Dec 23, 2027, with no rip-and-replace of equipment acquired earlier, and contractors may rely on supplier certifications.
- There is no FCC draft text, proceeding or threat determination on transceivers yet. The 3.2T and 65% details come from analyst readouts.

### 13.5 Acceptance
- [ ] All scenario controls re-rank the board and the register table immediately.
- [ ] Every instrument shows a "does not reach" entry.
- [ ] Status marks use shape as well as colour (solid / boxed / dashed / dotted).
- [ ] Works at 400px with no horizontal scroll. Tables collapse to labelled rows.
