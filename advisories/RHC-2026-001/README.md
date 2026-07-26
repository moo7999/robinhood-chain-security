# RHC-2026-001 — Unbounded offer array enables permanent denial-of-service on Rob Domains marketplace

| Field | Value |
|---|---|
| Advisory ID | RHC-2026-001 |
| Target | Rob Domains — DomainMarketplace |
| Chain | Robinhood Chain (chain ID 4663) |
| Contract | 0x97A9338083de9a01dA399B9A65768c79862dB9A4 |
| Severity | High (permanent denial-of-service, no funds lost) |
| Class | CWE-400 — Uncontrolled resource consumption / unbounded loop |
| Status | Fixed — remediated in v3 marketplace 0xA023960c... |
| Reported | 2026-07-25 (privately, to the team) |
| Fixed | 2026-07-25 (fix deployed and verified on-chain) |
| Disclosed | 2026-07-26 |
| Researcher | MO3TH |

## Summary

The DomainMarketplace contract stores every offer ever made on a domain in a per-token array, `offers[tokenId]`. A one-active-offer-per-buyer rule was added to bound this array, and it does bound the number of concurrently active offers. It does not bound the array's total length, because `withdrawOffer` clears the buyer's dedup slot.

An attacker can loop `makeOffer` then `withdrawOffer` indefinitely. Each cycle appends one permanent dead entry to `offers[tokenId]` and returns the attacker's ETH in full. The only cost is gas.

Two hot-path functions, `buyDomain` and `acceptOffer`, iterate the entire `offers[tokenId]` array, including inactive entries, via the internal `_queueRefundsForAllOffers` helpers. Once the array is large enough, both functions exceed the block gas limit and revert. The targeted domain becomes permanently unsellable and its accepted-offer path permanently frozen.

No funds are stolen. The impact is a permanent, irreversible denial-of-service on any chosen domain, mountable by anyone for the price of gas.

## Affected code

`DomainMarketplace.sol`

The dedup map is documented as the thing that bounds the array:

```solidity
// One active offer per buyer per token.
// Value is offerIndex + 1; 0 means no active offer.
mapping(uint256 => mapping(address => uint256)) private _activeOfferIndexPlusOne;
```

`makeOffer` only reuses an existing array slot when the dedup map still points at one; `withdrawOffer` zeroes that slot, so the next `makeOffer` from the same buyer falls into the else-branch and pushes again:

```solidity
function withdrawOffer(uint256 tokenId, uint256 offerIndex) external nonReentrant {
    Offer storage o = offers[tokenId][offerIndex];
    require(o.active, "Marketplace: offer not active");
    require(o.buyer == msg.sender, "Marketplace: not offer maker");

    uint256 amount = o.amount;

    o.active = false;
    _activeOfferIndexPlusOne[tokenId][msg.sender] = 0;

    (bool sent, ) = msg.sender.call{value: amount}("");
    require(sent, "Marketplace: refund failed");

    emit OfferWithdrawn(tokenId, offerIndex);
}
```

`_queueRefundsForAllOffers` and `_queueRefundsForAllOffersExcept` both iterate `os.length`, the full array including dead entries. `buyDomain` calls the first, `acceptOffer` calls the second.

## Attack

Preconditions: none beyond holding an EOA with gas. The attacker never needs to own or list the domain, and never loses principal.

**Step 1.** Pick any tokenId (existing domain).

**Step 2.** Call `makeOffer(tokenId, 0)` with a minimal msg.value.

**Step 3.** Call `withdrawOffer(tokenId, i)` for a full refund; array length stays grown by one.

**Step 4.** Repeat steps 2 and 3, N times. Each iteration adds one permanent dead Offer to `offers[tokenId]`.

**Step 5.** Once iterating the array exceeds the block gas limit, `buyDomain(tokenId)` and `acceptOffer(tokenId, _)` both revert for everyone, forever.

The domain can still be transferred directly via the NFT contract, but it can never again be sold through the marketplace, and the owner can never accept an offer on it.

### Aggravating factor: optimizer disabled

The vulnerable deployment was verified with `optimizer.enabled: false`. Every loop iteration therefore costs more gas than necessary, lowering the number of dead entries required to cross the block gas limit.

## Impact

**Severity:** High.

