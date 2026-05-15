**Quay OIDC setup via the API**

This guide covers the same Quay-side setup as the [blog post](quay-oidc.md), but uses `curl` instead of the UI. This is useful for automation or for AI coding agents that can run shell commands.

**Note:** The federation API endpoint only exists for organization robots, not user-namespace robots. If you need federation, use an organization on quay.io.

The GitHub Actions workflow YAML is the same in both cases — see [section 3 of the blog post](quay-oidc.md#3-use-the-oidc-token-in-a-github-actions-workflow).

---

## Prerequisites

You need an OAuth access token with the **Administer Organization** and **Administer Repositories** scopes.

1. In the Quay UI, go to your organization and click **Applications** in the left nav.
2. Click **Create New Application**, give it a name (e.g. `api-setup`), and press Enter.
3. Click the application name, then the **Generate Token** tab.
4. Select the **Administer Organization** and **Administer Repositories** scopes, then click **Generate Access Token**.
5. Complete the OAuth flow and copy the token. It is only shown once.

```bash
export QUAY_API_TOKEN="<your-token>"
```

---

## 1. Create a robot account

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description": "GitHub Actions OIDC robot"}' \
  "https://quay.io/api/v1/organization/<org>/robots/<robot-name>"
```

Example:

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"description": "GitHub Actions OIDC robot"}' \
  "https://quay.io/api/v1/organization/thoughtful-code/robots/githubactions"
```

---

## 2. Configure federation

The request body is a JSON array of `{"issuer", "subject"}` objects. You can include multiple entries to allow the same robot to authenticate from different repositories or providers.

**Important:** POST uses **replace semantics** — it overwrites the entire federation config for the robot. If you want to add an entry to an existing config, GET the current config first, add the new entry to the array, and POST the full array back.

```bash
curl -s -X POST \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"issuer":"https://token.actions.githubusercontent.com","subject":"repo:<owner>/<repo>:ref:refs/heads/<branch>"}]' \
  "https://quay.io/api/v1/organization/<org>/robots/<robot-name>/federation"
```

Example with a single entry:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"issuer":"https://token.actions.githubusercontent.com","subject":"repo:thoughtful-code/claudio:ref:refs/heads/main"}]' \
  "https://quay.io/api/v1/organization/thoughtful-code/robots/githubactions/federation"
```

Example with multiple entries (one robot serving two repositories):

```bash
curl -s -X POST \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {"issuer":"https://token.actions.githubusercontent.com","subject":"repo:thoughtful-code/claudio:ref:refs/heads/main"},
    {"issuer":"https://token.actions.githubusercontent.com","subject":"repo:ktdreyer/quay-oidc-demo:ref:refs/heads/main"}
  ]' \
  "https://quay.io/api/v1/organization/thoughtful-code/robots/githubactions/federation"
```

Verify with a GET to the same endpoint:

```bash
curl -s -H "Authorization: Bearer $QUAY_API_TOKEN" \
  "https://quay.io/api/v1/organization/thoughtful-code/robots/githubactions/federation"
```

The federation endpoint is listed in the [API discovery spec](https://quay.io/api/v1/discovery) at `/api/v1/organization/{orgname}/robots/{robot_shortname}/federation`. Note that this API is under-documented for now. In the future we'll improve quay's source code and document it further as an RFE.

---

## 3. Create the target repository

The robot needs a repository to push to. If it does not already exist:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"repository":"<repo>","namespace":"<org>","visibility":"public","description":"<description>"}' \
  "https://quay.io/api/v1/repository"
```

Example:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"repository":"claudio","namespace":"thoughtful-code","visibility":"public","description":"Claudio container image"}' \
  "https://quay.io/api/v1/repository"
```

---

## 4. Grant robot write access to the repository

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"write"}' \
  "https://quay.io/api/v1/repository/<org>/<repo>/permissions/user/<org>+<robot-name>"
```

Example:

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $QUAY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"write"}' \
  "https://quay.io/api/v1/repository/thoughtful-code/claudio/permissions/user/thoughtful-code+githubactions"
```

This step requires the **Administer Repositories** scope on your OAuth token.

---

## Further reading

- [Red Hat Quay API guide](https://docs.redhat.com/en/documentation/red_hat_quay/3/html-single/red_hat_quay_api_guide/index)
- [API discovery spec](https://quay.io/api/v1/discovery)
- [Keyless authentication with robot accounts](https://docs.redhat.com/en/documentation/red_hat_quay/3.16/html/manage_red_hat_quay/keyless-authentication-robot-accounts) — documents the `/oauth2/federation/robot/token` exchange endpoint
- [About security hardening with OpenID Connect](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect) — subject claim formats for the federation POST body
- [Quay OIDC blog post](quay-oidc.md) — GUI walkthrough and GitHub Actions workflow
