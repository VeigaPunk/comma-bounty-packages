# Research Package — openpilot #30302: Ford Lightning Q4 Harness Faults When openpilot Not Running

**Issue:** https://github.com/commaai/openpilot/issues/30302 — "[$200 bounty] [Ford Q4] Ford Lightnings Harness Causes Errors When Openpilot Not Running"
**Status:** OPEN, `bug`/`bounty`/`car`/`ford` labels, **locked ("too heated")** since ~Jul 2025 after a bounty-credit dispute.
**Bounty ask (adeebshihadeh, 2024-01-02):** "$200 bounty for root causing and coming up with a fix we can ship."
**Hardware blocker:** needs an F-150 Lightning (2022+). This package is groundwork only — no bounty completion is claimed.

---

## 1. Symptom (confirmed by ≥5 owners)

On F-150 Lightning only (ICE F-150 and Mach-E on the same Q4 harness are unaffected, confirmed by alan-polk 2025-06-30):

1. Install Ford Q4 harness → no errors.
2. Add Comma Power → no errors.
3. **Plug the Comma OBD-C (USB-C) cable into the harness box — with NO Comma 3/3X attached — → dash faults: Traction Control, Park Assist, One Pedal Drive.**
4. Plugging in the 3X doesn't immediately help; **once openpilot fully boots and drives the harness relay open, faults clear.**

Reporter's test matrix (coffee-cake-isaac, 2024-01-31):

| Config | Faults? |
|---|---|
| Comma OBD-C cable, no Comma Power | No |
| Comma OBD-C cable + Comma Power | **Yes** |
| Long (~6 ft) USB-C cable | **Yes** |
| 1 m OBD-C cable, no Comma Power | No |
| Harness box only, no cable | No |

DallasSehlhorst (2024-03-19): a plain USB-C *extension* cable alone caused no faults; faults appeared only when the **Comma-supplied cable** was added — argues against pure length/EMI and toward the specific conductors the Comma cable connects (all four CAN pairs + SBU lines).

Not BlueCruise-dependent: reported on BlueCruise Lariat and non-BlueCruise XLT (Mjack007, 2024-04-13).

---

## 2. How the hardware actually works (from comma's own schematics + panda source)

### OBD-C cable pinout (commaai/neo `car_harness/OBD-C.sch.pdf`)
- **A2/A3 = CAN0_H/L**, **A10/A11 = CAN1_L/H**, **B2/B3 = CAN2_H/L**, **B10/B11 = CAN3_L/H** — i.e. the cable carries **four** CAN pairs, not USB data.
- **SBU1** drives the harness relay: "relay between CAN0 and CAN2… to disconnect these two buses, SBU1 is driven high to 5V."
- **SBU2** = ignition detect. Orientation detected by SBU-to-GND resistance (~100 Ω vs ~1 kΩ).

### Ford Q4 harness — **v1 schematic analyzed** (commaai/neo `car_harness/v1/Ford_Q4_Harness.sch.pdf`, dated 7/19/23)
The details below are read from the **v1 drawing** (image-only; no machine-readable netlist):
- Vehicle side ("TO CAR", 20-pin IPMA plug): **CAN0_H/L, CAN1_H/L**, PT1–PT4, IGN, GND.
- Module side ("TO MODULE", to the IPMA camera): **CAN2_H/L**, PT1–PT5, IGN, GND.
- The harness box's 26-pin Molex (501646-2600) routes to the camera/OBD-C port: IGN, **CAN1_H/L, CAN2_H/L**, PT1–PT4 loopbacks, GND, resistor loopback.
- Note printed on schematic: "PT5, PT6… PT12 will have to be looped between the two connectors as 26 pin Molex does not have enough passthrough lines."
- Termination jumpers (per jyoung8607, 2024-02-02, and open_pinout.sch.pdf): Q4 is configured **single-ended termination** — P23–P25 jumpered to supply 60 Ω toward the camera; P3–P5 open, assuming 60 Ω already present vehicle-side.

> **⚠ Revision ambiguity (open verification item):** neo master also ships a **v3 redesign** (`car_harness/v3/Ford_Q4_Harness.pdf`, dated 26 Jul 2024) with an **18-pin Molex 501646-1800** box connector (pins 501648-1000), TE 2288276-1 module connector, different wire list (UL1007 26AWG), and different connector pinouts. In the v3 drawing, **CAN1_H/CAN1_L + IGN + GND + a loopback still run into the box connector**, so a CAN1 path toward the camera/OBD-C appears to persist — but the CAN0/CAN2 intercept routing and overall stub geometry **must be cross-checked against v3 (and against which revision comma actually ships)** before finalizing any fix. Which harness revision the affected Lightning owners have is not recorded in the issue thread.

