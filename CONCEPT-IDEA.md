# CONCEPT-IDEA: Proposed Regulation Simulation ("Reg-Intel Inject")

**Feature for:** Datacenter Supply-Chain Simulation Demo (built and hosted as a Claude Artifact)
**Delivery:** a new tab set in the existing demo (Overview · Simulation · Inject library · BOM import · Decision memo). The Overview tab is built: `overview-tab.html`
**Status:** Concept, ready for spec review
**Date:** 2026-11-23
**Demo scenario:** "Optics & Covered List 2026–2030" (FCC component rule + proposed Chinese optical transceiver restrictions)

---

## 1. The idea in one paragraph

Analysts often get early intelligence about a regulation that is not yet final: a draft rule, a bill that was just introduced, a Reuters exclusive, or a sell-side readout from Washington meetings. Today that intelligence ends up in a memo, and nobody tests it against the actual design. **Reg-Intel Inject** lets a user paste or select a piece of regulatory intelligence and turn it into a structured *inject*. The simulation then replays the datacenter design, its bill of materials (BOM), and its procurement timeline against that inject. The output shows which parts, racks, and build phases are exposed, when they become exposed, how confident we should be, and what design or sourcing moves reduce the exposure. Students can replace the demo's illustrative parts list with a real one they built in Sayari (actual part numbers, manufacturers, ownership chains, production locations), and the same rules re-score it.

## 2. Why this matters (the real concern behind the demo)

The demo scenario is real and current:

- **The FCC closed the "component part loophole."** FCC 26-50 (Third Report & Order, adopted July 22, 2026, effective **October 13, 2026**) lets the FCC refuse a *new* equipment authorization for any device that contains a logic-bearing hardware component produced by a Covered List entity. A Western brand on the box doesn't change that. The rule applies only to new authorizations. The accompanying Further Notice (FNPRM) asks whether to extend the ban to **software and firmware**, require **HBOM/SBOM** disclosure, and split the Covered List into producer-based and production-location-based categories.
- **Optical transceivers look like the next target.** Reuters reported on Aug 4, Nov 10 and Nov 17, 2026 that the administration is drafting, and as of Nov 17 is pressing ahead with, curbs on *new models* of Chinese optical transceivers, over objections from hyperscalers. A Morgan Stanley readout says the curbs would phase in at the **3.2T** generation, with a possible exemption for modules whose BOM value is **≥65% US-sourced**. No FCC document confirms this yet. InnoLight, Eoptolink, Dongshan Precision and TFC are **not** on the Covered List today.
- **Federal systems are the tightest scope.** **S.5548**, the Securing National Security Systems from Chinese Optical Transceivers Act of 2026 (introduced Sept 24, 2026), would extend the FY2023 NDAA semiconductor procurement ban to covered optical transceivers in national security systems. It names InnoLight and Eoptolink and gives a **five-year** transition. Zhongji InnoLight was added to DoD's **Section 1260H** Chinese Military Companies list on June 8, 2026, and DoD direct-procurement restrictions on 1260H entities started June 30, 2026.
- **Power and network gear are already covered by category.** Foreign-produced **power inverters** (July 28, 2026) and **consumer-grade routers** (March 23, 2026) are on the Covered List unless DoW or DHS grants a Conditional Approval. Many router approvals expire between **Oct 31, 2027 and Mar 6, 2028**.

An AI datacenter uses tens of thousands to millions of optical modules. Chinese suppliers hold roughly two-thirds of global transceiver supply and more than 60% of the 800G+ segment. Even a rule limited to *new models* changes the design: which optics generation to standardize on, when to lock SKUs, whether to keep federal tenants in segregated network zones, and how much copper or co-packaged optics to use.

## 3. Who uses it

| User | What they do with it |
|---|---|
| **Instructor / demo presenter** | Loads the canned "Optics 2026" inject, runs the timeline, shows exposure appear at Oct 2026, then 2028 (3.2T), then 2027–28 (router approvals expire), then enactment + 5 years (S.5548). |
| **Students (analysis track)** | Use Sayari to research real transceivers, switches, servers, inverters and their makers: ownership, 1260H/Covered List status, production sites (for example InnoLight and Eoptolink Thailand plants). Then import that parts file into the sim and defend a mitigation plan. |
| **Analyst / design reviewer** | Creates *new* injects from fresh intel (paste a Reuters paragraph or FCC notice), sets confidence, and compares design alternatives. |

## 4. Core concepts

