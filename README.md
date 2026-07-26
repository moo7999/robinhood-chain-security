> **Independent research disclaimer:** This is independent, unofficial security research. It is not affiliated with, endorsed by, or sponsored by Robinhood Markets, Inc.

# Robinhood Chain Security

Independent security research and coordinated-disclosure advisories for smart contracts deployed on Robinhood Chain (chain ID 4663).

Findings here are the result of manual source review of verified contracts, using on-chain data from the Robinhood Chain block explorer. Every advisory is handled under coordinated disclosure: the affected team is notified privately first and given time to remediate before full technical detail is published.

## Advisories

| ID | Target | Severity | Status |
|---|---|---|---|
| [RHC-2026-001](advisories/RHC-2026-001/README.md) | Rob Domains — DomainMarketplace | High (DoS) | Partially mitigated |

## Lower-severity findings

Grouped notes that don't warrant a standalone advisory live in `findings/`.

## Scope and method

**Chain:** Robinhood Chain, ID 4663.

**Source:** only contracts with verified source on the block explorer are reviewed; unverified bytecode is out of scope.

**Technique:** manual review plus live state reads. No fuzzing harness or formal verification is claimed. Findings should be independently confirmed before any remediation is relied upon.

## Disclaimer

All content is provided for informational and defensive security purposes only. Nothing here is financial advice, an audit, a guarantee, or an endorsement of any token, protocol, or project. Reviews are best-effort and may be incomplete. Interacting with any contract mentioned is at your own risk.

## Contact

Responsible-disclosure contact and PGP details: see `DISCLOSURE.md`.

Maintained by MO3TH.
