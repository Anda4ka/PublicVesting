# PublicVesting — Permissionless Fee-to-Vesting on Bitcoin L1

Anyone with OP_20 tokens can create vesting schedules for any beneficiary in one transaction. Revenue from deposited fees is shared proportionally to all locked-token holders via a Synthetix O(1) accumulator — no loops, no owner gatekeeping.

**Live:** https://public-vesting.vercel.app/

---

## Features

- **`depositAndVest(amount, beneficiary, cliff, duration)`** — fully public, no owner required
- **`depositRevenue(amount)`** — any address can forward protocol fees for proportional distribution
- **`claimRevenue()`** — beneficiaries claim accumulated revenue share
- **`release()`** — beneficiary releases all currently vested tokens after cliff
- Synthetix O(1) reward-per-token accumulator — no iteration over holder lists
- `StoredBoolean` reentrancy guard (persistent storage, survives cross-contract calls)
- Strict CEI (Checks-Effects-Interactions) on every state-changing method
- Single OP_20 token for both vesting and revenue distribution

---

## Contract Methods

| Method | Access | Description |
|---|---|---|
| `initialize(revenueToken)` | Owner, once | Set the OP_20 token address |
| `depositAndVest(amount, beneficiary, cliff, duration)` | Public | Lock tokens, create vesting schedule |
| `release()` | Beneficiary | Release vested tokens |
| `depositRevenue(amount)` | Public | Deposit revenue for proportional distribution |
| `claimRevenue()` | Beneficiary | Claim accumulated revenue share |
| `getVesting(address)` | View | Returns full vesting schedule for address |
| `getPendingRelease(address)` | View | Returns currently releasable amount |
| `getClaimableRevenue(address)` | View | Returns claimable revenue for address |
| `totalLocked()` | View | Total tokens locked across all schedules |
| `totalRevenueDeposited()` | View | Cumulative revenue ever deposited |
| `revenueToken()` | View | Token address |
| `owner()` | View | Contract owner |

---

## Build & Deploy

### Prerequisites

```bash
npm install
```

### Compile

```bash
npm run build          # debug build  →  build/feevest.debug.wasm
npm run build:release  # release build → build/feevest.release.wasm
```

### Deploy to OPNet Testnet

```bash
# Deploy contract (no constructor calldata — uses initialize() pattern)
opnet deploy \
  --network testnet \
  --wasm build/feevest.release.wasm \
  --gasSatFee 10000

# After deployment, call initialize once as owner:
# feeVest.initialize(revenueTokenAddress)
```

> **Note:** OPNet testnet delivers 0 bytes to `onDeploy()`, so token setup is done via the one-time `initialize()` call post-deployment. Use the Admin tab in the dashboard.

### Minimum gas

Use `gasSatFee: 10_000n` (or higher for large WASM). Values below this may revert silently.

---

## Architecture

```
depositAndVest()
  ├─ CHECKS:  validate inputs, verify no existing schedule
  ├─ EFFECTS: updateReward snapshot, write schedule, increase totalLocked
  └─ INTERACT: transferFrom(depositor → vault)

depositRevenue()
  ├─ CHECKS:  amount > 0, totalLocked > 0
  ├─ EFFECTS: rewardPerToken += (amount × 1e18) / totalLocked
  └─ INTERACT: transferFrom(depositor → vault)

release()
  ├─ CHECKS:  has schedule, releasable > 0
  ├─ EFFECTS: updateReward, mark released, decrease totalLocked, re-anchor debt
  └─ INTERACT: transfer(vault → beneficiary)

claimRevenue()
  ├─ CHECKS:  has schedule
  ├─ EFFECTS: updateReward, zero pendingRewards
  └─ INTERACT: transfer(vault → beneficiary)
```

---

## Security

- Audit: 0 critical — Bob MCP + manual review
- `StoredBoolean` reentrancy guard (not a class field — persists across call frames)
- No `tx.origin` — only `Blockchain.tx.sender`
- All u256 arithmetic via `SafeMath`
- No loops over unbounded data — O(1) everywhere
- One active schedule per beneficiary (griefing mitigation)

---

## License

MIT
