# Roobak Contracts — Security Review

| | |
|---|---|
| **Project** | Roobak — Uniswap v4 Swap-Rebate Hook |
| **Engagement type** | Source-code security review (static + dynamic) |
| **Scope** | `roobak-contracts` release tree — 39 files, 15 Solidity source contracts (2,962 sloc), 9 test files (1,483 sloc), 5 deployment scripts |
| **Scope identification** | SHA-256 `ce965b91d56506b683289ac83972fd288fa0d1812ed13dc42cca014ccc2468f0` (release archive `roobak-contracts-github.zip`) |
| **Toolchain** | Solidity `0.8.26`, EVM `cancun`, `via_ir`, Foundry (`forge build / test / fmt`) |
| **Dependencies (pinned)** | Uniswap `v4-core` v4.0.0 · `v4-periphery` @ `c6e4754` · OpenZeppelin Contracts v5.7.0 · `forge-std` v1.17.0 |
| **Review date** | 2026-11-17 |
| **Prepared by** | Internal security review (AI-assisted static analysis and dynamic testing) |
| **Classification** | Confidential — publish at the project's discretion |

---

## Important Notice

This report documents an **internal, AI-assisted security review** performed against a
specific snapshot of the source code identified by the hash above. It is **not** an
audit by an independent third-party firm and must not be presented as one. Security
review is inherently limited: it cannot guarantee the absence of vulnerabilities; it
can only record what was examined, how, and what was found. The findings below reflect
the reviewers' best effort at the time of review, under the stated methodology and
limitations. Neither the reviewers nor the project can warrant the correctness or
security of the software. **This codebase is an unaudited testnet reference
implementation and is not recommended for mainnet deployment carrying real value
without an independent external audit.**

---

## 1. Executive Summary

Roobak is a Uniswap v4 hook system that backruns the price divergence created by every
swap in its attached pools — atomically, inside the same transaction — and distributes
the *realized* profit between the trader, the liquidity providers, an optional
impermanent-loss compensation fund, and (for keeper-triggered captures) a searcher
commission.

The review covered the full protocol surface: the hook and its capture engine, the
registry that gates everything the engine may touch, the escrow ledger and IL fund,
the router, the order book, the searcher router, the two venue implementations, and
the deployment scripts. Every source contract was read line by line; the execution
paths (swap → capture → split → escrow → claim/donate) were traced end to end; the
invariants claimed by the documentation were checked against the code that enforces
them; and the full Foundry test suite (63 tests, 8 suites) was executed against the
reviewed snapshot — **all passing**.

**Overall assessment: MEDIUM risk, conditional on a trusted operator.**

No Critical findings were identified. The core accounting design is sound: payouts are
computed exclusively from realized, positive, post-execution balance deltas; escrow
precedes credit in the ledger; the IL fund is exactly solvent by construction; and
every loop, route and cycle the hook may run is bounded by registry parameters backed
by hard caps. The most significant exposure is not a exploit path available to
anonymous attackers — it is **concentration of privileged power**: a single owner key
controls the hook's working capital, the venue allowlist (backed by infinite token
approvals), and every economic parameter, with no timelock or escape hatch. Three
Medium findings concern non-standard token handling, rebate-recipient semantics for
integrators, and unbounded gas forwarding into the revert-isolated capture path.

### Finding distribution

| Severity | Count | IDs |
|---|---|---|
| Critical | 0 | — |
| High | 1 | H-1 |
| Medium | 3 | M-1, M-2, M-3 |
| Low | 3 | L-1, L-2, L-3 |
| Informational | 3 | I-1, I-2, I-3 |

### Verdict by subsystem