**Availability:** permanent. The marketplace sell path and offer-accept path for the targeted domain are frozen with no recovery function. There is no admin method to prune `offers[tokenId]`, so the state is irreversible.

**Scope:** any single domain per attack; repeatable across many domains.

**Cost to attacker:** gas only; principal fully recovered each cycle.

## Recommended fix

Stop treating the dedup slot as transient. Keep `_activeOfferIndexPlusOne` as a permanent buyer to array-index map, let the `Offer.active` flag carry liveness, and never clear the slot on withdraw, reject, accept, or queue. A returning buyer then always reuses their existing array element, capping `offers[tokenId].length` at the number of unique buyers, permanently.

The one subtlety: `makeOffer` must only credit a pending refund when the slot it is reusing is currently active, otherwise a buyer whose offer was already refunded gets credited a second time and can drain the contract.

```solidity
if (existingIndexPlusOne != 0) {
    offerIndex = existingIndexPlusOne - 1;
    Offer storage existing = offers[tokenId][offerIndex];

    if (existing.active) {
        pendingRefunds[msg.sender] += existing.amount;
        emit RefundPending(msg.sender, existing.amount);
    }

    existing.amount     = msg.value;
    existing.expiration = expiration;
    existing.active     = true;
}
```

### Defense in depth

**Add a MIN_OFFER floor** so spamming distinct addresses is not free either.

**Re-enable the optimizer** and pin an exact, well-exercised compiler version.

**Cap or paginate** any function that iterates offers.

## Related, lower-severity findings

Tracked separately in `../../findings/`:

**getOffersByMaker is unbounded** (Medium).

**DomainNFT.safeTransferFrom does not honor ERC-721 receiver checks** (High, unfixable).

**Single-EOA admin** (Medium, governance).

## Remediation

The team deployed a v3 marketplace at `0xA023960c23c7EFFC18c58b0049F14D4a85d2d85d` (fully verified) implementing the recommended fix exactly.

`withdrawOffer`, `rejectOffer`, and `acceptOffer` now clear only the `Offer.active` flag and preserve the buyer's permanent slot in `_activeOfferIndexPlusOne`. `makeOffer` reuses that slot in place rather than pushing a new element:

```solidity
if (existingIndexPlusOne != 0) {
    offerIndex = existingIndexPlusOne - 1;
    Offer storage existing = offers[tokenId][offerIndex];

    if (existing.active) {
        pendingRefunds[msg.sender] += existing.amount;
        emit RefundPending(msg.sender, existing.amount);
    }

    existing.amount     = msg.value;
    existing.expiration = expiration;
    existing.active     = true;
}
```

`offers[tokenId].length` is now permanently bounded by the number of unique buyers, so the makeOffer, withdraw, makeOffer cycle can no longer grow the array, and `buyDomain` / `acceptOffer` cannot be gas-locked. The double-refund guard flagged in the recommendation was included.

Also confirmed fixed in the same deployment: optimizer enabled (200 runs), reentrancy guard on all state-changing functions, pull-payment refunds via `pendingRefunds`, `rejectOffer` fallback to the refund queue, checks-effects-interactions ordering, and an O(1) circular buffer for recent sales.

Still open after v3: `getOffersByMaker` remains an unbounded view (no funds at risk); and DomainNFT (`0xF960519a...`) is unchanged and not upgradeable, so the safeTransferFrom receiver-check issue persists in that contract.

## Timeline

| Date | Event |
|---|---|
| 2026-07-24 | Initial report of first-generation marketplace issues (0xE62Eef82...) to the team |
| 2026-07-25 | Team shipped rewritten marketplace 0x97A9338083de9a01dA399B9A65768c79862dB9A4 (v2) |
| 2026-07-25 | RHC-2026-001 (this issue) identified in the v2 contract and reported privately |
| 2026-07-25 | Fix deployed as v3 marketplace 0xA023960c23c7EFFC18c58b0049F14D4a85d2d85d, verified on-chain |
| 2026-07-26 | Public disclosure |

## Disclosure policy

This issue was handled under coordinated disclosure. Full technical detail, including the exploit cycle, was withheld from public release until the team deployed a fix. See `../../DISCLOSURE.md`.

This advisory is provided for informational and defensive purposes only. It is not financial advice and not an endorsement of any token or project.
