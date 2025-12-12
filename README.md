# Rodurite Whitepaper
## Decentralized Escrow Protocol for Roblox Assets

**Version:** 1.0  
**Date:** December 2025

---

## 1. Abstract

Rodurite is a decentralized, trust-minimized escrow protocol designed to bridge the gap between the Roblox virtual economy and the Ethereum blockchain. By leveraging smart contracts and an automated off-chain oracle system, Rodurite enables secure, peer-to-peer trading of Roblox "Limited" assets for Ethereum (ETH), eliminating the risk of scams prevalent in traditional "trust-based" trading.

## 2. Introduction

The economy of user-generated content platforms like Roblox has grown exponentially, with rare digital items ("Limiteds") commanding significant real-world value. However, the infrastructure for trading these assets for liquid crypto-currencies is severely lacking. Users are often forced to rely on black-market forums, middleman services with high fees, or risky direct transfers that frequently result in fraud.

Rodurite solves this by introducing a programmable escrow layer that atomizes the trade: funds are only released when the asset transfer is cryptographically verified on the Roblox platform.

## 3. Problem Statement

### 3.1 The Trust Deficit
In a typical Real-Money Trade (RMT) for a Roblox item:
1.  **Buyer Risk**: The buyer sends payment first, and the seller blocks them without delivering the item.
2.  **Seller Risk**: The seller delivers the item, but the buyer reverses the payment (chargeback fraud).

### 3.2 Inefficiency
Existing centralized marketplaces charge exorbitant fees (often 10-30%) to cover their operational overhead and fraud insurance. They also require custodial deposits, meaning users must trust the platform with their assets.


## 4. The Rodurite Solution

Rodurite introduces a **Hybrid Escrow Model** with a mandatory **Seller Arming** phase:
1.  **Fund**: Buyer deposits ETH into the `Escrow2P` contract.
2.  **Arm**: Seller explicitly "arms" the escrow on-chain, declaring they have sent the trade offer. This starts the protection timer.
3.  **Verify**: The Rodurite Oracle monitors the Roblox platform for asset transfer.
4.  **Settle**:
    *   **Success**: Asset transfer is verified -> Funds released to Seller.
    *   **Safety**: Trade offer expires or is cancelled (120h max) -> Funds refunded to Buyer.

## 5. Technical Architecture

### 5.1 Smart Contracts (Ethereum/EVM)
The core of the system is the `Escrow2P.sol` contract. Each trade deploys a unique instance of this contract to ensure isolation and security.

*   **State Machine**: The contract moves through strict states: `Unfunded` -> `Funded` -> `Armed` -> `Released` OR `Refunded`.
*   **Immutable Metadata**: Each contract is cryptographically bound to specific trade details:
    *   `listingHash`: An opaque hash unique to the listing (hiding raw Roblox IDs).
    *   `expectedAmount`: The agreed-upon ETH price.
*   **Mediator Role**: A restricted `mediator` address (controlled by the Rodurite Oracle) is the only entity authorized to trigger `release()` or `refund()`.

### 5.2 The Oracle System
The Rodurite Oracle bridges the two worlds.

*   **Monitoring**: It polls for `Funded` and `Armed` events.
*   **Seller Arming**: The seller has **48 hours** to arm the escrow after funding. Failure to arm results in an automatic refund.
*   **Verification**: Once armed, the Oracle monitors for the asset transfer.
    *   *Success Condition*: The asset matches the `listingHash` criteria in the Buyer's inventory.
*   **Settlement**:
    *   If verified: The Oracle calls `release()` immediately.
    *   If unverifed:
        *   **Standard Acceptance Window**: Buyer has **48 hours** to accept the trade.
        *   **Protection Window**: If the trade is still pending, the refund is deferred up to **120 hours** to prevent "accept-after-refund" exploits.
*   **Notifications**: Automated emails for Funding, Arming, and Completion.

### 5.3 Frontend Application
A modern React + Vite application provides the user interface.
*   **Wallet Connection**: Integrated with `wagmi` and `viem` for seamless MetaMask/WalletConnect support.
*   **Dashboard**: Users can track their active listings, view escrow status, and receive real-time updates on their trades.

## 6. Security & Safety Mechanisms

### 6.1 Fund Safety
Funds are never held by Rodurite's operational wallets. They reside in the `Escrow2P` contract. The `mediator` key can only direct funds to the pre-agreed Seller (on success) or back to the Buyer (on failure). It cannot redirect funds to arbitrary addresses.

### 6.2 Anti-Fraud
*   **Unique Asset Tracking**: By tracking the specific `listingHash`, Rodurite prevents "bait-and-switch" scams.
*   **Deferred Refunds**: The **120-hour protection window** protects sellers from "delayed acceptance" attacks where a buyer accepts a trade after receiving a refund.

### 6.3 Protocol Fees
The smart contract enforces a **7% protocol fee** (`FEE_BPS = 700`) on successful trades. This fee is automatically deducted from the payout to the seller and sent to the `FEE_RECIPIENT` address. Refunds are fee-free (100% of ETH is returned to the buyer).

## 7. Future Roadmap

*   **Multi-Chain Support**: Expansion to L2 networks (Base, Optimism, Arbitrum) to reduce gas costs.
*   **DAO Governance**: Decentralizing the `mediator` role and fee parameter adjustments.
*   **Generic Asset Support**: Extending the Oracle to support other game economies (e.g., CS2, TF2) that expose public inventory APIs.

## 8. Conclusion

Rodurite represents the next evolution in virtual asset trading. By replacing human middlemen with code and cryptographic verification, we create a safer, faster, and more efficient market for the Roblox economy.