| Subsystem | Verdict |
|---|---|
| Capture engine (hook, RouteChecker, SwapExecutor) | Sound design; bounds enforced; gas-forwarding and venue-trust caveats (M-3, H-1) |
| Accounting (PayoutLedger, ILFund) | Exactly solvent by construction for standard ERC-20s; non-standard tokens break solvency (M-1) |
| Governance (PoolRegistry) | Complete parameter validation; fully centralized control (H-1) |
| Periphery (SwapRouter, OrderBook, SearcherRouter) | Clean; reentrancy handled; minor quality issues (L-4) |
| Venues (ExternalAMMPool, BatchAuctionPool) | ExternalAMMPool acceptable as a demo venue; BatchAuctionPool explicitly experimental with known manipulation surface (L-3) |
| Fixtures (TestToken, WrappedNative, Create2Factory) | Testnet-only / minimal; do not use in production |

---

## 2. Scope

All 15 source contracts under `src/` were in scope. Tests and scripts were reviewed
for behavior claims and operational safety. External dependencies (v4-core,
v4-periphery, OpenZeppelin, forge-std) were reviewed only as used, not audited.

| Contract | File | sloc | Role | Class |
|---|---|---|---|---|
| `RebateHook` | `src/hook/RebateHook.sol` | 570 | v4 hook: capture loop, dynamic fees, keeper entries, working capital | Core |
| `PoolRegistry` | `src/registry/PoolRegistry.sol` | 423 | Approved pairs/pools/venues, parameters, pauses | Core |
| `BatchAuctionPool` | `src/periphery/BatchAuctionPool.sol` | 348 | Experimental uniform-price batch venue | Periphery (experimental) |
| `OrderBook` | `src/periphery/OrderBook.sol` | 295 | Escrowed conditional orders, permissionless fills | Periphery |
| `SwapRouter` | `src/periphery/SwapRouter.sol` | 272 | Minimal non-custodial router | Periphery |
| `ILFund` | `src/rewards/ILFund.sol` | 221 | Liquidity×time-weighted LP compensation | Core |
| `PayoutLedger` | `src/rewards/PayoutLedger.sol` | 187 | Realized-profit escrow and split accounting | Core |
| `RouteChecker` | `src/libraries/RouteChecker.sol` | 171 | Bounded route enumeration, closed-form estimation | Core |
| `ExternalAMMPool` | `src/periphery/ExternalAMMPool.sol` | 141 | Demo constant-product external venue | Periphery (demo) |
| `SwapExecutor` | `src/libraries/SwapExecutor.sol` | 135 | Protected atomic cycle execution | Core |
| `WrappedNative` | `src/tokens/WrappedNative.sol` | 74 | WETH9-style wrapper | Fixture |
| `SearcherRouter` | `src/periphery/SearcherRouter.sol` | 48 | Batched permissionless capture submissions | Periphery |
| `IExternalVenue` | `src/interfaces/IExternalVenue.sol` | 29 | Venue integration surface | Interface |
| `Create2Factory` | `src/periphery/Create2Factory.sol` | 26 | Deterministic deployments (hook salt mining) | Fixture |
| `TestToken` | `src/tokens/TestToken.sol` | 22 | ERC-20 with public mint | Fixture (testnet only) |

**Out of scope:** the four vendored dependencies; the web front end; off-chain keeper
infrastructure; anything deployed on any chain (no deployed bytecode was reviewed —
the review is of the source tree only).

---

## 3. Methodology

1. **Manual line-by-line review** of every in-scope source file, including NatSpec
   cross-checking against actual behavior.
2. **Execution-path tracing.** The full lifecycle of a capture was traced through the
   call graph: user swap → `afterSwap` → revert-isolated `runCapture` → venue
   snapshot → plan → caps → `SwapExecutor.executeCycle` (v4 take/settle legs and
   external venue legs) → realized-profit measurement → `_pay` split → ledger escrow →
   `claim` / `releaseToHook` → `donate`. The keeper path (`backrun` →
   `unlockCallback`) and the donation path were traced separately.