### Relay behavior (commaai/panda `board/drivers/harness.h`)
- `set_intercept_relay(intercept, ignition_relay)`: relay coil driven over the **SBU1/SBU2 GPIOs (open-drain)**; which SBU pin is relay vs ignition depends on cable orientation (NORMAL vs FLIPPED).
- `harness_init()`: "**keep buses connected by default**" — `set_intercept_relay(false, false)`. The relay is **fail-safe closed**; it is only driven open after panda boots into a non-silent safety mode (`board/main.c`: `set_intercept_relay(true, false)` only in the active-safety path; SILENT/NOOUTPUT leave it closed).
- The relay is **DPDT only** (photo confirmation via madsci1016, 2024-02-02): it switches just the CAN0↔CAN2 intercept pair. **CAN1 is hard-wired through the box down the OBD-C cable at all times**, relay open or closed.

### Consequence of the architecture
Whenever the OBD-C cable is plugged into the box and nothing is terminating the far end (3X unplugged, or panda still booting with relay closed):
- **CAN1** becomes a permanent unterminated stub hanging off the vehicle bus mid-run;
- **CAN0 and CAN2 are electrically bridged** (relay closed) and *both* run down the cable as unterminated stubs;
- CAN stub-length rules of thumb cited in-thread: max ~1.7 m at 500 kbit and ~3 m at 250 kbit (madsci1016, 2024-01-31/02-02). For CAN FD's 2 Mbit data phase the tolerance is correspondingly tighter (our interpolation — no in-thread figure); the cable + harness wiring is at or beyond the 500 kbit limit either way. jyoung8607 (2024-02-02) concurs: "the OBD-C cable does become part of the bus whether the relay is open or closed… TWO stubs".
- When openpilot finishes booting, the relay opens, CAN0↔CAN2 are separated and the panda presents proper termination → reflections drop → modules stop faulting. This exactly matches symptom step 8.

### Why Lightning only (hypothesis, see §3)
EV-specific modules/messages on that bus (One Pedal Drive, traction control on an EV powertrain) plus the Lightning's bus topology/length make it the least reflection-tolerant of the CAN-FD Fords. Mach-E/ICE F-150 share the harness but not the symptom (alan-polk, 2025-06-30: "confirmed" ICE unaffected).

---

## 3. Ranked root-cause hypotheses

1. **Unterminated CAN stub(s) down the OBD-C cable causing signal reflections on the vehicle CAN-FD bus.** When the 3X is absent/booting, CAN1 (always) and CAN0/CAN2 (relay closed) hang off the bus unterminated; reflections corrupt frames and Lightning modules (traction control / park assist / one-pedal) set DTCs. *Strongly supported*: every community fix that removes or terminates the stub works (§5), and the fault clears precisely when the relay opens.
2. **Termination-config mismatch on the Lightning.** Q4 assumes 60 Ω already on the vehicle side; if the Lightning's IPMA bus has different factory termination (e.g., camera supplies one end), the harness is mis-terminated even in normal operation. *Partially supported*: reporter measured CAN1 at a stable 60 Ω but "CAN2 constantly jumping… settles at 60 Ω… then jumping" (coffee-cake-isaac, 2024-01-16). jyoung8607 explicitly asked for someone to unplug the factory camera and measure — never answered in-thread.
3. **Susceptible-bus + marginal signal integrity rather than a hard fault**: the Lightning's longer/larger bus makes borderline stubs fatal where other Fords tolerate them. (Complement to H1, explains vehicle specificity — madsci1016's EV-topology argument.)
4. **EMI/cable-shielding induced noise.** Reporter's early favorite ("necessary shielding only on higher-end cables"). *Weakened* by DallasSehlhorst's extension-cable test and by resistor fixes working.
5. **Software/relay-timing bug (openpilot sets relay wrong in SILENT/NOOUTPUT).** Advanced by AI-generated PR #38061 — *reject*: faults occur with **no device plugged in at all**, so no panda/openpilot software change can be the root cause. Software can only mask (keep relay open whenever powered).

---

## 4. Diagnostic experiments for a Lightning owner (ordered, low→high effort)

