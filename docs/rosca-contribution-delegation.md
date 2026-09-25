# Contribution Delegation in ROSCA

## Overview

Ahjoor ROSCA groups let a member delegate their **contribution-vote weight** to another member. When a member is unable to participate directly in a round's contribution vote, they can hand their voting weight to a trusted peer so the group can still reach a decision without stalling.

Contribution-vote delegation is implemented by two entry points in `contracts/ahjoor-rosca/src/lib.rs`:

- `delegate_contribution_vote` — assigns the caller's contribution-vote weight to a delegate.
- `revoke_contrib_vote_delegation` — removes an existing delegation and restores the caller's direct voting power.

The behaviour is covered by `contracts/ahjoor-rosca/src/test_contrib_delegation.rs`.

## Contribution-Vote Delegation vs. General Governance Delegation

Ahjoor ROSCA has two distinct delegation mechanisms, and they must not be confused:

| | Contribution-vote delegation | General governance delegation |
| --- | --- | --- |
| Entry points | `delegate_contribution_vote`, `revoke_contrib_vote_delegation` | `delegate_vote`, `revoke_delegation` |
| What is delegated | The member's contribution-vote weight for round contribution decisions | The member's general governance vote weight for proposals |
| Scope | Contribution voting only | Governance proposals (quorum, proposal types, etc.) |
| Revocation | `revoke_contrib_vote_delegation` | `revoke_delegation` |

In short, contribution-vote delegation affects how a member's weight counts when the group votes on contributions, while general governance delegation affects how that weight counts on governance proposals. Delegating one does not delegate the other; each must be set and revoked independently.

## Delegating Contribution Votes

A member calls `delegate_contribution_vote` to name another member as the recipient of their contribution-vote weight. From that point on, the delegate's contribution-vote weight includes the delegator's weight when contribution votes are tallied.

Typical use cases:

- A member is temporarily unavailable and wants their voice counted in a round's contribution decision.
- A group wants to consolidate contribution-vote weight behind a trusted member to reach a threshold.

## Revoking Delegation and Restoring Direct Voting

`revoke_contrib_vote_delegation` reverses a contribution-vote delegation. After revocation:

- The delegate no longer carries the caller's contribution-vote weight.
- The caller's contribution-vote weight is restored to them, so they vote directly again.

Revocation is independent of general governance delegation: calling `revoke_contrib_vote_delegation` does not affect a delegation created with `delegate_vote`, and vice versa. Members who want to stop both forms of delegation must revoke each one separately.

## Summary

- Use `delegate_contribution_vote` to hand off contribution-vote weight.
- Use `revoke_contrib_vote_delegation` to take it back and resume direct voting.
- Keep contribution-vote delegation separate from general governance delegation (`delegate_vote` / `revoke_delegation`).