3. **Invariant analysis.** Each documented invariant (ledger solvency, IL-fund
   solvency, base neutrality, bounds, single capture per swap, capture isolation)
   was checked against the code paths that could violate it, including rounding and
   overflow analysis of the `RouteChecker` closed-form math (magnitude guards
   `MIN_SQRT`/`MAX_SQRT`/`MAX_LIQUIDITY`, `Math.mulDiv` usage).
4. **Privilege and trust-model analysis.** Every `onlyOwner` / binding / approval
   surface was enumerated and assessed (Section 8).
5. **Adversarial reasoning.** Reentrancy (including via malicious registered venues
   and during batch settlement), oracle/estimate manipulation, gas griefing,
   sandwiching of permissionless fills and keeper backruns, fee-on-transfer and
   rebasing tokens, and rounding exploitation were each considered against the code.
6. **Dynamic testing.** `forge build`, `forge test` (63/63 passing), `forge fmt
   --check` against the exact reviewed snapshot.

**Limitations.** No formal verification, no stateful fuzzing/invariant fuzzing
(e.g., Echidna/Medusa), no fork simulation of adversarial block builders, and no
review of deployed bytecode or of the vendored dependencies' internals were
performed. Findings are reported against source, assuming the standard EVM/Cancun
semantics and honest v4-core behavior.

### Severity scale

| Severity | Definition |
|---|---|
| Critical | Direct loss or freeze of user/protocol funds exploitable by any party, or trivially broken core accounting |
| High | Loss/theft of funds or protocol break requiring a plausible condition (including compromise or malicious action of a privileged key) |
| Medium | Context-dependent break: solvency/availability failure with non-standard assets, integration trust leaks, or griefing with outsized impact |
| Low | Edge-case incorrectness, misleading behavior, or code-quality issues with limited blast radius |
| Informational | Notes on design intent, documentation, and hardening opportunities |

---

## 4. Findings Summary

| ID | Severity | Title | Status |
|---|---|---|---|
| H-1 | High *(privileged)* | Unilateral owner custody of working capital, venue allowlist with infinite approvals, and all economics — no timelock, no two-step handover | Acknowledged (design) |
| M-1 | Medium | Fee-on-transfer / rebasing quote tokens break ledger solvency | Open |
| M-2 | Medium | `hookData` recipient lets any executing router or aggregator silently capture the trader's rebate | Open |
| M-3 | Medium | Capture loop forwards `gasleft()` with no reserve — an approved venue can OOG-bleed into the user's swap | Open |
| L-1 | Low | `withdrawExcessCapital` underflows (`Panic(0x11)`) when balance is below the floor instead of reverting cleanly | Open |
| L-2 | Low | External *cheap* leg executes with `minOut = 0` | Open |
| L-3 | Low | BatchAuctionPool (experimental): pre-settlement reserve manipulation, post-deadline order injection, owner drain mid-batch | Acknowledged (experimental) |
| L-4 | Low | Misleading error names and an incorrect NatSpec mask constant | Open |
| I-1 | Informational | Native-currency pairs are structurally excluded — document as a supported-asset policy | Open |
| I-2 | Informational | A pool shared by pairs with different quote currencies will revert captures (`CurrencyMismatch`) — configuration constraint | Open |
| I-3 | Informational | Rounding policy favors the escrow/fund (positive note); per-venue gas ceiling would harden M-3 | Open |

---

## 5. Detailed Findings

### H-1 — Unilateral owner custody of working capital, venue allowlist with infinite approvals, and all economics

**Severity: High (privileged role) · Status: Acknowledged (design)**
**Files:** `src/hook/RebateHook.sol:530-547`, `src/registry/PoolRegistry.sol:162-238, 307-338`

The protocol's default deployment binds the same administrative key to every
privileged role: `PoolRegistry` owner, `PayoutLedger` owner, `ILFund` owner,
`RebateHook` owner, and `OrderBook` owner. That key, acting alone and instantly
(no timelock), can:

