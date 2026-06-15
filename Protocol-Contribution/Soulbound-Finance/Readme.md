# Soulbound Security – SBF Protocol

## Overview

This directory documents my security review, threat analysis, mitigation design process and implementation contribution for the SBF Protocol maintained by Soulbound Security.

The work focused on strengthening execution boundaries around gas-fund operations by introducing explicit control over external integration targets.

## Contribution

My work included:

* Reviewing the existing gas-fund interaction model.
* Identifying security considerations around unrestricted target selection.
* Evaluating multiple mitigation approaches and their associated trade-offs.
* Documenting the rationale for rejecting earlier mitigation proposals.
* Designing and implementing the final approved-target whitelist approach.
* Submitting a protocol improvement pull request for review.

## Outcome

### Status

- 2 findings merged.
- 1 finding accepted (merge pending).

## References
* Repo: [LINK](https://github.com/SoulboundSecurity/sbf-protocol)

