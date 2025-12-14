# Availability: Griefing via Mass Funding and Refunds

## 3.1 Problem: Buyer-Funded Denial of Service
In the baseline model, any user can:
1. Call `fund()` on any escrow with the correct ETH amount.
2. Never perform the Roblox trade.
3. Wait for the seller arming window (48h) or protection window (120h) to expire and be refunded.

An attacker can:
- Script funding of every active listing on Rodurite.
- Cause:
    – Sellers’ items to appear “taken” but never settle.
    – A large volume of refunds, increasing oracle gas costs and API usage.
    – Significant disruption to marketplace liquidity and user experience.

The attacker only pays gas and the opportunity cost of temporarily locked ETH, which may be acceptable for a determined adversary.

## 3.2 Design Goals
- **G7**: Ensure only vetted buyers can fund escrows.
- **G8**: Make large-scale griefing significantly more expensive or infeasible.
- **G9**: Keep the user experience simple for legitimate buyers.

## 3.3 Solution: Off-Chain Authorized Funding via ECDSA Tickets
We introduce a ticket-based authorization layer for funding escrows. Funding is no longer permissionless in the raw contract; it requires a signed approval from a Rodurite backend key.

### 3.3.1 The Authorization Signer
Rodurite maintains a dedicated authorization signer keypair:
- Private key: held securely by the backend.
- Public key / address: stored on-chain as, for example, `authSigner`.

This key is used strictly to authorize specific actions (e.g., “buyer X may fund escrow Y”).

### 3.3.2 Buyer “Fund Tickets”
To fund an escrow, a buyer must first:
1. Authenticate with Rodurite’s website (e.g., via Roblox login and wallet connection).
2. Pass internal checks:
    - Account age and verification.
    - Velocity limits and rate limiting.
    - Reputation or risk scoring.

If approved, the backend generates a fund ticket:
```solidity
message = keccak256(
    abi.encode(
        "RODURITE_FUND",
        escrowAddress,
        buyerAddress,
        listingHash,
        expectedAmount,
        nonce,
        deadline
    )
);

signature = Sign(message, authSignerPrivateKey);
```

The buyer’s frontend then calls:
```solidity
escrow.fund(nonce, deadline, signature) { value: expectedAmount }
```

### 3.3.3 On-Chain Verification in fund()
The `Escrow2P.fund` function is updated to perform the following checks:
1. Validate escrow state:
    - `status == Status.Unfunded`
    - `msg.value == expectedAmount`
2. Validate time and replay protection:
    - `block.timestamp <= deadline`
    - `nonce` has not been used before (tracked in a mapping).
3. Recompute the signed message from:
    - `address(this)` (escrow address).
    - `msg.sender` (buyer address).
    - `listingHash`.
    - `expectedAmount`.
    - `nonce`, `deadline`.
4. Use `ECDSA.recover` to ensure the signature is from `authSigner`.

Only if all checks pass does the contract:
- Set `buyer = msg.sender`.
- Move to `Status.Funded`.
- Emit the `Funded` event.

Without a valid fund ticket, `fund()` reverts.

### 3.3.4 Anti-DoS Effect
Since only the backend can issue valid fund tickets:
- Attackers can no longer mass-fund escrows directly at the contract layer.
- Any attempt to grief the marketplace must pass through backend defenses:
    – Rate limits per user, IP, device, or account.
    – Reputation thresholds.
    – Manual review for large or suspicious behaviors.

Rodurite can also:
- Limit the number of concurrent open fund tickets per user.
- Impose off-chain penalties or suspensions for abusive patterns.

The attack surface thus shifts from “any Ethereum address” to “only authenticated, rate-limited Rodurite users”, dramatically increasing the cost and complexity of griefing.

## 3.4 Optional Extension: Gating Factory Deployment
A symmetric technique can be applied at the factory level:
- Require a similar signed authorization ticket from `authSigner` to call `EscrowFactory.deployEscrow`.
- Ensure only backend-approved listing creations can result in on-chain escrows.

This further enforces the invariant that every escrow is tied to a known, validated listing and a known seller record in Rodurite’s database.
