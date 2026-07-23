# Audit Findings: FeeCalculator.sol — Fee Calculation Logic

**Contract:** `FeeCalculator.sol` (sandbox test contract)
**Reviewed functions:** `calculateFeeUnchecked`, `calculateFeeChecked`, `setFeePercent`
**Method:** Manual review + adversarial input testing via Remix VM (Cancun), cross-verified with exact uint256 arithmetic simulation

---

## Bug 1 — High: Unchecked multiplication silently wraps to a plausible-looking incorrect value

**Location:** `calculateFeeUnchecked()`, inside the `unchecked { }` block:
```solidity
unchecked {
    fee = (amount * feePercent) / PERCENT_BASE;
    amountAfterFee = amount - fee;
}
```

**Description:**
Because the multiplication `amount * feePercent` is wrapped in an `unchecked` block, it does not benefit from Solidity 0.8+'s default overflow protection. When the true mathematical product exceeds `2^256 - 1`, the result silently wraps modulo `2^256` instead of reverting.

The danger isn't just that this produces a wrong number — it's *which* wrong number. Some overflow cases produce an obviously-broken result (`fee = 0` on a huge `amount`), which a sanity check might catch. Others produce a **mid-sized, entirely plausible-looking value**, which would pass a superficial "does this fee look reasonable" review.

**Proof of Concept:**

| amount | feePercent | Real (unwrapped) product | `calculateFeeUnchecked` result | `calculateFeeChecked` result |
|---|---|---|---|---|
| `2^128` | `2^128` | exactly `2^256` | `fee = 0` — obvious red flag | REVERTS (overflow) |
| `2^128` | `2^128 + 1` | `2^256 + 2^128` | `fee = 34028236692093846346337460743176821` — **looks like a real fee, isn't** | REVERTS (overflow) |
| `MAX (2^256-1)` | `2` | `~2·MAX` | `fee = 11579208923731619542357098500868790785326998466564056403945758400791312963` — large but wrong | REVERTS (overflow) |

The second row is the highest-value finding: the wrapped output (`~3.4 × 10^37`) is not zero, not `MAX`, and not obviously absurd — it's the kind of number that could sit in a transaction log or a monitoring dashboard without raising suspicion, while being mathematically disconnected from the actual `amount`/`feePercent` inputs.

**Impact:**
Any caller who can influence `amount` and/or `feePercent` toward these ranges can force a fee calculation to silently return an incorrect value instead of reverting. Depending on how `fee` is consumed downstream (transferred, minted, used to adjust balances), this could mean incorrect fee collection, incorrect crediting of funds, or corrupted accounting state — all without a single reverted transaction to flag the issue after the fact.

**Recommendation:**
Remove the `unchecked` block. Solidity 0.8+'s default checked arithmetic will revert on overflow at negligible extra gas cost, converting this from a silent corruption into a safe, visible failure. If `unchecked` was added purely for a gas optimization, confirm via testing (as above) that the input space is actually bounded enough (e.g., by capping `feePercent`, see Bug 2) to make overflow structurally unreachable before relying on `unchecked` again.

---

## Bug 2 — Medium: No upper bound on `feePercent` allows fee to exceed principal (caught by underflow, not by design)

**Location:** `setFeePercent()`:
```solidity
function setFeePercent(uint256 _feePercent) public {
    feePercent = _feePercent; // no upper bound check
}
```

**Description:**
`feePercent` can be set to any `uint256` value, including values representing well over 100% (e.g., `20000` = 200% against a `PERCENT_BASE` of `10000`). This allows `fee` to exceed `amount` itself.

**Notable distinction worth flagging in a real audit:** the *checked* version of the fee function (`calculateFeeChecked`) does not fail here due to any explicit validation catching an out-of-range `feePercent`. It fails because `amountAfterFee = amount - fee` underflows once `fee > amount`, and Solidity 0.8+ reverts on unsigned integer underflow by default. In other words, the protection is **incidental** — a side effect of subtraction underflow checking — not a deliberate design decision to bound fees.

**Proof of Concept:**

| amount | feePercent | `calculateFeeUnchecked` fee | `calculateFeeChecked` result |
|---|---|---|---|
| `100` | `20000` (200%) | `200` (2x principal) | REVERTS — **subtraction underflow**, not multiplication overflow |
| `1` | `100000` (1000%) | `10` | REVERTS — subtraction underflow |
| `1,000,000` | `1,000,000` | `100000000` | REVERTS — subtraction underflow |

