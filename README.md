# comma-bounty-packages

Swarm-produced research & draft packages for the [comma.ai bounty board](https://github.com/commaai/openpilot/issues?q=label%3Abounty) (comma.ai bounty board #26), generated 2026-07-25.

**Disclaimer:** For the hardware-blocked bounties below, this is **research groundwork only — no completion is claimed**. For the code drafts, work exists as branches on our forks but **PRs to commaai are currently blocked by a commaai org block on this account**. In all cases, **merge and payout decisions are entirely comma's call**.

## The 9 bounties

### Code drafts (branches on our forks)

| # | Bounty | Status | Code |
|---|--------|--------|------|
| 1 | [opendbc#2557](https://github.com/commaai/opendbc/issues/2557) — Enable 100% branch coverage check | **Complete local draft**; independent review passed (3981 tests OK, 1930/1930 branches 100%, gcovr exit 0) | branch [`safety-branch-coverage-clean`](https://github.com/VeigaPunk/opendbc/tree/safety-branch-coverage-clean) on VeigaPunk/opendbc |
| 2 | [openpilot#32425](https://github.com/commaai/openpilot/issues/32425) — test_models: fuzz the tx messages | Local draft only; targets commaai/opendbc (test_models moved there); no completion claimed | branch [`tx-fuzzy-test-clean`](https://github.com/VeigaPunk/opendbc/tree/tx-fuzzy-test-clean) on VeigaPunk/opendbc |
| 3 | [openpilot#30950](https://github.com/commaai/openpilot/issues/30950) — Match stock Hyundai button logic | Local draft only; **requires real-car test routes** (HKG CAN + CAN-FD) before merge; not claimed | branch [`hyundai-stock-button-logic`](https://github.com/VeigaPunk/opendbc/tree/hyundai-stock-button-logic) on VeigaPunk/opendbc |
| 4 | [openpilot#30693](https://github.com/commaai/openpilot/issues/30693) — Run MetaDrive simulation test in GitHub Actions | Local draft only; the 20/20 CI reliability requirement must be proven on commaai's own CI before claiming | branch [`metadrive-ci-sim`](https://github.com/VeigaPunk/openpilot/tree/metadrive-ci-sim) on VeigaPunk/openpilot |
| 5 | [openpilot#33207](https://github.com/commaai/openpilot/issues/33207) — MetaDrive simulator on macOS | Analysis + **UNVALIDATED** draft patch; no macOS hardware was available, nothing run on a Mac; not claimed | branch [`metadrive-macos`](https://github.com/VeigaPunk/openpilot/tree/metadrive-macos) on VeigaPunk/openpilot |

### Research groundwork only (hardware-blocked — no completion claimed)

| # | Bounty | Status | Package |
|---|--------|--------|---------|
| 6 | [opendbc#3426](https://github.com/commaai/opendbc/issues/3426) — Ford F-150 2026 (TRON) support ($10k) | Research only; blocked on physical TRON-platform vehicle + harness prototyping | [docs/opendbc-3426-ford-tron.md](docs/opendbc-3426-ford-tron.md) |
| 7 | [openpilot#30302](https://github.com/commaai/openpilot/issues/30302) — Ford Lightning Q4 harness faults when openpilot not running ($200) | Research only (root-cause groundwork); blocked on F-150 Lightning hardware; issue is locked upstream | [docs/openpilot-30302-lightning-harness.md](docs/openpilot-30302-lightning-harness.md) |
| 8 | [openpilot#30894](https://github.com/commaai/openpilot/issues/30894) — Reduce kernel boot time to <1s on comma 3X | Research only (config/DT change map, expected savings, measurement plan); blocked on physical comma 3X | [docs/openpilot-30894-kernel-boot.md](docs/openpilot-30894-kernel-boot.md) |
| 9 | [openpilot#32386](https://github.com/commaai/openpilot/issues/32386) — Ship Ubuntu 24.04 + mainline kernel to master | **Status research package only** — documents that the maintainer lock was lifted 2025-07-16 ("Unlocked — good luck!"); no competing work proposed | [docs/openpilot-32386-mainline-kernel.md](docs/openpilot-32386-mainline-kernel.md) |

## Notes

- Detailed code-bounty cross-references: [docs/INDEX.md](docs/INDEX.md).
- **PR blockage:** we cannot open PRs against commaai/opendbc or commaai/openpilot because this account is blocked from the commaai org. The branches above are public on our forks; comma can fetch and review them directly.
- Nothing in this repo asserts bounty eligibility or payout. All judgments are comma's.
