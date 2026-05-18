# quay-oidc-demo

[![Build & push to Quay](https://github.com/ktdreyer/quay-oidc-demo/actions/workflows/build-push.yml/badge.svg)](https://github.com/ktdreyer/quay-oidc-demo/actions/workflows/build-push.yml)

Push container images to Quay without stored secrets, using OIDC keyless robot-account federation.

- [quay-oidc.md](quay-oidc.md) — Draft blog post for developers.redhat.com
- [quay-oidc-api.md](quay-oidc-api.md) — AI-friendly / curl-friendly version of the same setup
- [notes/](notes/) — Observations about Quay's current OIDC implementation. Things I discovered while building this demo. I will circulate these ideas more broadly, and then determine if they should be formal RFEs or bug reports to the Quay engineering team.
