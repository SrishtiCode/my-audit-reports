# Audit Reports

Practice smart contract security audits written against real-world protocols from past [Code4rena](https://code4rena.com) and [CodeHawks](https://codehawks.com) contests. Vulnerabilities in these reports are already known and fixed — this repo documents my process of independently finding, reproducing, and reporting them.

---

## Why This Exists

Security auditing is a skill built through repetition. Each report here represents a full audit cycle:

- Reading the protocol specification and mapping the architecture
- Running static analysis tooling (Slither, Aderyn)
- Manual line-by-line review
- Writing proof-of-concept tests in Foundry
- Producing a structured report as I would for a real engagement

The goal is to build muscle memory for the process, not just the findings.

---

## Reports

| Protocol | Findings (H/M/L) | Report |
| -------- | :----------------: | ------ |
| Venus Prime | 3 / 3 / 8 | [View](./VenusPrime.md) |
| *(more coming)* | | |

---

## Report Structure

Each report follows a consistent format:

- **Protocol Summary** — what the protocol does and its intended security guarantees
- **Architecture Overview** — contract map, data flow diagram, roles, and key state variables
- **Audit Methodology** — tooling used, phases followed, and time allocation
- **Findings** — each vulnerability with description, impact, proof of concept, and recommended mitigation

---

## Disclaimer

All protocols audited here are from past contests. The vulnerabilities are already public and have been fixed. These reports are written for **educational and portfolio purposes only**. They do not represent a live security engagement.

---

## About

Made by [Srishti](https://SrishtiCode.io) — learning smart contract security one audit at a time.

## Support

If you find this repository useful:
- Star ⭐ the repo  
- Share feedback  
- Open for opportunities
