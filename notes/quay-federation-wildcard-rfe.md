# RFE: Support wildcard patterns in robot federation subject matching

## Summary

Robot federation subject matching uses exact string comparison. This makes it impractical to grant a single robot access to all repositories in a GitHub organization, because each repository requires its own federation entry.

## Current behavior

Quay stores federation entries as `{"issuer", "subject"}` pairs and compares the OIDC token's `sub` claim against each entry using exact string matching (`util/security/federated_robot_auth.py`, line 101):

```python
if decoded_token.get("sub") not in matched_subs:
    raise InvalidRobotCredentialException("Token does not match robot")
```

A GitHub Actions OIDC token for `ktdreyer/quay-oidc-demo` on the `main` branch has:

```
"sub": "repo:ktdreyer/quay-oidc-demo:ref:refs/heads/main"
```

This means each GitHub repository (and each branch) needs its own federation entry. A wildcard subject like `repo:ktdreyer/*` is accepted by the federation config API, but the token exchange returns HTTP 400 because the exact string `repo:ktdreyer/*` does not match the token's actual subject.

## Problem at scale

An organization with 100 repositories that all need to push to Quay must create 100 separate federation entries on the robot account. If branches other than `main` also need access, the number multiplies further. Adding a new repository requires updating the robot's federation config.

## Proposed behavior

Support glob patterns in the subject field. For example:

| Pattern | Matches |
| :---- | :---- |
| `repo:myorg/*` | Any repo in `myorg`, any ref |
| `repo:myorg/*:ref:refs/heads/main` | Any repo in `myorg`, but only the `main` branch |
| `repo:myorg/my-app:ref:refs/heads/*` | One repo, any branch |

The matching logic in `verify_federated_robot_jwt_token()` would use `fnmatch` (or equivalent) instead of the `in` operator when the subject contains a `*` character. Entries without wildcards would continue to use exact matching for backward compatibility.

## Alternative

Instead of (or in addition to) wildcard subjects, Quay could match on additional OIDC claims beyond `sub`. GitHub's OIDC tokens include claims like `repository_owner`, `repository`, and `repository_visibility`. Matching on `repository_owner` alone would cover the "all repos in an org" use case without wildcards.

## Tested on

quay.io (SaaS), May 2026. Source code inspected at quay/quay commit `a56c75df4`.
