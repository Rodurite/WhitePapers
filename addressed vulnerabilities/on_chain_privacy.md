# Privacy: On-Chain Roblox Identifiers

## 1.1 Problem: Public “Ban List” for RMT Participants
In the initial design, each escrow contract (Escrow2P) stored Roblox-specific identifiers directly on-chain, for example:
• `sellerRobloxUserId`
• `userAssetId`
• `assetId`

Because Ethereum is a public ledger, any actor—including Roblox—can:
a) Enumerate all Escrow2P instances (e.g., by bytecode and factory address).
b) Read the associated Roblox account IDs and item IDs from contract storage.
c) Construct a precise, permanent list of accounts engaged in off-platform real-money trading (RMT).

This does not break the protocol cryptographically, but it creates a strong, unnecessary linkage between on-chain addresses and specific Roblox accounts, increasing the risk of targeted enforcement or bans against Rodurite users.

## 1.2 Design Goals
• **G1**: Remove direct Roblox identifiers from the blockchain.
• **G2**: Preserve a robust, immutable link between escrows and off-chain listings.
• **G3**: Keep the oracle capable of performing accurate cross-platform verification.

## 1.3 Solution: Opaque Listing Identifiers
We replace on-chain Roblox identifiers with a single opaque listing identifier, denoted `listingHash`, and move all Roblox-specific data off-chain.

### 1.3.1 Opaque Listing Metadata
The escrow contract `Escrow2P` now stores:
```solidity
bytes32 public immutable listingHash;
```
No Roblox IDs (`sellerRobloxUserId`, `userAssetId`, `assetId`) are stored on-chain. The value `listingHash` is computed off-chain by Rodurite’s backend when the seller creates a listing. A typical construction is:

```solidity
listingHash = keccak256(
    abi.encode(
        sellerRobloxUserId,
        userAssetId,
        assetId,
        randomSalt
    )
);
```

This `listingHash`:
• Uniquely identifies the listing and the underlying Roblox item.
• Does not reveal the underlying Roblox identifiers without the original data and salt.
• Is stored in both:
    – The Rodurite backend database.
    – The escrow contract at deployment time.

### 1.3.2 Oracle Resolution
The Rodurite Oracle, upon seeing a funded escrow, proceeds as follows:
a) Reads `listingHash` from the escrow contract.
b) Looks up the corresponding listing record in Rodurite’s database.
c) Retrieves from the database:
    • Seller Roblox ID.
    • Buyer Roblox ID (inferred from off-chain context).
    • `userAssetId` and `assetId`.
d) Uses the Roblox API to perform the normal inventory checks:
    • Asset is present in the buyer’s inventory.
    • Asset is absent from the seller’s inventory.

Only the backend and oracle ever see raw Roblox identifiers; the blockchain remains agnostic, holding only opaque references.

## 1.4 Result
• Roblox or any third party cannot directly map on-chain addresses to Roblox accounts via contract storage alone.
• The escrow contract retains a deterministic, immutable link to its listing via `listingHash`.
• Rodurite maintains full off-chain verifiability while significantly improving user privacy.