Confirmed: in every tested case, the multiplication itself does **not** overflow (the numbers are too small for that) — it's specifically the later `amount - fee` step that reverts.

**Impact:**
While `calculateFeeChecked` happens to be protected today, this protection is fragile: it depends entirely on `amountAfterFee = amount - fee` being the very next line, in the very next operation, with no intervening logic. Any refactor that separates the fee calculation from the subtraction (e.g., calculating `fee` in one function and consuming it in another, or storing `fee` before using it elsewhere) would remove this incidental protection and allow a `fee > amount` state to propagate silently. Relying on an accidental side effect instead of an explicit invariant is itself a code-quality/maintainability risk, independent of whether it currently "works."

**Recommendation:**
Add an explicit cap in `setFeePercent`, independent of relying on underflow elsewhere:
```solidity
function setFeePercent(uint256 _feePercent) public {
    require(_feePercent <= PERCENT_BASE, "Fee too high");
    feePercent = _feePercent;
}
```
Verified fix: `setFeePercent(20000)` now reverts with `"Fee too high"` at the point of assignment, rather than deferring the failure to whatever function happens to subtract `fee` from `amount` later.

---

## Bug 3 — Low/Medium: Integer division silently rounds small fees to zero

**Location:** Both `calculateFeeUnchecked` and `calculateFeeChecked`, in the shared line:
```solidity
fee = (amount * feePercent) / PERCENT_BASE;
```

**Description:**
Solidity integer division truncates. Whenever `amount * feePercent < PERCENT_BASE`, `fee` truncates to `0` — with no revert, no event, no signal that anything unusual happened. This is not an overflow/underflow issue; it reproduces identically in both the checked and unchecked versions, since it's a pure logic/precision property of integer division, not a Solidity 0.8 safety feature.

**Proof of Concept:**

| amount | feePercent | fee | Notes |
|---|---|---|---|
| `1` | `1` | `0` | smallest possible nonzero inputs on both sides |
| `9999` | `1` | `0` | largest amount that still rounds to zero at this feePercent |
| `10000` | `1` | `1` | boundary — first amount where fee becomes nonzero |
| `1` | `9999` | `0` | fee still zero even at feePercent just below 100% |

Identical results were confirmed on both `calculateFeeUnchecked` and `calculateFeeChecked` — this is expected, since the bug is in the division logic itself, not in overflow handling.

**Impact:**
A caller can repeatedly transact with `amount` values below the rounding threshold (`amount < ceil(PERCENT_BASE / feePercent)`) and pay **zero fee indefinitely**, split across many small transactions instead of one large one. Depending on the protocol's fee model, this could represent a systematic way to avoid fees entirely at scale by structuring activity into many small operations rather than fewer large ones.

**Recommendation:**
Depending on intended behavior, consider one of:
- A minimum fee floor (e.g., `if (fee == 0 && amount > 0) fee = 1;`), if any nonzero-amount transaction is meant to always incur some fee.
- Aggregating small amounts before fee calculation, if the protocol's design allows batching.
- Explicitly documenting and accepting this as intended behavior, if dust-level fee avoidance is considered economically irrelevant at the protocol's expected minimum transaction size — but this should be a deliberate, stated decision rather than an unexamined side effect.

---

## Summary Table

| # | Severity | Bug | Root Cause | Status |
|---|---|---|---|---|
| 1 | High | Unchecked multiplication wraps to plausible-looking wrong value | `unchecked` block bypassing Solidity 0.8+ overflow protection | Fix verified (removing `unchecked` causes revert) |
| 2 | Medium | Fee can exceed principal | No upper bound on `feePercent`; incidentally caught by underflow, not by design | Fix verified (`require` cap added) |
| 3 | Low/Medium | Small amounts round fee down to zero | Integer division truncation | Fix proposed, not yet applied |

## Methodology Note
All numeric edge cases above were generated via an AI-assisted adversarial test matrix, then independently verified two ways: (1) executed live against a deployed instance in Remix VM (Cancun), and (2) cross-checked against an exact uint256 arithmetic simulation to confirm the Remix output matched expected EVM wrap/revert behavior. The underflow-vs-overflow distinction in Bug 2 was discovered during this cross-verification step, not anticipated in the original test matrix — underscoring the value of confirming *why* a revert happens, not just *that* it happens.
