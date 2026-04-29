# L1 Troubleshooting: DepChain Devnet Halt

> Source: Telegram export at `/Users/nicolas.arnedo/depchain.json`, analyzed on 2026-04-28.
> Status: open / unresolved in the captured chat.

## Case Overview

DepChain reported a Fuji subnet-evm L1 halt after validator balance issues were corrected. The chain still answers RPC requests and accepts transactions into the mempool, but no new blocks are produced. The main symptom is a ProposerVM fallback scheduling delay of roughly 60 minutes.

This case is related to the MetaDOS recovery playbook because it involves Etna-style L1 validator state, Warp/ValidatorManager context, P-Chain validator balance or registration state, and ProposerVM proposer selection. Unlike MetaDOS, current public P-Chain state shows three active validators for the subnet, so the exact "validator missing from P-Chain set" failure is not established.

## Chain Details

| Field | Value |
| --- | --- |
| Network | Fuji |
| Blockchain name | Depchain Devnet |
| Blockchain ID | `2o8s1eAmYBTqgnQ4wrYiLanHF926N7KUnkEbA2xp5ErN5dDdpf` |
| EVM Chain ID | `90108` |
| Subnet ID | `28SmrMWyGVswhYPtetX8wBzxVJ1gXqsiX4QdsExfggzdvbWB4U` |
| VM ID | `srEXiWaHuhNyGwPUi444Tu47ZEDwxTWrbQiuD7FmgSAQ6X7Dy` |
| VM | subnet-evm |
| AvalancheGo | `v1.14.0`, commit `b4cae25c024cce8606062e7fc8559f9d4a4a4b4a` |
| Subnet-EVM | `v0.8.0-fuji@46ff1ccb` |
| Last accepted height | `183` |
| Reported halt time | 2026-04-19 01:07 UTC |
| Warp genesis config | `blockTimestamp: 1776324096`, `quorumNumerator: 67`, `requirePrimaryNetworkSigners: true`, `enabled: true` |
| Chain create metadata | block timestamp `1776324235`, block number `273868` |

Current public P-Chain state checked on 2026-04-28:

| NodeID | Weight | Balance | Validation ID |
| --- | ---: | ---: | --- |
| `NodeID-4sQcbcd4zxzqqHa6cHsTAyLSW5KcemJUm` | `20100` | `135916928` | `2yxP569iY72jGhBTo2eo9jdF8mC8fPZbWPntbC6SfcCT3oBDF` |
| `NodeID-APnoF9k1hoYXXM9Dyty45KtiqRiJG9PVE` | `100000` | `85907200` | `nhuD5QWuh6zxyFNYN6jMdwMCqE7dE8GtPfxVxceS5oN5qhzaS` |
| `NodeID-AZUKXf87fLGPMLUCmWftppuCeDxVrZjV7` | `1000` | `465398784` | `2UZoHqp7dtMHxRCvrb8qvw6jdr7CQC7AKQfv9hbEHAJK4LMtCt` |

The largest validator currently has about 82.6% of active P-Chain weight.

## 2026-04-29 Follow-Up Checks

The Telegram export was re-scanned for the remaining missing fields. It does not include:

- A direct DepChain L1 RPC URL.
- ValidatorManager, StakingManager, VMC, or deployer contract addresses.
- P-Chain transaction IDs for the balance refill, validator add/remove, stake, unstake, or claim operations.
- Full `platform.getValidatorsAt` responses or the exact P-Chain heights queried.
- Full validator logs or chain config files.

Additional public checks:

- `platform.getValidatorsAt` with `height: "proposed"` on public Fuji P-Chain returns the same three validators and weights as current state.
- `platform.getCurrentValidators` on public Fuji P-Chain shows all three validator balances are currently nonzero.
- Public `api.avax-test.network` does not expose the custom L1 RPC for this blockchain, so L1 block/header/state checks require a DepChain RPC or access to one of their nodes.
- Glacier exposes genesis/precompile metadata and shows native minter and reward manager admin address `0x90b93F1708ea054E8980830E4889b3dE1447514E`, but this is not enough to identify the ValidatorManager, staking manager, or deployer workflow.

Version nuance:

- The chat reports AvalancheGo commit `b4cae25c024cce8606062e7fc8559f9d4a4a4b4a` and subnet-evm commit `46ff1ccb43a56385d1a63eab187a9d94272171b0`, which correspond to the Fuji pre-release pair `v1.14.0-fuji` and `v0.8.0-fuji`.
- The stable matching Granite pair is AvalancheGo `v1.14.0` with subnet-evm `v0.8.0` using VM plugin protocol version 44.
- Docker image for the stable matching pair exists as `avaplatform/subnet-evm_avalanchego:v0.8.0_v1.14.0`.
- Docker image for their reported Fuji pre-release pair is `avaplatform/subnet-evm_avalanchego:v0.8.0-fuji_v1.14.0-fuji`.
- AvalancheGo `v1.14.2` is the latest GitHub release checked on 2026-04-29, but it updates the plugin protocol to 45. Do not recommend upgrading only AvalancheGo to `v1.14.2` while keeping the external subnet-evm `v0.8.0` plugin unless a compatible plugin path is confirmed.