1. **Reproduce the matrix** above on the target truck; log exactly which DTCs set (FORScan/OBD-II snapshot) to identify which modules and bus complain.
2. **Termination-plug A/B test (the decisive experiment, proposed by madsci1016 2024-01-31):** build a USB-C female coupler with 120 Ω across each CAN pair — A10–A11, A2–A3, B2–B3, B10–B11 — plug the OBD-C cable into it with no 3X. If faults disappear, H1 is proven. Caveat per Uncle-Tony (2025-07-16): full 120 Ω on every pair **over-terminates** and stresses transceivers; a single higher-value "tuning" resistor across the offending pair damps reflections without over-termination. (Our suggested starting sweep: ~120–500 Ω on the CAN0/CAN2 pair — this range is our proposal, not an in-thread value; Uncle-Tony did not publish a number.) Sweep values, log DTCs.
3. **Resistance topology survey (car off, battery settled):** unplug factory IPMA camera; measure H–L resistance on CAN0/CAN2 at the IPMA connectors (vehicle side and module side), and CAN1; repeat with harness installed, cable attached/detached, relay open/closed. Answers jyoung8607's open question (H2).
4. **candump comparison:** with a second panda or CANable on the OBD-II port / bus tap, `candump` the PT bus (bus 0 in openpilot Ford terms; CAN-FD 500k/2M) during (a) cable-unplugged clean state vs (b) cable-plugged fault state. Look for error frames / bus-off recovery and which IDs go missing (traction control & park assist frames). Note: gustavoclimaco's Cabana screenshot (2024-04-01) showed "CAN 2 = N/A", but that was on a **Ford Fusion 2017 with a Q3 harness, not a Lightning/Q4** — treat only as an analogy; worth reproducing on a Lightning whether bus 2 disappears during faults.
5. **Scope the bus (hiattwl1 offered, has oscilloscopes — never delivered):** eye diagram / ringing at the IPMA connector with cable stub attached vs detached. Direct reflection evidence.
6. **Relay-forced-open test:** flash panda with `set_intercept_relay(true, false)` forced from boot (or earlier in bootstub) → if faults *still* occur whenever the cable is attached, that confirms the **unswitched CAN1 stub** is sufficient to cause faults (predicted by the DPDT-only relay finding). If faults vanish, CAN0/CAN2 bridging is the dominant path. Either result scopes the required harness fix.
7. **SBU isolation desk test (garrettpall, 2024-01-16):** box+cable on the bench, verify SBU1/SBU2 at the cable end are open to all Molex pins; rules out relay-drive leakage pulling a CAN line.

## 5. Candidate fixes (ranked by shippability × evidence)

