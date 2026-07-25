# Status Research Package — openpilot #32386: "Ship Ubuntu 24.04 + mainline kernel to master"

**Issue:** https://github.com/commaai/openpilot/issues/32386 (open, labels: `bounty`, `comma three`; milestone: "project vamOS" → https://github.com/commaai/vamOS)
**Prepared by:** Team F4 — informational status package only. No competing work proposed.

---

## ⚠️ IMPORTANT CORRECTION on the lock status

The issue body (authored by @adeebshihadeh, 2024-05-09) says: *"Locked to @andiradulescu, who did the initial 24.04 work."*

**However, the lock was explicitly lifted by @adeebshihadeh on 2025-07-16.** In the issue comments, @workinright asked "Is the kernel mainlining part of the bounty also still locked, or can I start working on this?" and @adeebshihadeh replied: **"Unlocked -- good luck!"** (https://github.com/commaai/openpilot/issues/32386#issuecomment-3081024945; comment id 3081024945, dated 2025-07-16T21:32:04Z).

So per the most recent maintainer statement, the remaining mainline-kernel part is **not locked**. That said:
- @andiradulescu's tracking PR (agnos-builder #233) remains the canonical work-in-progress.
- comma's own @greatgitsby has open mainline bring-up PRs (see below), and the issue sits under comma's internal "project vamOS" milestone — meaning comma staff are actively working the problem in-house. A third-party claimant should confirm payout/claim terms with comma before investing effort (as commenter @Gooseboy2234 asked on 2026-06-07, unanswered as of this writing).

## Bounty split (from issue body)

- [x] **Part 1 — Get 24.04 shipped to master ($1k): DONE.** Checkbox checked; @adeebshihadeh (2024-08-14): "I consider 24.04 complete since it's merged into agnos-builder master."
- [ ] **Part 2 — Get mainline kernel shipped to master ($2k): OPEN.**

## What's shipped (Part 1 evidence)

- agnos-builder **PR #262 "Ubuntu 24.04"** (@andiradulescu, merged 2024-08-13) — 24.04 landed in agnos-builder master. Related: #235 "Ubuntu 24.04 branch fixes" (merged 2024-07-05), #247 "Build kernel in docker", #419 "Fix NetworkManager crashes on 24.04".
- openpilot **PR #33775 "AGNOS 11"** (@adeebshihadeh, merged 2024-10-13) — shipped the 24.04-based AGNOS to openpilot master (PR description: "* Ubuntu 20.04 -> 24.04", https://github.com/commaai/openpilot/pull/33775), incl. kernel support for ISP debayering and boot-time improvements; release checklist included `test_onroad` passing.
- Current openpilot master pins `AGNOS_VERSION="18.7"` (launch_env.sh), i.e. the 24.04-based AGNOS line is the shipping OS.

## What remains (Part 2: mainline kernel, $2k)

Canonical tracker: **commaai/agnos-builder PR #233 "Kernel mainline"** (@andiradulescu, open, last updated 2026-03-27). Kernel tree: https://github.com/andiradulescu/linux (branch `sdm845/6.9-release`, based on postmarketOS SDM845 mainline + @robin-reckmann's work).

Task list status (from PR #233 body):
- ✅ Done: near-mainline base from postmarketOS; robin-reckmann work applied; **C3 (tici) dtb**; ufs; display; i2c; pcie/nvme
- ❌ Remaining: **C3X (tizi) dtb, C3X (mici) dtb**, wifi, usb, modem, sound, **cameras**, graphics (OpenGL), OpenCL via rusticl/msm_drm (no kgsl), weston (drm)

Validation gates (all unchecked in PR #233):
1. dmesg is clean (background: agnos-builder issue #325)
2. **`test_onroad` passes** — per the bounty issue, this is "a good litmus test for functionality while developing"
3. **Internal testing closet** — once test_onroad passes reliably on a single device, @adeebshihadeh deploys to several devices in comma's internal testing closet (final step, maintainer-controlled)

Camera note: @adeebshihadeh (2024-08-11) — "the mainline kernel has a different driver and API for the camera. All camerad work should focus on making that port easy." Related open openpilot PR #37454 "camerad: setup IFE sharing" (@adeebshihadeh).

## Recent activity signals (2025–2026)

- 2025-07-16: bounty **unlocked**; @workinright expressed intent (requested remote device access; no comma-org PRs from them since — only unrelated CI-speedup PRs in openpilot).
- 2026-07: comma staff @greatgitsby has three open mainline bring-up PRs in agnos-builder:
  - #607 "wifi: add userspace linux-msm daemons for mainline linux"
  - #608 "lte/gpio: resolve TLMM pins against the gpiochip base" (references vamOS userspace)
  - #609 "sound: bring up the mici audio card on the mainline kernel"
- Issue is under milestone "project vamOS" (https://github.com/commaai/vamOS) — mainline-kernel work appears to have been folded into comma's internal vamOS project.
- 2026-06-07: @Gooseboy2234 asked what the payout is for landing kernel mainlining + validation given 24.04 is done — **no maintainer reply yet** (latest issue activity).

## Claimability note

- The issue text says "Locked to @andiradulescu," but the maintainer explicitly **unlocked** it on 2025-07-16. There is no current assignee on the issue.
- Nevertheless: the $2k payout terms for a third party are unconfirmed (open question from @Gooseboy2234), and comma staff are actively shipping mainline pieces under project vamOS. **This package is informational only; it does not propose or initiate any competing work.** Anyone considering a claim should get explicit confirmation from comma first.

## Sources

- Issue: https://api.github.com/repos/commaai/openpilot/issues/32386 (body, lock text, bounty split)
- Comments: https://api.github.com/repos/commaai/openpilot/issues/32386/comments (unlock 2025-07-16; camera API note; payout question 2026-06-07)
- https://github.com/commaai/agnos-builder/pull/233 (Kernel mainline tracker + task list)
- https://github.com/commaai/agnos-builder/pull/262 (Ubuntu 24.04, merged 2024-08-13)
- https://github.com/commaai/openpilot/pull/33775 (AGNOS 11 = 24.04 to master, merged 2024-10-13)
- https://github.com/commaai/agnos-builder/pulls/607, /608, /609 (greatgitsby mainline bring-up, open)
- https://github.com/commaai/openpilot/blob/master/launch_env.sh (AGNOS_VERSION 18.7)