1. **Drain the hook's working capital.** `setWorkingCapitalFloor(token, 0)` followed
   by `withdrawExcessCapital(token, balance)` withdraws every token the hook holds —
   including quote capital that LP-side rewards and pending rebates depend on, and
   any pool credits already released to the hook via `releaseToHook` but not yet
   donated.
2. **Register any contract as an external venue** (`registerVenue` checks only that
   the address has code and self-reports the right token pair), then
   `RebateHook.approveVenue` grants it `type(uint256).max` allowance on **both** pair
   tokens. A malicious or buggy venue can pull the hook's entire balances at any
   time — no per-venue cap, no allowance decay, no pull-only settlement.
3. **Rewrite all economics per pair** — trader/LP/searcher/IL splits (including
   routing 100% of profit to the "trader" share), fee schedule, size caps, loss
   guard — and pause captures selectively while keeping swaps flowing.

OpenZeppelin `Ownable` is one-step: a mistyped `transferOwnership` irreversibly
bricks administration (there is no `renounceOwnership`-safe two-step pattern, and no
accept step).

**Impact.** No anonymous exploit path — but a single compromised or malicious admin
key means total loss of hook working capital and of any undistributed escrow, and
silent rerouting of all future value flow. Users of the system (traders earning
rebates, LPs accruing rewards) bear counterparty risk against one EOA/multisig.

**Recommendations.**
- Move all parameter-changing and fund-moving calls behind a timelock with an
  explicit event surface; run the owner role as a multisig.
- Adopt two-step ownership transfer (OpenZeppelin `Ownable2Step`) across the five
  ownable contracts.
- Replace infinite venue approvals with bounded, per-leg approvals set immediately
   before each external leg (the executor knows the exact `amountIn`), or with a
  pull-pattern venue interface.
- Separate the "economic parameter" role from the "capital custody" role so the
  registry owner cannot unilaterally approve venues that spend the hook's balance.
- Consider a public `workingCapitalFloor` that cannot be lowered below the sum of
  outstanding ledger/IL obligations without timelock.

---

### M-1 — Fee-on-transfer / rebasing quote tokens break ledger solvency

**Severity: Medium · Status: Open**
**Files:** `src/rewards/PayoutLedger.sol:119-124`, `src/registry/PoolRegistry.sol:163-168`

`PayoutLedger.onCapture` credits `traderShare + Σ poolShares` (the hook's measured
realized profit) and escrows via `safeTransferFrom(hook, ledger, total)`. For a
fee-on-transfer quote token the ledger receives **less than `total`** while booking
the full amount into `capturedTotal`, `rebatesOutstanding` and `poolOutstanding`.
The invariant `isSolvent` still reads `true` (it compares book values only), but the
physical balance is short: later `claim`s and `releaseToHook`s revert on
insufficient balance once the shortfall surfaces — trader rebates and LP rewards
become frozen. Rebasing tokens create the mirrored problem (upward rebases strand
value; downward break collateralization). The same applies to `ILFund.fund`.

The only token gate is `registerPair`'s check that both tokens have bytecode
(`code.length > 0`); nothing verifies ERC-20 standardness or transfer-fee behavior.

**Impact.** A pair registered with a deflationary or rebasing quote token eventually
freezes user rebates and LP distributions (availability loss), while reported
accounting remains nominally "solvent".

**Recommendation.** Validate quote/base tokens against an explicit allowlist at
registration (or at minimum a standardness probe transferring a dust amount and
measuring the delta), and document the supported-token policy. Keep the honest
fallback: revert registration for tokens whose round-trip delta ≠ 0.

---

### M-2 — `hookData` recipient lets any executing router capture the trader's rebate

**Severity: Medium (integration trust) · Status: Open**
**Files:** `src/hook/RebateHook.sol:194-195, 451-456`, `src/periphery/SwapRouter.sol:178-179`

