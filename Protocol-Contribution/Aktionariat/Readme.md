# Aktionariat Contracts – Security Findings

## Overview

This directory documents two security findings identified in the [Aktionariat Contracts](https://github.com/aktionariat/contracts) repository and privately disclosed under the project's Bug Bounty Program.

## Disclosure

Both findings were privately reported to Aktionariat through their Bug Bounty
Program.

The reported issues were reproduced against the latest public commit of the
repository at the time of review, with working proof-of-concepts.

## Outcome

# 📊 Findings Summary

- **Total Findings:** 2
- **Severity Breakdown:**
  - 🟥 High: 2
  - 🟧 Medium: 0
  - 🟨 Low / Informational: 0

# 🚀 Findings Overview

| # | Finding | Severity |
|---|---------|----------|
| 1 | Zero-value `transferFrom()` can bypass Admin authorization and allowlist arbitrary addresses | 🔴 High |
| 2 | Frozen / Restricted holders can bypass the freeze and redeem via `unwrap()` | 🔴 High |

## Project Response

Aktionariat confirmed the reported findings and stated that they had already been fixed in newer versions / the latest commit.

But, both findings were identified and successfully reproduced against the latest public commit available at the time of the review. Following their response, clarification was requested regarding the specific fixing commit/version and the prior disclosure in which the issues had already been documented, but no such details were provided.

Proof-of-concepts and reproduction evidence from the audited code are included
in the `image` folder.

## References

* Repo: [https://github.com/aktionariat/contracts](https://github.com/aktionariat/contracts)
