# Integrity: Escrow Impersonation via Clone Contracts

## 2.1 Problem: Deceptive Clone Escrows
Because `Escrow2P` is a public contract, any third party can:
• Deploy their own instance with identical parameters (same `expectedAmount`, same `listingHash`, same `mediator`).
• Present this contract to users as “the official Rodurite escrow” for a given listing (e.g., via DMs, phishing pages, or third-party UIs).

Consequences include:
• Buyers may fund orphan escrow contracts that Rodurite does not recognize.
• Naive oracles that simply watch “all contracts with this bytecode and mediator” can be confused by multiple escrows claiming to represent the same listing.
• Even when funds eventually refund, users suffer lockups and confusing UX.

The underlying issue is a lack of provenance: the chain cannot distinguish a Rodurite-managed escrow instance from a random clone with identical configuration.

## 2.2 Design Goals
• **G4**: Make it possible to cryptographically identify escrows created by Rodurite’s canonical factory.
• **G5**: Restrict Rodurite’s oracle to only process escrows that are both factory-originated and backend-registered.
• **G6**: Prevent users from interacting with untrusted clone escrows via the official Rodurite UI.

## 2.3 Solution: Factory Binding and Database Registration
We add two layers of provenance:
a) On-chain binding to the canonical factory.
b) Off-chain registration in the Rodurite database.

### 2.3.1 Factory Binding in Escrow2P
The `Escrow2P` contract records the deploying factory upon construction:
```solidity
address public immutable factory;

constructor(
    address _mediator,
    address _seller,
    uint256 _expectedAmount,
    bytes32 _listingHash
) {
    require(_mediator != address(0), "bad_mediator");
    require(_seller != address(0), "bad_seller");
    require(_expectedAmount > 0, "bad_amount");

    mediator = _mediator;
    seller = _seller;
    expectedAmount = _expectedAmount;
    listingHash = _listingHash;
    factory = msg.sender; // must be EscrowFactory
    status = Status.Unfunded;
}
```
The canonical Rodurite factory address is known and fixed. Any escrow whose `factory` field does not equal this address is rejected by:
• The Oracle.
• The frontend.
• Backend services.

### 2.3.2 Backend-Registered Escrows
Escrow lifecycle is now:
1. Seller creates a listing via Rodurite’s website.
2. The backend:
    a) Validates seller ownership of the Roblox asset (via the Roblox API).
    b) Creates a listing record in the database.
    c) Calls the canonical `EscrowFactory` to deploy an escrow with `listingHash`.
    d) Stores the returned `escrowAddress` in the listing record.
3. The frontend only displays escrow addresses retrieved from the backend.

The Oracle, when processing events, requires:
• `escrow.factory == ESCROW_FACTORY_ADDRESS`, and
• `escrowAddress` exists in the backend’s listing database.

Escrows deployed manually by third parties—even with identical configuration—are not present in Rodurite’s database and are therefore ignored.

## 2.4 Result
• Clone contracts no longer have any effect on Rodurite’s protocol logic.
• Oracles and UIs only interact with factory-originated, backend-registered escrows.
• Users cannot be tricked into funding rogue escrows via the official interface.