1. **Inject:** a structured description of one regulatory signal. It records who is acting (FCC, Congress, DoW), the instrument (rule, bill, Covered List entry, conditional approval), the scope predicates (component class, entity list, production country, speed generation, logic-bearing flag), effective and phase-in dates, exemptions (for example ≥65% US BOM value), and the **evidence level** of the signal.
2. **Evidence ladder:** every inject carries one of five levels: `ENACTED` → `ADOPTED/PUBLISHED` → `PROPOSED (official text)` → `REPORTED (credible press)` → `ANALYST CHATTER`. The sim weights exposure by evidence level. The UI never shows a reported rule as if it were law.
3. **Exposure states per part:** `CLEAR`, `WATCH`, `NEW-AUTH BLOCKED` (the installed base is fine, but new models or refreshes are blocked), `FEDERAL-ZONE BLOCKED`, and `EXEMPT (via condition)`.
4. **Time:** a scrub-able timeline from 2026 to 2032 that overlays the design's build phases, such as Phase 1 at 800G/1.6T and Phase 2 at 3.2T, against regulatory dates.
5. **Mitigation levers:** actions the user can toggle to see the score change. Examples: lock authorized SKUs now, switch a module to a US DSP and US laser to cross 65%, change vendor, move to LPO/CPO, keep a federal enclave on a vetted supply chain, swap inverter or UPS vendor, or add an HBOM/firmware governance process.

## 5. Demo storyline (8–10 minutes)

1. **Baseline:** a 20 MW AI hall. GPU racks, a leaf/spine fabric with 800G and 1.6T optics (a mix of InnoLight, Eoptolink, Coherent and Lumentum modules), OOB management routers, UPS, BESS and inverters. Everything shows CLEAR.
2. **Inject 1, the component rule (ENACTED, Oct 13, 2026):** nothing in the illustrative parts list turns red, because no transceiver maker is on the Covered List. One white-label switch whose management SoC comes from a Covered List entity becomes *NEW-AUTH BLOCKED* for its next hardware revision. Lesson: the rule looks *inside* the box.
3. **Inject 2, transceiver curb (REPORTED, phase-in ~2028):** move the slider to 2028. Every Phase 2 3.2T order from InnoLight or Eoptolink turns *WATCH* (amber, because the evidence is only reported). Turn on the ">=65% US BOM" lever: modules with a US DSP and US lasers flip to *EXEMPT*.
4. **Inject 3, S.5548 (PROPOSED, enactment + 5 years):** mark one hall as a federal or NSS tenant zone. Its optics turn *FEDERAL-ZONE BLOCKED* at the transition date. Commercial zones are unaffected.
5. **Inject 4, power and routers (ENACTED, category rules):** inverter and OOB router replacement plans that depend on a Conditional Approval turn *WATCH* as the approval expiry dates (2027–28) pass.
6. **Mitigation run:** the user picks levers. The sim shows the cost delta, lead-time delta, and remaining exposure, plus a one-page "design decision memo" export.
7. **Student hand-off:** "Replace this illustrative BOM with your Sayari-researched one, then re-run."

## 6. How this feeds datacenter deployment and design considerations

The inject engine produces a list of **design considerations**. These carry over to any future regulation scenario:

- **Optics generation timing:** standardizing on 1.6T for longer, or buying 3.2T early, shifts exposure. The reported curbs target *new models*, so a model that already holds an authorization is worth more.
- **SKU lock and buy-forward:** the component rule and the reported transceiver curb both apply to *new* authorizations. Keeping qualified SKUs stable, and treating last-time buys as a strategy, both reduce risk.
- **BOM content engineering:** the 65% test (it mirrors Buy American, 48 CFR 25.101(a), which the FCC already uses for inverters and robots) makes DSP and laser sourcing a compliance variable as well as a cost variable.
- **Architecture alternatives:** copper and active electrical cables in-rack, LPO (no DSP), CPO, and optics-light topologies all lower the transceiver count exposed.
- **Tenant zoning:** separate federal or NSS halls on a vetted supply chain contain S.5548-type scope without re-sourcing the whole site.
- **Power chain:** inverter, BESS and UPS sourcing must account for Covered List category rules and the expiry dates of Conditional Approvals.
- **Management plane:** OOB routers, BMCs and console servers are small and inexpensive but carry logic, so they are easy to miss. Check whether consumer-grade category rules or component rules reach them.
- **Evidence records:** HBOM/SBOM, production-location records and FCC IDs per part. The FNPRM signals these may become mandatory, and they are what make an exemption claim defensible.
- **Firmware governance:** firmware is not covered today. The FNPRM asks about extending coverage, so track firmware provenance now.