1. **Harness-box revision: switch all CAN pairs (add relays for CAN1/CAN3) or don't route pass-through buses down the OBD-C cable at all.** True root-cause fix; matches hiattwl1's "needs a second set of relays" (2024-01-31) and madsci1016's design critique. Requires comma hardware rev — exactly "a fix we can ship" if comma takes it on.
2. **Terminated/tuned USB-C coupler (community-validated by its authors; needs independent Lightning validation).** alan-polk (2025-05-12): "We have managed to prototype a custom USB-C Coupler with 120-ohm resistors integrated. This resolves the issues and only requires the USB-C coupler to be swapped" — reported working by the builders, but not yet independently reproduced on other Lightnings; Uncle-Tony's refinement uses a single tuning resistor to avoid over-termination. Cheapest shippable fix: comma already makes custom PCBs; alan-polk explicitly offered to hand it over. Open question: production-tolerance resistor value + validation it can't harm ICE/Mach-E buses.
3. **Long Q4 harness relocating the box to the mirror so the stock short OBD-C is used (community-PROVEN).** alan-polk's 9-ft rebuild (2024-03-19, looped PT lines at IPMA plugs, extended IGN/GND/CAN1/CAN2, 60 Ω camera-side) gave coffee-cake-isaac "0 issues and 0 faults" (2024-04-10); alexose published a visual guide (Google Doc, 2024-04-29); alan-polk then made a **no-cut extension harness** (parts: Molex 501646-2600 plugs ×2, 503091-2621 receptacle, 501647-1100 pins ×32, 3D-printed coupler Thingiverse 6600129; 2024-04-29). windydrew (2025-02-25) builds these for others, "0 faults to date." Works but is a harness variant, not a root fix; in the same 2024-04-11 comment alan-polk proposes comma offer the Q4 harness in two lengths ("8-Foot Q4 harness: 21+ F150, 21+ Lightning… 16-foot Q4 harness: 21+ MachE, 23+ Escape…").
4. **Panda firmware: drive the relay open whenever powered (incl. SILENT/NOOUTPUT and earlier in boot).** Only masks the fault while the device is attached and powered; **cannot fix the no-device case** (the actual repro). Useful as a mitigation, not a bounty answer. (Related prior art: PR #36420 touched relay/ELM327 offroad behavior.)
5. **Docs-side change:** `opendbc/car/ford/values.py` — `FordF150LightningPlatform.init()` currently sets `self.car_docs = []` with comment "Don't show in docs until this issue is resolved. See #30302". Companion PR to un-hide Lightning once a hardware fix ships. (PR #38061 attempted this but with a bogus root cause; closed unmerged. PR #37942 was AI-bot spam; closed unmerged.)

## 6. Claim strategy

- **The hard part (root cause + proven fix) is already community-solved in-thread**, but nobody has formally collected the bounty; the issue is open and locked after a heated credit dispute (madsci1016 vs alan-polk/Uncle-Tony, Jul 2025). Comma never publicly accepted a fix.
- Viable paths for a new claimant:
  1. **Get comma to bless a design first.** The ask is "a fix we can ship" — only comma can ship harness/coupler hardware. Contact adeeb/comma via Discord (#ford / bounty channel) with the tuned-coupler design + validation data before building; get written confirmation the design qualifies.
  2. **Recruit a Lightning owner for validation** — volunteers exist in-thread (windydrew, ebfio, LowCutKilt). Run §4 experiments 2–4 to produce clean before/after evidence (DTC logs, candump, resistance table, ideally scope shots).
  3. **Deliverable:** a short engineering write-up + schematic/BOM for the tuning-resistor coupler + owner-validated test data, posted to the issue (comma staff can unlock/lock-thread participation on request) or a referenced repo/gist.
  4. **Follow-up PR:** un-hide Lightning in opendbc docs (remove `car_docs = []`) once comma adopts the fix.
- **Attribution risk is real and documented.** madsci1016 (theory + termination-plug test + pinout), alan-polk (prototypes, extension harness), Uncle-Tony (tuning-resistor refinement) all have prior art in-thread. Any claim should explicitly credit them and, practically, propose shared credit/payment — the previous dispute is why the thread is locked. Do **not** repackage their fix as novel.
- Do not claim bounty completion from this package; it is read-only groundwork. No owner validation has been performed by us.

## 7. Sources

- Issue + all 66 comments: https://github.com/commaai/openpilot/issues/30302 (fetched via GitHub API 2025; key comments: adeebshihadeh 2023-10-22 & 2024-01-02, coffee-cake-isaac 2023-11-08 / 2024-01-16 / 2024-01-31 / 2024-04-10, madsci1016 2024-01-31 ×3 / 2024-02-02 / 2025-07-01, jyoung8607 2024-02-02, alan-polk 2024-01-31 / 2024-03-19 / 2024-04-11 / 2024-04-29 / 2025-05-12 / 2025-06-30, hiattwl1 2023-11-07 / 2024-01-31, DallasSehlhorst 2024-03-19, alexose 2024-04-29, windydrew 2025-02-25, Uncle-Tony 2025-07-16)
- OBD-C schematic (pinout, SBU relay/ignition functions): https://github.com/commaai/neo/blob/master/car_harness/OBD-C.sch.pdf
- Ford Q4 harness schematic (bus routing, PT loopbacks, 26-pin Molex): https://github.com/commaai/neo/blob/master/car_harness/v1/Ford_Q4_Harness.sch.pdf (v3 copy: car_harness/v3/Ford_Q4_Harness.pdf)
- Harness termination jumpers: https://github.com/commaai/neo/blob/master/car_harness/v1/open_pinout.sch.pdf
- panda harness relay driver: https://github.com/commaai/panda/blob/master/board/drivers/harness.h (`harness_init` "keep buses connected by default"; SBU open-drain relay drive)
- panda relay vs safety mode: https://github.com/commaai/panda/blob/master/board/main.c (intercept relay only driven in active safety mode)
- Lightning hidden from docs: https://github.com/commaai/opendbc/blob/master/opendbc/car/ford/values.py (`FordF150LightningPlatform`, `self.car_docs = []` citing #30302)
- Failed fix PRs: https://github.com/commaai/openpilot/pull/38061 (closed, wrong root cause), https://github.com/commaai/openpilot/pull/37942 (closed, bot spam); related PR #36420 (pandad ELM327/relay offroad behavior)
- alexose's visual harness-mod guide: https://docs.google.com/document/d/1keSikuriLE7qTHo2GoivIsX3x6P-m_xfNeW5oRVqzzY/edit
- Extension-harness parts/3D print: https://www.thingiverse.com/thing:6600129 (+ Mouser links in alan-polk 2024-03-20/2024-04-29 comments)
- Related forum thread (Mach-E install discussion referencing reflections): https://www.macheforum.com/site/threads/installing-comma-ai-openpilot-bluecruise-alternative.29591/page-9