Claude comparison / refined diagnostic stance:

- Claude's halt summary matches the chat: known L1 last accepted height is `183`; what remains missing is block `183`'s timestamp, hash, and referenced P-Chain height/view.
- The trigger sentence about validators running out of P-Chain AVAX is the strongest clue.
- Since current public P-Chain state shows nonzero balances, the leading hypothesis is not "a validator is still balance zero now"; it is that block `183` references a historical P-Chain view where one or more validators were balance zero or otherwise ineligible.
- First diagnostic should be lightweight and read-only: block `183`, chain config, upgrade config, current/proposed validator set, and logs.
- Assume the team is running Docker unless they say otherwise; provide Docker-oriented commands and include a secret-handling warning.

## Timeline

### 2026-01-28 to 2026-01-30: Fee Distribution Design

DepChain asked whether subnet-evm can mimic Ethereum fee behavior: burn base fee while sending only priority fees to validators. They had found that default subnet-evm burns both base fee and priority fee, while `allowFeeRecipients` sends both to the validator.

Initial guidance explored Reward Manager and `setValidatorRewardAddress`, but after engineering review the conclusion was that base-fee-burn plus priority-fee-to-validator would require subnet-evm core changes. The team later decided to send all transaction fees to validators, avoiding a subnet-evm fork.

### 2026-01-30 to 2026-02-06: Builder Hub / L1 Setup Issues

DepChain moved from Avalanche CLI to Builder Hub Console because Avalanche CLI appeared stale and fee manager configuration was panicking.

Reported setup issues:

- Glacier / Builder Hub Console `aggregateSignature` API returned 500 during Initialize Validator Set.
- Blockscout could not sync, likely because the RPC node was not syncing.
- The staking flow appeared to use C-Chain and P-Chain transactions instead of direct L1 transactions, which affected their intended business logic.
- Toolkit support for deploying staking contracts on the L1 was later reported as resolved.

### 2026-02-12 to 2026-03-02: Reward Calculator Behavior

DepChain asked whether changes to the reference reward calculator's `rewardBasisPoints` are retroactive for the full staking period. Engineering confirmed the reference implementation calculates rewards using the current setting at claim time, so changes apply retroactively. The implementation was framed as a reference, not a production-grade accounting system.

### 2026-03-27 to 2026-04-17: Signature Aggregator Reliability

DepChain repeatedly hit Glacier signature aggregator 500s when completing the P-Chain update step in staking and unstaking flows.

Endpoint in use:

```text
https://glacier-api.avax.network/v1/signatureAggregator/fuji/aggregateSignatures
```

The team suspected indexing delays of hours or days. Ash clarified that Fuji P-Chain itself was not understood to be unstable in that way, but the public signature aggregator tooling had reliability issues. Ava Labs was planning an internal signature aggregator fallback. For validator count, guidance was 1-2 validators for testnet, preferably 2, and 3-5 for mainnet depending on uptime guarantees.

### 2026-04-20 to 2026-04-22: Current Halt

DepChain reported the L1 had halted:

- RPC still returns chain data.
- Transactions can be submitted and appear to enter the mempool.
- No blocks are built.
- Halt began around 2026-04-19 01:07 UTC.
- Last accepted height is `183`.
- Logs show ProposerVM failing to fetch the expected proposer delay, then scheduling the next build around 59m59s later.

Representative log behavior from the chat:

```text
failed to fetch the expected delay: context canceled
```

Then ProposerVM falls back to a roughly 60-minute delay, consistent with `MaxLookAheadSlots * WindowDuration`.

DepChain noted that the same behavior occurs on the validator holding about 82.6% of stake, so their expectation was that proposer sampling should still select that validator. They also suspected the issue may be connected to validators running out of P-Chain AVAX balance during the previous week and then being refilled.

DepChain said `platform.getValidatorsAt` at the relevant P-Chain heights showed all three validators active, with public keys and correct weights. The captured chat does not include the exact heights, responses, or last accepted block metadata, so this remains unverified from the export alone.

## What Has Been Tried

- Moved L1 setup workflow from Avalanche CLI to Builder Hub Console.
- Worked around or waited on Builder Hub / Glacier signature aggregator issues.
- Resolved toolkit support for deploying staking contracts on the L1.
- Increased validator balances after validators reportedly ran out of P-Chain AVAX.
- Checked `platform.getValidatorsAt` and reported that all three validators were active at relevant P-Chain heights.
- Ran with AvalancheGo `v1.14.0` and subnet-evm `v0.8.0-fuji`.
- Collected ProposerVM debug logs showing the 59m59s fallback delay.

No final resolution is present in the chat export.