## 7. What's in and out (v1)

**In:** inject authoring (form + JSON), four canned demo injects with citations, a rule engine, timeline, exposure heatmap, mitigation levers, student BOM import (CSV/JSON, Sayari-ready fields), and a decision-memo export.
**Out (v1):** live calls to Sayari or the FCC from inside the artifact (the artifact is offline and self-contained, and students import exported files), legal determinations, and pricing feeds.

## 8. Guardrails

- Every inject shows its evidence level and source links. Reported or analyst-only signals render amber, never red.
- Banner: "Simulation for education and planning. Not legal advice. Verify against the current FCC Covered List and official texts."
- The demo BOM is labeled **illustrative**. Vendor names are real because the scenario is about real named entities, but the quantities, prices and part numbers are made up until students supply real ones.

## 9. Success criteria

- A presenter can run the whole storyline in under 10 minutes with no setup.
- A student can import a 50–500 line BOM and get a scored result in under 5 seconds.
- Every red or amber state can be traced in one click to the rule predicate and source that caused it.
- A new inject from fresh intel can be authored in under 5 minutes.

## 10. Primary sources found (and what is only reported)

| Scenario claim | Best source | Status |
|---|---|---|
| Component rule, effective Oct 13, 2026, new authorizations only | FCC 26-50 Third R&O: https://docs.fcc.gov/public/attachments/FCC-26-50A1.pdf ; Federal Register final rule (pub. Sept 11, 2026): https://public-inspection.federalregister.gov/2026-18535.pdf ; FCC news release: https://docs.fcc.gov/public/attachments/DOC-423291A1.pdf | **Primary, confirmed** |
| "Does not cover firmware" | FCC 26-50 Third FNPRM: https://public-inspection.federalregister.gov/2026-16197.pdf | **Primary.** True today, but the FNPRM proposes extending to software/firmware and HBOM/SBOM disclosure. The scenario should say "not *yet*." |
| Transceiver makers not on Covered List | FCC Covered List: https://www.fcc.gov/supplychain/coveredlist | **Primary (check live list before each demo)** |
| Transceiver ban drafting, phased from 3.2T, ≥65% US BOM exemption | Reuters exclusives (Aug 4, Nov 10, Nov 17, 2026), relayed via secondary coverage; Morgan Stanley readout for the 3.2T and 65% details | **Reported only, no FCC text.** Tag as REPORTED / ANALYST. |
| S.5548, five-year transition, names InnoLight and Eoptolink | Sponsor release: https://www.mccormick.senate.gov/news/press-releases/senators-mccormick-gallego-cornyn-fetterman-introduce-bill-to-keep-chinese-transceivers-out-of-u-s-national-security-systems/ ; bill page on congress.gov (S.5548, 119th Congress) | **Primary.** Introduced, not passed. Scope is *federal procurement* for NSS. |
| InnoLight on 1260H list (June 2026) | DoD Federal Register notice (June 10, 2026): https://public-inspection.federalregister.gov/2026-11571.pdf | **Primary, confirmed** |
| Foreign-produced power inverters (July 2026) | FCC PN DA 26-786: https://docs.fcc.gov/public/attachments/DA-26-786A1.pdf ; modified by DA 26-870 (Aug 20, 2026): https://docs.fcc.gov/public/attachments/DA-26-870A1.pdf | **Primary, confirmed** |
| Foreign-produced routers (March 2026) | FCC PN DA 26-278: https://docs.fcc.gov/public/attachments/DA-26-278A1.pdf | **Primary.** Applies to **consumer-grade** routers. Enterprise and datacenter routers aren't named, so the demo should not imply all routers are covered. |
| Router approvals expire 2027–2028 | e.g. DA 26-390 (eero, Oct 31, 2027), DA 26-548 (Sagemcom, Dec 5, 2027), DA 26-768 (Humax/Askey, Jan 17, 2028), DA 26-957 (ASUS, Mar 6, 2028) at docs.fcc.gov | **Primary, confirmed** |

**Wording fixes to make in the scenario text:**
1. "Does not cover firmware" → "does not *yet* cover firmware. The FCC's FNPRM seeks comment on extending it."
2. "Routers (March 2026)" → "consumer-grade routers (March 2026)."
3. Transceiver bullet: add "per Reuters and analyst reporting; no FCC text yet."
4. S.5548: add "introduced Sept 24, 2026; not enacted. Bars federal *procurement* for national security systems."
