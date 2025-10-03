# liSTX Rebasing Token Architecture & DEX Integration

## Overview

liSTX (Liquid Stacked STX) by LISA Lab implements an elegant shares-based rebasing mechanism inspired by Lido's stETH. This document explains how the rebasing works internally and how to integrate liSTX into AMM pools on Charisma and Faktory.

---

## How liSTX Rebasing Works

### Core Concept: Shares-Based Architecture

Unlike traditional rebasing tokens that mint/distribute new tokens to users, liSTX uses a **shares model** where:

- Users hold a fixed number of **shares** (the internal fungible token)
- The **reserve** variable tracks the total STX backing all shares
- User **balances are calculated dynamically** based on their share proportion

### The Mathematics

**Key Formula:**

```clarity
user_balance = (user_shares × total_reserve) / total_shares
```

**Example:**

```
Initial State:
- Total Reserve: 1,000,000 STX
- Total Shares: 1,000,000
- Alice's Shares: 10,000

Alice's Balance: (10,000 × 1,000,000) / 1,000,000 = 10,000 liSTX

After 10% Stacking Yield:
- Total Reserve: 1,100,000 STX (updated via rebase())
- Total Shares: 1,000,000 (unchanged!)
- Alice's Shares: 10,000 (unchanged!)

Alice's Balance: (10,000 × 1,100,000) / 1,000,000 = 11,000 liSTX ✨
```

### Key Functions

#### `rebase()` - Updates the Reserve

```clarity
(define-public (rebase)
    (let ((total-stx (get-total-stx)))
        (try! (contract-call? token-lqstx set-reserve total-stx))
        (ok total-stx)))
```

Called at the start of most operations (mint, burn, transfer) to ensure the reserve reflects current STX holdings.

#### `get-total-stx()` - Calculates Total Backing

```clarity
(define-read-only (get-total-stx)
    (let (
        (available-stx (stx-get-balance lqstx-vault))
        (deployed-stx (contract-call? .public-pools-strategy-v2 get-amount-in-strategy))
        (pending-stx (get-mint-requests-pending-amount)))
    (- (+ available-stx deployed-stx) pending-stx)))
```

Aggregates STX from:

- Vault balance (unstaked/liquid STX)
- Deployed in stacking strategies
- Minus pending mint requests

#### Token-to-Shares Conversion

```clarity
(define-read-only (get-tokens-to-shares (amount uint))
    (if (is-eq (get-reserve) (ok u0))
        amount
        (/ (* amount (unwrap-panic (get-total-shares))) (unwrap-panic (get-reserve)))))
```

**Formula:** `shares = (amount × total_shares) / reserve`

#### Shares-to-Token Conversion

```clarity
(define-read-only (get-shares-to-tokens (shares uint))
    (if (is-eq (get-total-shares) (ok u0))
        shares
        (/ (* shares (unwrap-panic (get-reserve))) (unwrap-panic (get-total-shares)))))
```

**Formula:** `tokens = (shares × reserve) / total_shares`

---

## Transfer Mechanics

### How Transfers Work

When transferring liSTX tokens:

1. User specifies amount in **tokens** (e.g., 10,000 liSTX)
2. System converts to **shares** using current reserve ratio
3. **Shares** are transferred between accounts
4. Both sender and receiver see correct **token balances** based on their shares

### Transfer Example

```
State:
- Reserve: 1,100,000 STX
- Total Shares: 1,000,000

Alice transfers 10,000 liSTX to Bob:

Step 1: Convert tokens to shares
shares = (10,000 × 1,000,000) / 1,100,000 = 9,091 shares

Step 2: Transfer shares
Alice: loses 9,091 shares
Bob: receives 9,091 shares

Step 3: Calculate new balances
Bob's balance = (9,091 × 1,100,000) / 1,000,000 = 10,000 liSTX ✅
```

**Important:** The token amount transferred is always accurate. The shares transferred represent the exact proportion of the reserve needed to transfer that token amount.

---

## Why This Design is Brilliant

### ✅ No Gas-Intensive Distributions

- No `send-many` operations needed
- No individual token minting per user
- Rebasing happens mathematically via the formula

### ✅ Automatic Yield Distribution

- When reserve increases (yield arrives), all balances update instantly
- Proportional rewards based on share ownership
- No user action required

### ✅ Maintains 1:1 STX Peg

- Reserve always tracks actual STX backing
- Token amounts calculated from real STX holdings
- Accurate pricing for DEX integrations

### ✅ SIP-010 Compatible

- Implements standard fungible token interface
- `transfer()` converts amounts to shares automatically
- `get-balance()` returns token-equivalent of shares
- Works with existing DeFi protocols

---

## DEX Integration: Listing liSTX on Faktory & Charisma

### Current State: Uniswap V2 Style AMM

liSTX can be listed on Faktory and Charisma DEXs using the standard **Dexterity trait** AMM pool implementation, which follows the Uniswap V2 constant product formula (x × y = k).

### Pool Structure: STX-liSTX

```clarity
;; Example pool structure (similar to sBTC-B pool)
(define-fungible-token STX-liSTX-LP)

Pool Tokens:
- Token A: STX (native Stacks token)
- Token B: liSTX (SM26NBC8SFHNW4P1Y4DFH27974P56WN86C92HPEHH.token-lqstx)
- LP Token: STX-liSTX-LP
```

### Integration Points

