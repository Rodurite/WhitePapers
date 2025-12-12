# Pending Roblox Trades & Buyer Inaction: Risk Model and Settlement Policy

## 1. Background and Risk Description
Roblox asset trades are non-atomic and asynchronous. Upon submission of a trade offer:
- The asset remains in the seller’s inventory until acceptance.
- The buyer may accept the trade at any time.

Roblox does not provide reliable, documented, or enforceable guarantees regarding:
- Outbound trade visibility.
- Trade cancellation status.
- Maximum acceptance or expiration time.

As a result, third-party escrow systems cannot deterministically verify whether a trade offer is pending, cancelled, or expired at any given time.

## 2. Identified Abuse Vector
If an escrow protocol refunds buyer funds before all possible acceptance windows have elapsed, a malicious buyer may exploit the following sequence:
1. Seller submits a Roblox trade offer.
2. Buyer intentionally delays acceptance.
3. Escrow refund conditions are met.
4. Buyer receives a refund.
5. Buyer subsequently accepts the Roblox trade.
6. Buyer receives both the asset and the refunded funds.

This risk arises from platform-level constraints and cannot be mitigated through additional polling or API usage.

## 3. Design Objectives
Rodurite’s settlement policy is designed to:
- Prevent economic exploitation caused by delayed trade acceptance.
- Avoid reliance on undocumented Roblox trade mechanics.
- Eliminate the need for financial collateral, bonding, or partial refunds.
- Ensure that refunds are executed only when it is reasonably safe to do so.
- Explicitly define responsibility boundaries between participants.

## 4. Seller Arming Requirement
Rodurite introduces a mandatory Seller Arming Confirmation (“Arming”).
Arming is an on-chain declaration by the seller that:
**A Roblox trade offer for the specified asset has been submitted to the buyer.**

Until the seller arms the escrow:
- Buyer identity information is not disclosed.
- Extended settlement timers do not commence.
- Buyer funds remain fully protected.

## 5. Settlement Timeline and Conditions

### 5.1 Buyer Funding
The buyer deposits the agreed ETH amount into escrow. The seller is notified of the pending transaction.

### 5.2 Seller Response Window (48 Hours)
The seller is provided up to forty-eight (48) hours to arm the escrow.
If the seller fails to arm within this window:
- The escrow is cancelled.
- The buyer receives a full refund.
- No further obligations are imposed on either party.

### 5.3 Buyer Acceptance Window (48 Hours from Arming)
Once armed:
- The buyer has forty-eight (48) hours to accept the Roblox trade.
- The system monitors asset ownership continuously.
- If ownership transfer is verified, escrow funds are released immediately to the seller.

### 5.4 Trade Expiry Protection Window (Up to 120 Hours from Arming)
If delivery is not detected within the initial acceptance window:
- Rodurite assumes a trade may remain pending.
- **No refund is executed.**
- The escrow enters a protection period lasting up to one hundred twenty (120) hours from the time of arming.

During this period:
- Asset ownership continues to be monitored.
- The seller receives automated notifications regarding buyer inactivity, upcoming refund execution, and cancellation responsibilities.

### 5.5 Seller Cancellation Responsibility
Prior to refund execution:
- The seller is responsible for cancelling any outstanding Roblox trade offers.
- This ensures the buyer cannot accept the trade after escrow resolution.
- Rodurite does not verify trade cancellation status and cannot enforce cancellation on behalf of the seller.

### 5.6 Refund Execution
At the conclusion of the protection window:
- If asset delivery has not occurred:
    - Buyer funds are refunded in full.
    - The escrow is permanently closed.
- Refunds are executed only after the maximum protection period has elapsed.

## 6. Seller Acknowledgment and Consent
To arm an escrow, the seller must explicitly agree to the following acknowledgment:

> **Seller Acknowledgment**
> I understand that after arming this escrow, I am responsible for cancelling any outstanding Roblox trade before the refund executes. Failure to do so may allow the buyer to accept the trade at a later time, resulting in the transfer of the asset without payment.

This acknowledgment constitutes informed consent to the settlement risk model.

## 7. Risk Allocation Summary

| Scenario | Outcome |
| :--- | :--- |
| Seller fails to respond | Buyer refunded |
| Seller arms and buyer accepts | Seller paid |
| Buyer delays acceptance | Refund delayed safely |
| Seller fails to cancel trade | Risk borne by seller |
| Late trade acceptance | Refund not executed prematurely |

## 8. Rationale and Compliance Considerations
This policy:
- Reflects conservative assumptions about platform behavior.
- Avoids representations of guaranteed trade expiry.
- Clearly discloses non-custodial limitations.
- Places responsibility on the party best positioned to mitigate risk.
- Eliminates economic incentives for malicious delay.

## 9. Conclusion
Due to inherent limitations in Roblox’s trade system, escrow settlement must account for prolonged acceptance risk.
Rodurite’s Seller Arming and Deferred Refund Policy ensures:
- Refunds are not executed while asset transfer remains possible.
- Buyers cannot profit from intentional inaction.
- Sellers retain control over trade cancellation.
- All risks are explicitly disclosed and consented to.

This framework enables secure, non-custodial escrow settlement without reliance on collateral, trust-based enforcement, or undocumented platform guarantees.
