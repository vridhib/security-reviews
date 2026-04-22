# CodeHawks First Flight: SNARKeling Treasure Hunt

**Auditor:** vridhib

**Date:** April 23 2026  

**Contest Link:** https://codehawks.cyfrin.io/c/2026-04-snarkeling

## Overview

This repository contains my security review of the SNARKeling Treasure Hunt protocol, conducted as part of a CodeHawks First Flight audit. The scope included the `TreasureHunt.sol` smart contract and the associated Noir ZK circuit (`main.nr`).

## Findings Summary

| Severity | Count | Description |
| :--- | :--- | :--- |
| **Medium** | 2 | Duplicate treasure hash in circuit, missing access control on `withdraw` |
| **Low** | 1 | Inconsistent funding mechanism (`fund` vs `receive`) |

The full report is available in [`findings.md`](./findings.md).

## Key Takeaway

This audit demonstrates my ability to identify cross‑domain vulnerabilities (circuit logic + smart contract business rules) and to produce clear and concise findings following industry‑standard templates.

## Contact

- **GitHub:** [vridhib](https://github.com/vridhib)
- **Discord:** 0xvridhi

---
*This audit was conducted for educational and portfolio‑building purposes as part of the Cyfrin Updraft security curriculum.*