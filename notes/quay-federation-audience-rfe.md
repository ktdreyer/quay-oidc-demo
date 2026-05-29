# RFE: Verify the `aud` (audience) claim in robot federation tokens

## Summary

Quay's robot federation token exchange does not verify the `aud` (audience) claim in OIDC tokens. A token minted for a different service could be replayed against Quay's federation endpoint.

## Current behavior

When verifying a federated OIDC token, Quay explicitly disables audience verification (`util/security/federated_robot_auth.py:92`):

```python
options = {"verify_aud": False, "verify_nbf": False}
decoded_token = service.decode_user_jwt(token, options=options)
```

Quay only checks `iss` (issuer) and `sub` (subject). The `aud` and `azp` claims are ignored.

## Risk

If an attacker obtains a valid OIDC token from GitHub Actions that was minted for a different audience (e.g. AWS, GCP, or another service), they could present that same token to Quay's `/oauth2/federation/robot/token` endpoint. As long as the `iss` and `sub` match a federation entry, Quay will accept it.

In practice, this requires the attacker to intercept a token that also happens to have a matching subject claim, which limits the attack surface. Verifying `aud` is required by the [OpenID Connect Core spec, Section 3.1.3.7](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) (step 3: "The Client MUST validate that the aud (audience) Claim contains its client_id value").

## Proposed behavior

1. Allow federation entries to optionally specify an `audience` field:

```json
[
  {
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "audience": "https://quay.io"
  }
]
```

2. When `audience` is set on a federation entry, verify the token's `aud` claim matches.

3. When `audience` is not set, skip verification (backward compatible with existing configs).

On the GitHub Actions side, workflows would request a token with a specific audience:

```
$ACTIONS_ID_TOKEN_REQUEST_URL&audience=https://quay.io
```

This ensures the token was minted specifically for Quay and cannot be replayed from another service exchange.

## How other providers handle audience

There is no universal convention for OIDC audience values. Each provider defines its own:

- **AWS** hardcodes a global value: `sts.amazonaws.com`. Every client uses the same string. Simple, but inflexible.
- **GCP** always enforces the `aud` claim. If the admin does not configure `allowed_audiences`, GCP defaults to requiring the workload identity provider's full resource path (e.g. `//iam.googleapis.com/projects/123/locations/global/workloadIdentityPools/pool/providers/provider`). See [GCP's REST API reference](https://docs.cloud.google.com/iam/docs/reference/rest/v1/projects.locations.workloadIdentityPools.providers).
- **HashiCorp Vault** uses per-role `bound_audiences` configured by the admin. The client must match.
- **PyPI** uses a discovery endpoint. The client GETs `https://upload.pypi.org/_/oidc/audience` and PyPI responds with the expected audience value. The client then requests an OIDC token with that audience. See [`oidc-exchange.py` in pypa/gh-action-pypi-publish](https://github.com/pypa/gh-action-pypi-publish/blob/main/oidc-exchange.py).

PyPI's approach is the most robust: the server publishes its expected audience, so clients never have to guess. Without a published canonical value, clients must hardcode a string that may not match what the server eventually enforces.

Quay could expose a similar discovery endpoint (e.g. `https://quay.io/_/oidc/audience`) so that clients always know the correct value.

## Tested on

quay.io (SaaS), May 2026. Source code inspected at quay/quay commit `a56c75df4`.
