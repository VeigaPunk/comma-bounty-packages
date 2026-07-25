# Team F1 Research Package — opendbc #3426: "[$10k bounty] Ford F-150 2026 (TRON) support"

**Issue:** https://github.com/commaai/opendbc/issues/3426 (opened by @adeebshihadeh, labels: `bounty`, `car port`, `ford`)

**Bounty terms (verbatim):**
- "Let's get the new F-150s supported! Any TRON platform car model is also eligible for this bounty."
- Requires: (1) a **harness design (schematic)**, (2) a **high quality lateral port**, (3) at least a **proof-of-concept for longitudinal**.
- **$10k** = full openpilot experience (user installs hardware + openpilot and is ready to go).
- **$2k** = "2021 Toyota Rav4 Prime"-style support — works only if the user manually configures extra parameters (i.e., per-vehicle key extraction / manual SecOC configuration, as done for Toyota TSK/SecOC cars).

**Status of this package:** groundwork research only. This bounty is hardware-blocked — it cannot be completed without physical access to a TRON-platform vehicle (2024+ F-150 or other TRON model) plus harness prototyping. Nothing here claims bounty completion.

---

## 1. What the TRON platform is

**TRON = "Trusted Realtime Operating Network"** — Ford's encrypted/authenticated CAN-FD architecture (SecOC-style message authentication appended to critical ADAS messages). Ford began rolling it out with the **2023 Super Duty**, and has been extending it to refreshed models. On TRON vehicles the lateral-control messages that openpilot must man-in-the-middle (IPMA → PSCM) carry a cryptographic MAC, so a third party cannot forge steering/ACC commands without a per-vehicle key. This is directly analogous to Toyota's TSK/SecOC situation (see the optskug/docs community documentation, which explicitly cross-references Ford TRON: "Ford has been rolling out their Trusted Realtime Operating Network (TRON) on CANFD vehicles starting with the 2023 Superduty ... likewise currently incompatible with openpilot" — https://github.com/optskug/docs).

**How TRON is detected in opendbc today (important, concrete):** upstream `opendbc/car/ford/interface.py` already detects and locks out TRON cars:
```python
# TRON (SecOC) platforms are not supported
# LateralMotionControl2, ACCDATA are 16 bytes on these platforms
if len(fingerprint[CAN.camera]):
  if fingerprint[CAN.camera].get(0x3d6) != 8 or fingerprint[CAN.camera].get(0x186) != 8:
    carlog.error('dashcamOnly: SecOC is unsupported')
    ret.dashcamOnly = True
```
i.e. on the camera bus, `LateralMotionControl2` (0x3d6) and `ACCDATA` (0x186) grow from 8 to 16 bytes because a SecOC authenticator is appended. Source: https://github.com/commaai/opendbc/blob/master/opendbc/car/ford/interface.py. The identical TRON/SecOC lockout block is also carried in BluePilot's fork: https://github.com/BluePilotDev/bluepilot/blob/master/opendbc_repo/opendbc/car/ford/interface.py (verified verbatim).

**Confirmed/suspected TRON models** (per the Blue Pilot "Confirmed TRON Status List" published 2025-08-13 — note: the link in the issue, https://bluepilot.dev/2025/08/13/confirmed-tron-status-list/, currently HTTP-301s to a Rickroll; contents corroborated via multiple community threads quoting it):
- **2023+ Super Duty (F-250/350)** — first TRON vehicle.
- **2024+ ICE F-150** — TRON (confirmed on f150gen14: "2023 was superduty, 2024 was ICE F150 and ICE mustang" — https://www.f150gen14.com/forum/threads/introducing-bluepilot-a-ford-specific-fork-for-comma3x-openpilot.24241/page-20). Also corroborated: "OpenPilot will not work, at least for now, on a 2024 F150 due to security enhancements on the modules" (https://www.f150gen14.com/forum/threads/will-comma3x-openpilot-work-on-a-2024-f150-powerboost.29203/). Some early-build 2024s may predate encryption — build-date matters; verify per vehicle via the 8-vs-16-byte fingerprint check above.
- **2024+ ICE Mustang (S650)** — TRON.
- **2026 Transit** — confirmed TRON, "remains incompatible" (sunnypilot community weekly journal: https://community.sunnypilot.ai/t/weekly-community-journal/653?page=2).
- **Mach-E** — rollout timing unknown as of the bluepilot posts; 2021-24 Mach-E is currently supported upstream, so TRON has not hit existing model years.
- 2026 F-150 (bounty title vehicle) is squarely TRON.

**Bounty eligibility:** *any* TRON platform car qualifies — a 2024/2025 F-150, Super Duty, ICE Mustang, or 2026 Transit would all satisfy it. The F-150 is simply the highest-value target.

**How to check a specific vehicle:** sunnypilot community thread "How can I tell if my Ford has encrypted canbus (TRON)?" — https://community.sunnypilot.ai/t/how-can-i-tell-if-my-ford-has-encrypted-canbus-tron/808. Practically: fingerprint the camera bus and check the DLC (data length code) of 0x3d6/0x186 — 16 bytes = TRON.

---

## 2. Current Ford support in opendbc/openpilot and what carries over

Supported Ford platforms today (`opendbc/car/ford/values.py`, https://github.com/commaai/opendbc/blob/master/opendbc/car/ford/values.py):

| Config | Bus | Models |
|---|---|---|
| `FordPlatformConfig` (CAN, Q3 harness, 12-pin) | CAN | Bronco Sport 2021-24, Escape/Kuga 2020-22, Explorer/Aviator 2020-24, Focus Mk4, Maverick 2022-24 |
| `FordCANFDPlatformConfig` (CAN-FD, Q4 harness, 20-pin) | CAN-FD | Escape/Kuga 2023-24, Expedition 2022-24, **F-150 2021-23** (`FORD_F_150_MK14`), F-150 Lightning 2022-23 (docs-hidden), Mustang Mach-E 2021-24, **Ranger 2024** |

Key architecture facts (comma Ford wiki: https://github.com/commaai/openpilot/wiki/Ford):
- **Lateral:** Ford's lateral planner lives **inside the PSCM (steering rack)**, not the camera. The IPMA sends a path polynomial `y(x) = c0 + c1·x + c2·x²/2 + c3·x³/6` (path offset, path angle, curvature, curvature rate) via `LateralMotionControl`/`LateralMotionControl2` (CAN-FD) at 20 Hz, plus `Lane_Assist_Data1` at 33 Hz. openpilot imitates the IPMA's message set (7+ signals). Angle control, `steerControlType = angle`, curvature limits ~0.02 m⁻¹, ~2.0-3.6 m/s² lateral accel cap depending on branch.
- **Longitudinal:** stock ACC via radar (Delphi MRR, `FORD_CADS` DBC) with alpha/experimental openpilot longitudinal available when radar is absent; `ACCDATA` at 50 Hz, `FordSafetyFlags.LONG_CONTROL`.
- **Safety:** `SafetyModel.ford` with `FordSafetyFlags.CANFD` for Q4 cars; panda enforces TX limits and the harness relay opens the camera bus for MITM.
- **Fingerprinting:** FW-version fuzzy matching with Ford part-number platform/model-year hints + as-built data blocks (0xDE00) — this machinery will work unchanged on TRON cars (UDS queries are diagnostic, not SecOC-blocked).

**What likely carries over to a 2026/TRON F-150 port:**
- The entire Ford CAN-FD platform scaffold: DBCs (`ford_lincoln_base_pt`, `FORD_CADS`), `fordcan.py` message builders, carstate parsing, fuzzy fingerprinting, safety model skeleton.
- PSCM path-polynomial lateral semantics — TRON encrypts the messages, it (almost certainly) doesn't change the path-following API inside the PSCM.
- Tuning knowledge from bluepilot (curvature blend ratios, path-angle amplitude, in-curve reduction) and upstream PRs (below).

**What changes / is new work:**
1. **SecOC on the camera bus**: `LateralMotionControl2` and `ACCDATA` (and likely other IPMA-sourced frames) carry an authenticator. openpilot must either (a) obtain/inject valid MACs (Toyota-Rav4-Prime-style per-vehicle key extraction → the $2k tier), or (b) find a bypass (IPMA firmware patch, gateway manipulation, or a different unauthenticated lateral API on the PSCM — e.g., whether TJA/LKA or APA paths remain unauthenticated). No public bypass is known.
2. **Harness**: 2024+ F-150 IPMA connector/pinout must be re-verified (see §4); TRON vehicles also re-architect the gateway (GWM), and forum reports indicate TRON modules go through an encrypted provisioning handshake ("TRON for GSM replacement" appears in Ford TSB 25-2276 service procedures — i.e., TRON provisioning is a dealer-tool operation).
3. **FW database**: new platform codes for 2024-2026 ECUs must be collected from real cars.

---

## 3. Existing community work

- **BluePilot** (https://github.com/BluePilotDev/bluepilot, installer `installer.comma.ai/BluePilotDev/bp-x.y`, docs https://bluepilot.dev) — the dominant Ford-specific fork, with extensive F-150 lateral tuning. Their opendbc ford interface carries the same TRON/SecOC lockout as upstream. Their TRON status list (Aug 2025) is the reference the bounty issue itself links. Community: f150gen14 thread https://www.f150gen14.com/threads/introducing-bluepilot-a-ford-specific-fork-for-comma3x-openpilot.24241/ and sunnypilot community Ford section.
- **Upstream Ford lateral PRs** (active, relevant to a "high quality lateral port"):
  - commaai/opendbc#3315 — "Ford: Use Path Angle and Offset for CAN FD Lateral Control" (@hiimisaac, open). Documents the PSCM path polynomial `y(x)=c0+c1x+c2x²/2+c3x³/6` with ranges (c0 ±5.12 m, c1 ±0.5 rad, c2 ±0.02 m⁻¹), master sends c2 only; validation routes on F-150. **This is the current best lateral direction to build a TRON port on.**
  - commaai/opendbc#3405 — "Ford native curvature" (@elkoled, **merged 2026-06-04**) — generic curvature limits in lat-accel/lat-jerk space (MAX_LATERAL_ACCEL 3.6 m/s², MAX_LATERAL_JERK 3.6 m/s³), now in master: a TRON port inherits the current curvature-limit framework rather than the old hand-picked breakpoints.
- **optskug/docs** (https://github.com/optskug/docs) — the Toyota SecOC/TSK playbook, directly relevant as the $2k-tier template: per-vehicle key extraction (https://icanhack.nl/blog/secoc-key-extraction/), "works if user manually configures extra parameters" — exactly what comma means by the Rav4 Prime analogy. Its timeline also notes "SecOC longitudinal control support merged into upstream comma openpilot/opendbc" (Toyota, by chrispypatt, Sept 2025) — upstream already has SecOC infrastructure precedent.
- **opendbc issue #2887** ("Ford F-250 port", closed) — F-250 2023 (Super Duty = first TRON vehicle) port inquiry; shows community interest but no public SecOC break. https://github.com/commaai/opendbc/issues/2887
- **Issue #3426 comment** — user `0xjc65eth` has publicly volunteered for software-validation-side work (emulated validation harness, CAN replay, safety-test coverage) and is explicitly soliciting SecOC captures/harness pinout notes from anyone with a TRON vehicle. Coordinate rather than duplicate.
- **Historical F-150 work**: tinkerborg's hard-coded F-150 branches and phoenixpilot (old, pre-CAN-FD) — superseded.
- **No public TRON port attempt or SecOC key extraction for Ford exists on GitHub as of this writing** (searched code/issues for TRON/SecOC/F-150 2024+). This is the true blocker and where novel work would go.

---

## 4. Harness: existing Q4 design and what a TRON harness needs

**How the current Ford Q4 harness works** (comma wiki: https://github.com/commaai/openpilot/wiki/Ford#ford-q4--bluecruise--can-fd-20-pin):
- Tap point is the **IPMA (Image Processing Module A, Mobileye EyeQ4, BlueCruise camera) behind the rearview mirror**. The harness intercepts the IPMA's **HS CAN-FD bus** so openpilot can MITM the lateral (`LateralMotionControl2`) and ACC (`ACCDATA`) messages toward the rest of the vehicle, while a second bus connection goes via OBD-C to the powertrain/OBD side through the GWM. A relay in the harness box opens/closes the intercepted bus; comma power v3 provides constant power.
- Shop BOM for Q4 cars (F-150): Ford Q4 connector, harness box, comma four, comma power v3, OBD-C 2 ft + **long OBD-C 9.5 ft** (cable runs from driver's footwell to headliner), mount.
- Documented IPMA connectors (2021-23 F-150): **C4242A** (black, 24-pin: 12 V pin 1 WH-OG, GND pin 13, parking-aid sensors), **C4242B** (grey, 20-pin: HS CAN-FD LOW pin 7 BU-OG / HIGH pin 8 YE-OG; rear-radar CAN pins 12/13/15/16; forward-radar CAN pins 19/20; radar power/grounds; LDW heater), **C4242C** (black, 20-pin: front corner-radar CAN + power). Candidate connector part: TE Connectivity 2288276-1 (Generation Y, 20-pos) for C4242B/C; C4242A source unresolved — "3D printed parts required" if DIY. Full pin tables: wiki link above.
- Known Q4 harness caveat: F-150 Lightning faults (traction/park-assist/one-pedal errors when device absent) — openpilot issue #30302; Lightning hidden from docs until resolved. A TRON harness design must re-validate this class of behavior.

**What a TRON (2024+ F-150) harness design needs:**
1. **Physical re-verification**: confirm whether 2024+ IPMA keeps C4242A/B/C or moved to new connectors (Ford refreshed the IPMA on TRON models; connector part numbers must come from a real vehicle teardown or Ford workshop wiring diagrams (WSM) — budget for WSM/FDRS access).
2. **Bus topology mapping on a TRON car**: which bus carries PSCM-bound lateral messages post-TRON (still the IPMA HS-CAN-FD?), whether the GWM now terminates/authenticates additional segments, and where an MITM tap remains physically possible. The SecOC check shows the authenticated frames are visible on the camera bus, so an IPMA tap still sees them.
3. **Deliverable schematic expectations** (matching how comma evaluates harness designs): connector part numbers + sources (TE/Aptiv), pinout table (vehicle side ↔ harness box), CAN-FD pair routing with the relay MITM on the camera bus, 12 V/GND sourcing, radar-bus passthrough (unchanged — radars stay on the vehicle side), termination notes, and a photo-verified install location. Precedent deliverable format: the wiki pinout tables + comma shop harness BOM.
4. Practical path: start from the existing Q4 harness on a TRON vehicle — mechanically it may already fit; the schematic work is then verification + delta documentation rather than a clean-sheet design.

---

## 5. Realistic execution plan & claim strategy

**Phase 0 — vehicle & data acquisition (the critical path; everything else waits on it):**
- Secure a TRON vehicle: 2024+ F-150 (bounty target), or any TRON model (Super Duty 2023+, ICE Mustang 2024+, Transit 2026). Cheapest test mule may be a rental/owner partnership via f150gen14 / comma Discord `#ford`.
- With a comma four + Ford Q4 harness (or developer harness): record routes with LKAS/ACC active; confirm TRON via the 8-vs-16-byte fingerprint check (0x3d6/0x186 on camera bus); dump full CAN-FD captures; collect FW versions + as-built blocks for the fingerprint DB.
- Coordinate with `0xjc65eth` (issue comment) who offered emulated validation; share captures.

**Phase 1 — harness schematic (bounty deliverable 1):**
- Verify connector/pinout on the TRON IPMA (teardown + Ford WSM pin tables); document deltas vs C4242A/B/C; produce the schematic as above. This is achievable with vehicle access alone and is the most bankable deliverable.

**Phase 2 — SecOC strategy decision (determines $2k vs $10k):**
- **$2k tier (Rav4-Prime-style)**: attempt Ford SecOC key extraction per vehicle, adapting the Toyota TSK methodology (optskug/docs; icanhack.nl SecOC key-extraction writeup). openpilot already carries Toyota SecOC longitudinal support upstream, so infra precedent exists; lateral SecOC signing for Ford messages would be new panda/opendbc work (safety-code implications — note comma's fork safety policy).
- **$10k tier**: requires a *general* solution — e.g., an unauthenticated control path (alternate PSCM API), an IPMA firmware patch that disables SecOC, or a key-derivation break. Treat as research-track; no public evidence any of these exist yet.
- Also examine TRON provisioning behavior (Ford TSB 25-2276 "PMI and TRON for GSM replacement") for how keys are provisioned — possible attack surface.

**Phase 3 — lateral port (deliverable 2):**
- Build on opendbc#3315 (path angle + offset CAN-FD lateral) and current master curvature-limits; validate with lateral maneuver reports on the TRON vehicle once messages can be emitted. "High quality" per comma = no ping-pong, no windup, route evidence across highway/curves, and passing the Ford safety tests (100% line coverage required).

**Phase 4 — longitudinal PoC (deliverable 3):**
- Stock-ACC passthrough first (easiest: openpilot lateral + stock ACC = still a shippable config), then alpha-long PoC via ACCDATA if radar-absent or SecOC-signed ACCDATA is solvable with the Phase-2 key.

**Claim strategy / review expectations:**
- Coordinate in **comma Discord `#dev-opendbc-cars` / Ford channel**; bounty admin is comma staff (@adeebshihadeh opened the issue). Standard bounty flow: comment on the issue to signal work, open PRs against `commaai/opendbc` (+ `panda` for safety), provide **test routes (dongle ID + route names, public)** — comma's review requires real-route evidence; emulated validation is supporting material only (see #3315's route format).
- Follow comma contribution norms (docs.comma.ai/CONTRIBUTING): small single-goal PRs, no 500+ line mega-PRs; safety code has strict CI (MISRA, 100% line-coverage unit tests).
- Bounty reference points: standard port bounties ($2k brand / $250 model / $300 actuation message) at https://github.com/commaai/opendbc#bounties and comma.ai/bounties; this $10k is a special bounty, judged by comma on the three deliverables.
- Honest risk statement: without a SecOC solution, the deliverables cap out at harness schematic + dashcam-mode port scaffolding; the realistic near-term win is the **$2k manual-config tier** keyed to per-vehicle key extraction, contingent on Ford SecOC key extraction proving feasible.

---

## Key sources

- Bounty issue: https://github.com/commaai/opendbc/issues/3426
- TRON status list (linked in issue; currently redirects): https://bluepilot.dev/2025/08/13/confirmed-tron-status-list/ — corroboration: https://www.f150gen14.com/forum/threads/introducing-bluepilot-a-ford-specific-fork-for-comma3x-openpilot.24241/page-20 , https://community.sunnypilot.ai/t/how-can-i-tell-if-my-ford-has-encrypted-canbus-tron/808 , https://community.sunnypilot.ai/t/f-150-compatibility/1139 , https://github.com/optskug/docs
- TRON/SecOC detection in upstream opendbc: https://github.com/commaai/opendbc/blob/master/opendbc/car/ford/interface.py
- Ford platforms: https://github.com/commaai/opendbc/blob/master/opendbc/car/ford/values.py
- Ford wiki (harness pinouts, lateral architecture): https://github.com/commaai/openpilot/wiki/Ford
- Lateral PRs: https://github.com/commaai/opendbc/pull/3315 , https://github.com/commaai/opendbc/pull/3405
- F-150 Lightning Q4 harness issue: https://github.com/commaai/openpilot/issues/30302
- Toyota SecOC analogue ($2k tier template): https://github.com/optskug/docs , https://icanhack.nl/blog/secoc-key-extraction/
- BluePilot fork: https://github.com/BluePilotDev/bluepilot
- Bounty/CONTRIBUTING process: https://github.com/commaai/opendbc#bounties , https://docs.comma.ai/CONTRIBUTING/
- 2024 F-150 incompatibility report: https://www.f150gen14.com/forum/threads/will-comma3x-openpilot-work-on-a-2024-f150-powerboost.29203/