## Working Hypotheses

### 1. ProposerVM / P-Chain height view mismatch after validator balance refill

This is the closest match to the MetaDOS class of problems. If validators had zero L1 validator balance at the P-Chain height referenced by the last accepted L1 block, later `IncreaseL1ValidatorBalanceTx` transactions may not be visible to ProposerVM until the L1 produces a block. That creates a chicken-and-egg failure: the later funded validator set exists on the current P-Chain, but the halted chain's proposer selection may still be evaluating an older P-Chain view.

This maps more closely to MetaDOS Scenario A, where validators are present but balance or effective eligibility is wrong, rather than Scenario B, where a validator is missing and needs off-chain BLS registration.

### 2. AvalancheGo v1.14.0 L1 conversion / proposer scheduling bug

MetaDOS was also on the early Etna/L1 conversion path and was unblocked by moving past `v1.14.0` plus correcting validator state. DepChain is still on `v1.14.0`. Upgrading to the latest Fuji-compatible AvalancheGo and subnet-evm should be treated as an early mitigation and as required context before deeper escalation.

### 3. ProposerVM configuration or fallback behavior

The 59m59s delay resembles the default long fallback delay seen in MetaDOS. In this case, the log includes a failure to fetch expected delay, so the delay may be a symptom of failed proposer lookup rather than merely a missing `proposervm-block-delay-max` setting. Still, the node chain config should be collected and checked.

### 4. Warp / ValidatorManager state issue

Glacier metadata shows Warp enabled with a timestamp before chain creation, so the exact MetaDOS wrong-future-Warp-activation issue is not obvious from public metadata. This still needs direct checks against the last accepted block timestamp and ValidatorManager state, especially `totalWeight()`.

### 5. Signature aggregator instability as a separate issue

The signature aggregator 500s explain broken staking and unstaking UX, but they do not by themselves explain why an already active validator set cannot build blocks. They may be indirectly related if a registration, removal, or balance update was partially completed around the halt.

## Missing Information

Collect this before deciding whether DepChain needs a MetaDOS-style emergency transaction:

- Full last accepted L1 block header: block hash, timestamp, height, and referenced P-Chain height.
- The exact `platform.getValidatorsAt` responses for the P-Chain height referenced by the last accepted block and for the current P-Chain height.
- Current `platform.getCurrentValidators` response including balances, public keys, weights, and validation IDs.
- P-Chain transaction IDs for all validator balance increases, validator adds, removals, stake, unstake, and claim operations around 2026-04-19.
- Confirmation of each validator's L1 validator balance before and after the refill.
- Full logs from all validators around 2026-04-19 01:07 UTC, especially lines containing:

```text
proposervm
expected delay
windower
context canceled
build
validator
balance
warp
facade
```

- Each validator's chain config at:

```text
~/.avalanchego/configs/chains/2o8s1eAmYBTqgnQ4wrYiLanHF926N7KUnkEbA2xp5ErN5dDdpf/config.json
```

- AvalancheGo and subnet-evm versions on every validator.
- `info.isBootstrapped` and peer connectivity for the L1 on every validator.
- ValidatorManager and staking contract addresses, deployment chain, owner/admin, and current contract state.
- Direct calls to ValidatorManager state, especially `totalWeight()`.
- Whether any chain config, upgrade config, or Warp timestamp changed near the halt.
- A direct L1 RPC URL so `eth_blockNumber`, pending transaction count, and precompile calls can be verified.

## Suggested Next Steps

1. Preserve validator databases and logs before restarting, resyncing, or deleting state.
2. Upgrade AvalancheGo and subnet-evm to the latest Fuji-compatible versions, then restart validators one at a time.
3. Verify validator eligibility at the P-Chain height referenced by the last accepted L1 block, not only at current P-Chain height.
4. If validators were present but had zero or insufficient balance at that historical height, escalate as a ProposerVM/P-Chain view mismatch after balance refill.
5. If ValidatorManager `totalWeight()` is zero or validators are missing from contract state, compare against the MetaDOS Scenario A/B decision tree.
6. Treat public Glacier signature aggregator 500s as a separate reliability problem and plan a fallback aggregator path for staking and unstaking operations.

## Builder Hub Documentation Candidate

This case should likely become a short Builders Hub runbook:

- Title: `L1 halted: no blocks after validator balance, registration, or conversion change`
- Explain why current P-Chain validator state is not enough; operators must check the validator set at the P-Chain height referenced by the last accepted L1 block.
- Include a checklist for ProposerVM 59m59s fallback delay.
- Include the Scenario A/B split from MetaDOS:
  - Validator exists but balance/effective eligibility is wrong: use balance/top-up path.
  - Validator is missing: off-chain BLS registration path may be required.
- Warn that public signature aggregator failures can block staking UX but may be a separate problem from block production.
- Recommend capturing logs, P-Chain tx IDs, last block header, and ValidatorManager state before restarts or resync attempts.
