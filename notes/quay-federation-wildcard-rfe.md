# RFE: Flexible matching for robot federation subjects

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

## How major cloud providers handle this

### AWS

AWS matches OIDC claims as **separate condition keys**, each with its own operator. For GitHub Actions, the trust policy can use `StringEquals` for exact match or `StringLike` for wildcards:

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  },
  "StringLike": {
    "token.actions.githubusercontent.com:sub": "repo:MyOrg/*"
  }
}
```

The wildcard applies to a single claim value, so it cannot cross into other claims. AWS also matches on `aud` as a separate key — a natural defense-in-depth layer.

### GCP

GCP Workload Identity uses [CEL (Common Expression Language)](https://cloud.google.com/iam/docs/workload-identity-federation-with-other-providers#create-custom-mapping) for attribute conditions. You can write arbitrary boolean expressions:

```
assertion.repository_owner == 'my-org'
```

Each OIDC claim is a separate field in the `assertion` object. No delimiter-crossing risk, and far more expressive than string matching.

### Azure

Azure recently added a [claims matching expression](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-set-up-flexible-federated-identity-credential?tabs=azure-portal%2Cgithub#set-up-a-flexible-federated-identity-credential) feature (preview, March 2026). It supports wildcards on individual claim values using a `matches` operator:

```
claims['sub'] matches 'repo:my-org/*:ref:refs/heads/main'
```

It can also match on separate claims like `repo_property_team` and combine conditions with `&&`. Azure's documentation [recommends using immutable ID-based claims](https://josh-ops.com/posts/azure-federated-credential-claims-matching-expressions/) (`repository_owner_id`, `repository_id`) instead of name-based ones, since org and repo names can be renamed.

## Why simple wildcards on the `sub` claim are risky

GitHub's `sub` claim is colon-delimited: `repo:org/repo:ref:refs/heads/branch`. A naive wildcard like `fnmatch` treats `*` as matching everything including `:`, so it crosses field boundaries.

Example:

- Pattern: `repo:myorg/*:ref:refs/heads/main`
- Intended: any repo in `myorg`, but only the `main` branch
- Actually matches: `repo:myorg/foo:environment:prod:ref:refs/heads/main` — because `*` swallows `foo:environment:prod`

Colons are also valid in Git branch names, so a branch named `main:ref:refs/heads/main` could match patterns it shouldn't.

AWS avoids this entirely by matching each claim as a separate key. GCP avoids it with CEL expressions. Azure's new claims matching expression feature applies wildcards to individual claim values and recommends using immutable IDs over names.

## Proposed behavior

Match on individual OIDC claims as separate fields, similar to AWS. Federation entries would gain optional claim fields beyond `subject`:

```json
[
  {
    "issuer": "https://token.actions.githubusercontent.com",
    "claims": {
      "repository_owner": "opendatahub-io"
    }
  }
]
```

This single entry would grant access to any workflow in any `opendatahub-io` repository. The matching logic would check each configured claim against the corresponding claim in the decoded JWT.

For backward compatibility, existing `{"issuer", "subject"}` entries would continue to work with exact matching.

If Quay prefers a simpler approach, `StringLike`-style matching (wildcards on individual claim values, not on concatenated strings) would be a safe middle ground — matching AWS's well-established pattern.

## Security: path reuse and mutable identifiers

The scalability problem above has a security counterpart. On both GitHub and GitLab, organization/group and project paths can be renamed, deleted, and re-created by other users. OIDC trust relationships that rely solely on path-based claims like `sub` may grant access to unintended actors if a deleted project is re-created under the same path.

AWS recently emailed all customers with GitLab OIDC trust relationships, recommending they add conditions on stable, unique identifiers — such as `namespace_id` or `project_id` — rather than relying on path-based claims alone. The same principle applies to GitHub, where `repository_owner` and `repository` are mutable names.

Both providers include immutable numeric claims in their OIDC tokens:

**GitHub Actions:**

| Mutable (name-based) | Immutable (ID-based) |
| :---- | :---- |
| `repository` | `repository_id` |
| `repository_owner` | `repository_owner_id` |

**GitLab CI:**

| Mutable (path-based) | Immutable (ID-based) |
| :---- | :---- |
| `project_path` | `project_id` |
| `namespace_path` | `namespace_id` |

AWS, Azure, and GCP all recommend matching on immutable IDs. Azure's [claims matching expression docs](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-set-up-flexible-federated-identity-credential?tabs=azure-portal%2Cgithub#set-up-a-flexible-federated-identity-credential) explicitly warn against name-based claims because names can be transferred. AWS now supports validating additional custom claims (beyond `sub`) from both GitHub and GitLab IdPs in IAM role trust policies specifically to enable this.

Because Quay can only match on the `sub` claim, and `sub` is composed of mutable names, there is no way to use immutable IDs today. Supporting individual claim matching (as proposed above) would let users write federation entries like:

```json
[
  {
    "issuer": "https://token.actions.githubusercontent.com",
    "claims": {
      "repository_owner_id": "123456"
    }
  }
]
```

This would be resilient to path changes regardless of GitHub or GitLab name transfers.

## Workaround available today

GitHub allows [customizing the subject claim template](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-subject-claims-for-an-organization-or-repository) at the org level. Setting `{"include_claim_keys": ["repository_owner"]}` changes the `sub` claim to `repository_owner:opendatahub-io`, which a single exact-match federation entry can match. However, **this changes the subject format for the entire GitHub organization and will affect other OIDC integrations.** For ODH, this concern is not theoretical: we are already using OIDC to GCP Vertex for [live integration testing](https://github.com/opendatahub-io/ogx-distribution).

## Tested on

quay.io (SaaS), May 2026. Source code inspected at quay/quay commit `a56c75df4`.
