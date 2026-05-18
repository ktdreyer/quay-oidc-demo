# RFE: Document the robot federation API endpoint in the Swagger spec

## Summary

The robot federation endpoint at `/api/v1/organization/{orgname}/robots/{robot_shortname}/federation` works but is not documented in the [API discovery spec](https://quay.io/api/v1/discovery). The spec lists the path but defines no HTTP methods, request body schema, or response examples.

## Current behavior

The discovery spec contains only metadata for this endpoint:

```json
"x-name": "endpoints.api.robot.OrgRobotFederation"
"x-path": "/api/v1/organization/{orgname}/robots/{robot_shortname}/federation"
"x-tag": "robot"
```

No GET, POST, PUT, or DELETE operations are defined.

## Actual behavior (discovered by testing)

The endpoint supports at least two methods:

**GET** returns the current federation configuration as a JSON array:

```
GET /api/v1/organization/{orgname}/robots/{robot_shortname}/federation
Authorization: Bearer <token>

Response 200:
[{"issuer": "https://token.actions.githubusercontent.com", "subject": "repo:myorg/myrepo:ref:refs/heads/main"}]
```

**POST** accepts a JSON array of federation entries. Multiple entries are supported (e.g. to allow the same robot to authenticate from different repositories). **POST uses replace semantics** — it overwrites all existing federation config for the robot. To add an entry without removing existing ones, GET the current config, append the new entry, and POST the full array.

Duplicate entries within a single POST are rejected with a 400 error.

```
POST /api/v1/organization/{orgname}/robots/{robot_shortname}/federation
Authorization: Bearer <token>
Content-Type: application/json

[
  {"issuer": "https://token.actions.githubusercontent.com", "subject": "repo:myorg/repo-a:ref:refs/heads/main"},
  {"issuer": "https://token.actions.githubusercontent.com", "subject": "repo:myorg/repo-b:ref:refs/heads/main"}
]

Response 200:
[
  {"issuer": "https://token.actions.githubusercontent.com", "subject": "repo:myorg/repo-a:ref:refs/heads/main"},
  {"issuer": "https://token.actions.githubusercontent.com", "subject": "repo:myorg/repo-b:ref:refs/heads/main"}
]
```

Both require an OAuth token with the `org:admin` scope.

## Request

Add the GET and POST operations to the Swagger spec for this endpoint, including request/response schemas and authentication requirements. This would allow users to script robot federation setup instead of using the UI.

## Tested on

quay.io (SaaS), May 2026. Source code inspected at quay/quay commit `a56c75df4`.
