# Coordinated Disclosure Policy

This repository follows coordinated disclosure. The goal is to get issues fixed without exposing users to avoidable risk in the meantime.

## Process

**1. Private report.** The affected team is contacted through a private channel with a full technical writeup, including reproduction steps and a suggested fix.

**2. Remediation window.** The team is given time to acknowledge and deploy a fix. A standard target is up to 90 days, shorter for issues with an easy fix, longer by mutual agreement.

**3. Public disclosure.** Once a fix is deployed, or the window elapses without remediation, a redacted advisory is published, followed by full technical detail after the fix is confirmed live.

## What is withheld before a fix

Until a fix is confirmed on-chain, public material omits ready-to-run exploit sequences and any detail that would let a third party trivially reproduce the attack. High-level impact and affected function names may be published earlier.

## Reporting an issue to this repository's maintainer

If you have found an issue in a contract covered here, or want to report one for review, open a minimal public issue asking for a secure contact, or reach the maintainer through the channel listed on the profile. Please do not post exploit detail in public issues.

## Good faith

Research covered here is limited to reading public, verified source and public on-chain state. It does not include, and this repository does not endorse, exploiting live contracts against real users' funds.