`afterSwap` derives the rebate recipient as `_decodeRecipient(hookData, sender)`:
if the swap's `hookData` is ≥ 32 bytes, the first 20 bytes are used verbatim;
otherwise the recipient is the PoolManager caller (`sender`). The recipient of a
hook-contract swap is therefore whoever assembled the calldata. The reference
`SwapRouter` honestly forwards the end user (`abi.encode(user)`), but **any
aggregator, custom router, or contract integrating the pool can substitute itself**
and accrue the 40%/10% trader rebate invisibly — the end trader has no on-chain way
to designate themselves except through the executor's cooperation.

**Impact.** Value leak at the integration layer: rebate mechanics degrade into a
rebate to whoever routes the trade. Not exploitable by pure bystanders, but the
system's headline economics ("every swap pays the trader back") silently depend on
untrusted intermediaries choosing honesty.

**Recommendation.** Document recipient semantics prominently in the integrator
guide; consider an opt-in recipient registry (`hook.setRebateRecipient(user)` or
EIP-712 signed designation consulted by `_decodeRecipient`) so the EOA can bind the
rebate to themselves independently of the router.

---

### M-3 — Capture loop forwards `gasleft()` with no reserve — approved venues can OOG-bleed into the user's swap

**Severity: Medium · Status: Open**
**Files:** `src/hook/RebateHook.sol:205` (`try this.runCapture{gas: gasleft()}`), `:428-449` (venue lens calls), `src/libraries/SwapExecutor.sol:121-134` (external leg)

The capture is revert-isolated (`try/catch`), which correctly converts *reverting*
captures into `CaptureSkipped`. However, the inner call is given **all** remaining
gas. Work inside the capture includes unbounded external calls to owner-registered
venues (`slot0()`, `liquidity()`, `token0/1()`, `swap()`). A buggy or malicious
venue can consume gas until the 63/64th-forwarded remainder is exhausted; the
`catch` clause and the outer `afterSwap`/swap settlement then themselves run out of
gas, reverting the **user's** transaction. The user pays for a failed swap, and the
venue's misbehavior is indistinguishable from organic OOG.