1. **Token Trait Implementation**

   - liSTX implements SIP-010 interface
   - `transfer()`, `get-balance()`, `get-total-supply()` all work as expected
   - Rebasing is transparent to the AMM

2. **AMM Operations**

   - **Swaps:** Standard Uniswap V2 swap logic with slippage protection
   - **Add Liquidity:** Users provide STX + liSTX in current pool ratio
   - **Remove Liquidity:** Users burn LP tokens to receive STX + liSTX
   - **LP Rebate:** 0.3% fee (LP_REBATE = 3000 / 1000000)

3. **Rebasing Considerations**
   - Pool balances update automatically as liSTX rebases
   - Price ratio adjusts naturally (STX reserve stays constant, liSTX increases)
   - LPs benefit from rebasing yield proportional to their pool share
   - No special handling needed - AMM sees liSTX as any other SIP-010 token

### Pool Implementation Example

Based on the attached sBTC-B pool, a STX-liSTX pool would look like:

```clarity
;; Token A: STX (via stx-transfer)
;; Token B: liSTX (via contract-call to token-lqstx)

(define-public (swap-a-to-b (amount uint) (min-y-out uint))
    (let (
        (sender tx-sender)
        (delta (get-swap-quote amount (some 0x00)))
        (dy-d (get dy delta)))
        (asserts! (>= dy-d min-y-out) ERR_TOO_MUCH_SLIPPAGE)
        ;; Transfer STX to pool
        (try! (stx-transfer? amount sender CONTRACT))
        ;; Transfer liSTX to sender
        (try! (as-contract (contract-call?
            'SM26NBC8SFHNW4P1Y4DFH27974P56WN86C92HPEHH.token-lqstx
            transfer dy-d CONTRACT sender none)))
        (ok delta)))

(define-public (swap-b-to-a (amount uint) (min-y-out uint))
    (let (
        (sender tx-sender)
        (delta (get-swap-quote amount (some 0x01)))
        (dy-d (get dy delta)))
        (asserts! (>= dy-d min-y-out) ERR_TOO_MUCH_SLIPPAGE)
        ;; Transfer liSTX to pool
        (try! (contract-call?
            'SM26NBC8SFHNW4P1Y4DFH27974P56WN86C92HPEHH.token-lqstx
            transfer amount sender CONTRACT none))
        ;; Transfer STX to sender
        (try! (as-contract (stx-transfer? dy-d CONTRACT sender)))
        (ok delta)))
```

### Key Differences from Standard Pools

**None.** The beauty of liSTX's shares-based design is that it appears as a standard SIP-010 token to external contracts. The rebasing happens internally and transparently.

---

## Future: Uniswap V3 Upgrade Path

### Why Upgrade?

Uniswap V3 concentrated liquidity pools offer:

- **Reduced slippage** through concentrated liquidity positions
- **Capital efficiency** - LPs can focus liquidity at specific price ranges
- **Higher fees** for active LPs providing liquidity where it's needed

### Migration Strategy

When Uniswap V3 implementation becomes available:

1. **Deploy New V3 Pool**

   - Create STX-liSTX V3 pool with concentrated liquidity
   - Set appropriate fee tiers (0.05%, 0.3%, 1%)

2. **User Migration**

   - Users remove liquidity from V2 pool
   - Receive back STX + liSTX
   - Add liquidity to V3 pool with chosen price range

3. **Incentivize Migration**

   - Could offer temporary rewards for V3 LP providers
   - Gradually reduce V2 pool incentives
   - Eventually deprecate V2 pool once majority migrated

4. **Advantages for liSTX**
   - Better price discovery around 1:1 STX peg
   - Lower slippage for large swaps
   - More efficient for yield-optimizing LPs

---

## Technical Summary

### liSTX Design Principles

1. **Shares are the truth** - internal fungible token that never changes due to rebasing
2. **Reserve tracks reality** - updated continuously to reflect actual STX backing
3. **Balances are calculations** - dynamically computed from shares × (reserve / total_shares)
4. **Transfers move shares** - amount specified in tokens, shares calculated and transferred
5. **SIP-010 compliant** - works seamlessly with existing DeFi protocols

### AMM Integration

- **Compatibility:** Full SIP-010 compatibility means standard AMM integration
- **Rebasing transparency:** AMM doesn't need to know about rebasing mechanism
- **Price dynamics:** Pool naturally adjusts as liSTX balance grows from yield
- **LP benefits:** Liquidity providers benefit from both trading fees and rebasing yield

### Recommended Pool Parameters

```
Pool: STX-liSTX
AMM Type: Constant Product (Uniswap V2)
Fee: 0.3% (standard)
Initial Price: ~1:1 (assuming similar backing)
Slippage Protection: Required (min-dy-out parameter)
LP Rebate: 0.3% to liquidity providers
```

---

## Conclusion

liSTX's elegant shares-based rebasing design makes it trivial to integrate into existing AMM infrastructure. The token behaves exactly like any other SIP-010 fungible token from the DEX's perspective, while internally managing the complex rebasing logic through mathematical conversions rather than token distributions.

For Faktory and Charisma, listing STX-liSTX pairs requires no special considerations beyond standard pool deployment. The rebasing happens transparently, and liquidity providers benefit from both trading fees and the underlying stacking yield that accrues to their liSTX holdings.

Future upgrades to Uniswap V3 style concentrated liquidity will provide even better capital efficiency and reduced slippage, with a straightforward migration path for existing liquidity providers.
