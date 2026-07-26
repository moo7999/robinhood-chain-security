# Lower-severity findings

Notes that don't warrant a standalone advisory but are worth recording. All relate to Rob Domains contracts on Robinhood Chain (chain ID 4663) unless noted.

## F-001 — getOffersByMaker is unbounded (Medium)

Contract: DomainMarketplace `0xA023960c23c7EFFC18c58b0049F14D4a85d2d85d`

`_tokenIdsWithOffers` is append-only and never pruned. `getOffersByMaker` runs a nested loop over every tracked token and every offer on each — O(tokens × offers). It is a view, so no funds are at risk, but it will start reverting against RPC gas limits well before the marketplace is large, taking the "My Offers" page down with it.

Suggested fix: maintain a per-maker index, or add offset/limit pagination. Prune tokens from `_tokenIdsWithOffers` when they have no remaining active offers.

## F-002 — DomainNFT.safeTransferFrom ignores ERC-721 receiver check (High, unfixable in place)

Contract: DomainNFT `0xF960519a5e2a88FcB83c6a91A45602FE3B4555a1`

The NFT contract's safeTransferFrom does not call onERC721Received on contract recipients, contrary to the ERC-721 spec implied by the function name. A domain transferred to a contract that cannot handle NFTs is permanently stuck. The contract is not upgradeable (no proxy), so this cannot be patched on the existing deployment.

Suggested handling: document the behavior publicly so integrators do not assume standard safe-transfer semantics; if a future NFT contract is deployed, implement the receiver check correctly.

## F-003 — Single-EOA admin, no timelock or pause (Medium, governance)

Contracts: DomainMarketplace, DomainNFT

`admin` is a single externally owned account. On the marketplace it can cancel any user's listing, change `feeRecipient`, and transfer admin. There is no multisig, no timelock, and no emergency pause. For a contract custodying offer ETH this concentrates significant trust in one key.

Suggested fix: move admin to a multisig, add a timelock on sensitive parameter changes, and consider a scoped pause for the offer/buy paths.

## F-004 — Legacy marketplace approvals remain live (Medium, operational)

Contract: old DomainMarketplace `0xE62Eef82F99c53A4bC2A97609673A3ACa89530c2`

After the marketplace redeploy, sellers who listed on the previous contract still have live `setApprovalForAll` / per-token approvals to it, and the old contract may still hold ETH from pending offers. Live approvals to a deprecated contract are standing risk.

Suggested handling: publish a notice instructing users to revoke approvals to the old contract, and document how to withdraw anything left in it.