**Impact.** Griefing/DoS of swaps in hook pools by any owner-registered venue
(combines with H-1's weak venue vetting); availability risk for the trader
experience. Bounded by the fact that venues are admin-approved.

**Recommendation.** Forward a capped budget, e.g.
`try this.runCapture{gas: gasleft() - 60_000}` (or a configurable per-capture gas
ceiling), guaranteeing the outer frame always settles; optionally bound each venue
lens call with a low gas stipend.

---

### L-1 — `withdrawExcessCapital` underflows below the floor

**Severity: Low · Status: Open**
**File:** `src/hook/RebateHook.sol:530-532`

`if (amount > balance - workingCapitalFloor[token])` — when `balance < floor` the
subtraction underflows and the call reverts with `Panic(0x11)` (arithmetic
overflow) rather than the intended `ExceedsWorkingCapitalFloor`. Behavior is
safe (funds cannot leave) but the failure mode is opaque to integrators and
monitoring.

**Recommendation.** Compute `if (balance < workingCapitalFloor[token] || amount >
balance - workingCapitalFloor[token]) revert ExceedsWorkingCapitalFloor();`.

### L-2 — External *cheap* leg executes with `minOut = 0`

**Severity: Low · Status: Open**
**File:** `src/libraries/SwapExecutor.sol:51, 57-63`

The 70%-of-cost protective floor is applied only when the **rich** leg is external.
When the cheap (buy) leg is an external venue, it runs with `minOut = 0`: a venue
whose state moved since the snapshot (or which misprices) can hand back dust for
the full input. The realized-profit measurement then correctly reverts or skips the
payout, and the loss is bounded by `maxTradeSizeQuote` and the liquidity-proportional
cap — but the cycle still burned its input for nothing when it could have
self-aborted earlier.

**Recommendation.** Derive a conservative `minOut` for the cheap external leg from
the plan estimate (e.g. 70% of the closed-form `baseOutForQuote`), symmetric to the
rich-leg protection.

### L-3 — BatchAuctionPool: pre-settlement manipulation, post-deadline injection, owner drain mid-batch

**Severity: Low (explicitly experimental) · Status: Acknowledged (experimental)**
**File:** `src/periphery/BatchAuctionPool.sol:126-156, 164-181, 197-340`

Three interacting behaviors, acceptable for a demo venue but worth recording:

1. **Pre-settlement reserve manipulation.** Mode-1 `swap()` is always live; an
   attacker can trade the pool just before calling `settle()`, skewing the reserves
   from which the uniform clearing price is derived, then trade back after
   settlement. Batch participants' only protection is their own `minOut`.
2. **Post-deadline order injection.** `submitOrder` does not require
   `block.timestamp < batchDeadline`; between deadline and settlement, new orders
   still join the closing batch and shift the net residual — and therefore the
   clearing price — after participants believed the batch closed.
3. **Owner drain mid-batch.** `drain()` can empty the reserves while a batch is
   open; `settle()` then likely trips `BatchInsolvent` and reverts, leaving
   escrowed orders stuck until the owner re-seeds.

**Recommendation (if the venue is ever promoted).** Snapshot reserves at batch open
(or at deadline) for the clearing-price computation; reject orders once the deadline
has passed; forbid `drain()` while a batch is open. Until then, keep the venue
unregistered (as deployed) and the "experimental" label prominent.

### L-4 — Misleading error names and an incorrect NatSpec mask constant

**Severity: Low · Status: Open**
**Files:** `src/periphery/OrderBook.sol:112-116`, `src/periphery/BatchAuctionPool.sol:187`, `src/periphery/Create2Factory.sol:5-6`

- `OrderBook.setFillRewardBps` reverts with `ZeroAmount()` when `bps > MAX_FILL_REWARD_BPS` — wrong error for a bound violation.
- `BatchAuctionPool.cancel` reverts with `BatchStillOpen()` when the batch is *closed* (settlement window) — the name is inverted relative to the condition.
- `Create2Factory`'s NatSpec states the mined mask is `0x1040`; the actual required mask (AFTER_INITIALIZE | BEFORE_SWAP | AFTER_SWAP, as enforced by `DeploySystem` and tested by `test_H1_hookAddressBits`) is `0x10C0`.

**Recommendation.** Fix the two error names and the docstring constant; misleading
diagnostics cost real debugging time during incidents.

### I-1 — Native-currency pairs are structurally excluded

`registerPair` requires both tokens to have code, so the native asset can never be
a hook pair's base or quote; execution paths are written ERC-20-only accordingly.
This is coherent and safe, but it is an implicit asset policy — document it as such
(the `SwapRouter` itself does support native input for non-hook pools).

### I-2 — Pools shared across pairs with different quotes will revert captures

`PayoutLedger.poolRewardCurrency` and `ILFund.rewardCurrency` are fixed per pool at
first accrual. An operator misconfiguration that funnels two different quote
currencies into the same pool id makes subsequent captures revert
(`CurrencyMismatch` → captured by the isolation boundary as a skip). Safe failure,
but the registry could enforce pair/pool disjointness at registration time.

### I-3 — Rounding policy notes (positive)

Rounding consistently favors the escrow: IL-fund index arithmetic truncates toward
the fund; the LP split sends residual dust (`lpShare - Σ shares`) to paired pool 1;
capture sizing floors at `minProfitQuote`. No rounding-based value extraction was
identified.

---

## 6. Architecture and Trust Model

### Roles

| Role | Held by | Powers |
|---|---|---|
| Registry owner | deployer EOA/multisig | Register pairs/venues/protocol pools, set all pair parameters, all pause switches, bind hook |
| Hook owner | deployer | Withdraw working capital above (owner-set) floor, set floor, approve venues (infinite allowances) |
| Ledger / ILFund / OrderBook owners | deployer | Bind hook (once), set filler bounty |
| Searcher / keeper | anyone | `backrun` / `SearcherRouter.submit` — earns `searcherBps` on realized profit only |
| Filler | anyone | Execute crossed `OrderBook` orders — earns ≤ 10% bounty of output |
| Trader / LP | anyone | Swap, provide liquidity, `claim` rebates/compensation, `accrue`, trigger `distributePoolRewards` |

### Trust boundaries

- **Anonymous parties** can only ever *cause* executions whose payouts depend on
  *realized* profit; every permissionless entry point (backrun, fill, settle,
  distribute) was checked for reentrancy and value extraction and found bounded.
- **Registered venues** are fully trusted with the hook's tokens (see H-1, M-3):
  the interface assumes honest price/liquidity lenses and honest output delivery;
  over-reporting outputs self-corrects (leg 2 sells only what actually arrived or
  reverts atomically), but under-delivery is a realized loss bounded by the caps
  and the loss guard.
- **The owner** is the residual risk sink (H-1).
- **v4-core** is trusted as released (v4.0.0): pool math, fee accounting, unlock
  accounting, and hook permission encoding.

### Threat notes

- **Keeper competition** (backrun races) resolves off-chain in ordering; no
  on-chain auction — commissions are fixed-bps. No in-protocol MEV protection is
  claimed or needed for the *user's* swap (the capture is same-transaction by
  design).
