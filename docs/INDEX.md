# Code bounty index

Cross-reference from bounty issues to the branches containing the draft code.
All branches live on our forks; PRs to commaai are blocked by a commaai org block on the account.

| Bounty issue | Fork | Branch |
|---|---|---|
| [commaai/opendbc#2557](https://github.com/commaai/opendbc/issues/2557) — Enable 100% branch coverage check | [VeigaPunk/opendbc](https://github.com/VeigaPunk/opendbc) | [`safety-branch-coverage-clean`](https://github.com/VeigaPunk/opendbc/tree/safety-branch-coverage-clean) |
| [commaai/openpilot#32425](https://github.com/commaai/openpilot/issues/32425) — test_models: fuzz the tx messages | [VeigaPunk/opendbc](https://github.com/VeigaPunk/opendbc) | [`tx-fuzzy-test-clean`](https://github.com/VeigaPunk/opendbc/tree/tx-fuzzy-test-clean) |
| [commaai/openpilot#30950](https://github.com/commaai/openpilot/issues/30950) — Match stock Hyundai button logic | [VeigaPunk/opendbc](https://github.com/VeigaPunk/opendbc) | [`hyundai-stock-button-logic`](https://github.com/VeigaPunk/opendbc/tree/hyundai-stock-button-logic) |
| [commaai/openpilot#30693](https://github.com/commaai/openpilot/issues/30693) — MetaDrive sim test in GitHub Actions | [VeigaPunk/openpilot](https://github.com/VeigaPunk/openpilot) | [`metadrive-ci-sim`](https://github.com/VeigaPunk/openpilot/tree/metadrive-ci-sim) |
| [commaai/openpilot#33207](https://github.com/commaai/openpilot/issues/33207) — MetaDrive simulator on macOS | [VeigaPunk/openpilot](https://github.com/VeigaPunk/openpilot) | [`metadrive-macos`](https://github.com/VeigaPunk/openpilot/tree/metadrive-macos) |

Notes:

- The tx-fuzzy (#32425) and Hyundai button-logic (#30950) drafts target the **opendbc** repo even though the issues are filed against openpilot, because the car ports, `test_models.py`, and panda safety modes now live in commaai/opendbc (consumed by openpilot via the `opendbc_repo` submodule).
- Research-only (hardware-blocked) packages have no code branch; see the markdown files in this directory.
- No completion is claimed for unvalidated drafts; merge/payout is comma's call.