- **Fill sandwiching** (`OrderBook`) is bounded: limit-style orders bound execution
  at the trigger; stop-style orders rely on the placer's `minAmountOut`; optimistic
  status marking (`orders[orderId].status = STATUS_FILLED` *before* `unlock`)
  closes reentrancy.
- **Estimate manipulation** of `RouteChecker` inputs only affects whether a capture
  is *attempted* — payment depends on realized post-trade balances, so inflated
  estimates cannot extract value by themselves.

---

## 7. Invariant Analysis

| Invariant | Enforcement | Verdict |
|---|---|---|
| Ledger solvency: `captured ≥ rebatesOutstanding + poolOutstanding` per token | Credits created only after `safeTransferFrom` escrow of the exact total (`onCapture`); claims/releases decrement symmetrically | Holds for standard ERC-20s; broken by FOT tokens (M-1) |
| ILFund solvency: `funded ≥ paid + owed` | Index arithmetic truncates toward the fund; `pendingRewards` roll-forward for pre-stake funding | Holds |
| Base neutrality per capture | All bought base is sold; post-cycle snapshot check within `baseDustToleranceBps` | Holds (tested H-series) |
| No payout without realized positive profit | Payout computed from post-cycle balance delta; `realized ≥ minProfitQuote` gate | Holds |
| Capture never reverts the user swap | `try/catch` isolation + transient guard | Holds unless OOG-bleed (M-3) |
| Single capture per swap | Transient-storage flag, auto-discarded on revert | Holds (tested `H7`) |
| Bounded work | `maxRouteChecks` (≤ 16 hard), `maxCyclesPerSwap` (≤ 4 hard), venues ≤ 4/pair, searcher batches ≤ 8 | Holds |
| Only registry-approved pools/venues/routes touched | Venue set assembled exclusively from pair config; `attachPool` hook-only; `OrderBook` checks `isApprovedPool` | Holds |
| Overflow-free estimation math | `MIN_SQRT/MAX_SQRT/MAX_LIQUIDITY` guards + `Math.mulDiv` throughout | Holds (fuzz `RC5`, guard test `RC7`) |
| Sum-of-splits conservation | `lpPool*` must sum to 10,000 bps; remainder dust to pool 1 | Holds (tested `H3b`) |

---

## 8. Test Coverage Assessment

Executed against the reviewed snapshot: **8 suites, 63 tests, 63 passing, 0
skipped** (`forge test`), plus clean `forge build` (`via_ir`) and `forge fmt
--check`. Coverage is strong on the happy paths and the stated guarantees: split
arithmetic parity, auto-pairing, pauses, loss guard, dynamic fees, IL carve-out,
order lifecycle, venue integration, batch bounds, solvency overdraft rejection,
and one parameterized fuzz (`RC5` monotonicity).

**Gaps observed (recommendations, not defects):**

1. No **adversarial venue** tests (malicious `IExternalVenue` lying on lens values,
   delivering short, or burning gas — directly exercises M-3/H-1 assumptions).
2. No **fee-on-transfer / rebasing token** fixtures (would have surfaced M-1).
3. No **stateful invariant fuzzing** (Echidna/Medusa/Foundry invariant mode) over
   ledger/IL-fund solvency across random operation sequences.
4. `BatchAuctionPool` settlement is tested on uniform behavior, not on the
   manipulation vectors in L-3.
5. No fork tests against the actual deployed v4 stack on the target chain.

---

## 9. Code Quality and Gas Observations

- Uniform, high-quality NatSpec; consistent error/event hygiene; clean sectioning —
  materially above typical testnet-code standard.
- Compiler-enforced safety: Solidity 0.8.26 checked arithmetic everywhere except
  deliberate `unchecked` in the WETH9-style wrapper (correctly scoped).
- `forge fmt --check` clean; lint advisories only (e.g. internal-function-used-once).
- Gas: `via_ir` required for size; capture loop is O(venues²) worst case with
  venues ≤ 6 — acceptable in `afterSwap` context given the caps; transient-storage
  guard is the cheap reentrancy primitive (Cancun).
- Minor: `OrderBook._payout` pushes native via low-level call (dead path for
  registry-approved pools — native pairs impossible per I-1, but the branch would
  let a reverting placer brick their own fill); `SearcherRouter` emits
  `KeeperBackrun` (via `backrun`) even when the capture later skips — indexers
  should treat it as intent, not outcome.

---

## 10. Conclusion

The Roobak contracts implement a coherent and unusually disciplined value-flow
design: nothing is paid that was not first measured as realized profit and escrowed;
every autonomous action is bounded; failure is contained. The review found **no
Critical issues and no exploitable-by-anonymous-attacker fund-loss paths**. The
risk profile is dominated by (a) the deliberate concentration of administrative
power over working capital and the venue allowlist (H-1), and (b) integration-layer
assumptions — token standardness (M-1), router honesty (M-2), and venue gas behavior
(M-3) — that hold in the reference deployment but deserve explicit enforcement or
documentation before the system is relied upon more broadly.

**Recommended remediation priority:** M-3 and L-2 (small, local fixes) → M-1 and
M-2 (policy + enforcement) → H-1 (operational hardening: timelock, multisig,
two-step ownership, bounded allowances) → keep `BatchAuctionPool` unregistered and
labeled experimental (L-3). After remediation, commission an independent external
audit before any mainnet deployment carrying real value.

---

## Appendix A — Reviewed Artifact Identification

- Release tree: `roobak-contracts/` (39 files: 15 `src` contracts, 9 `test` files,
  5 `script` files, CI workflow, configuration, README, LICENSE).
- Archive SHA-256:
  `ce965b91d56506b683289ac83972fd288fa0d1812ed13dc42cca014ccc2468f0`
- Toolchain: Foundry, `solc 0.8.26`, `evm_version cancun`, optimizer 10000 runs,
  `via_ir` (default profile; `test` profile without `via_ir`).
- Dependency pins (`foundry.lock`): `v4-core` v4.0.0 (`e50237c`), `v4-periphery`
  (`c6e4754`), `openzeppelin-contracts` v5.7.0 (`cab1993`), `forge-std` v1.17.0
  (`f3dae6e`).

## Appendix B — Severity Distribution Chart

```
Critical │  (0)
High     │ █        (1)
Medium   │ █ █ █    (3)
Low      │ █ █ █    (3)
Info     │ █ █ █    (3)
```

*End of report.*
