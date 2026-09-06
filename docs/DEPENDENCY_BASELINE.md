# Atlas HRMS — Reproducible Dependency Baseline

> Exact application/runtime baseline verified on 2026-09-06. Compatibility
> ranges remain documented in architecture/specification text; these values,
> committed lockfiles, and immutable image digests control reproducible builds.

## Runtime selectors

| Component | Exact selection | Source |
|---|---:|---|
| Python | 3.12.14 | `.python-version` in four services, platform tools, and service template |
| Node.js | 20.20.2 | `hrms-web/.nvmrc` |
| npm | 10.8.2 | Bundled with the selected Node.js release |
| uv (container builds) | 0.12.10 | Exact install in service Dockerfiles/template |

`requires-python = ">=3.12,<3.13"` and `engines.node = ">=20 <21"` remain
compatibility declarations. The selector files choose the exact development
runtime; container digests choose the exact build/runtime filesystem.

## Backend and platform direct dependencies

The four services and generated service template use the same exact set.
Every transitive package and artifact hash is fixed by the committed `uv.lock`
in each executable Python repository.

| Package | Exact version | Scope |
|---|---:|---|
| aio-pika | 9.6.2 | Services |
| alembic | 1.13.3 | Services |
| argon2-cffi | 23.1.0 | Services |
| fastapi | 0.115.14 | Services |
| httpx | 0.27.2 | Services and platform tools |
| psycopg[binary] | 3.2.13 | Services |
| pydantic | 2.13.5 | Services |
| pydantic-settings | 2.15.0 | Services |
| pyjwt | 2.13.0 | Services |
| sqlalchemy | 2.0.52 | Services |
| uvicorn[standard] | 0.30.6 | Services |
| jsonschema | 4.26.0 | Service development |
| mypy | 1.20.2 | Service development |
| pytest | 8.4.2 | Service development |
| pytest-asyncio | 1.4.0 | Service development |
| ruff | 0.16.6 | Service and platform development |
| cookiecutter | 2.7.1 | Platform development |

## Frontend direct dependencies

All transitive packages and integrity hashes are fixed by
`hrms-web/package-lock.json`.

| Package | Exact version | Package | Exact version |
|---|---:|---|---:|
| @emotion/react | 11.14.0 | @emotion/styled | 11.14.1 |
| @mui/material | 5.18.0 | @reduxjs/toolkit | 2.12.0 |
| react | 18.3.1 | react-dom | 18.3.1 |
| react-redux | 9.3.0 | react-router-dom | 7.18.3 |
| @eslint/js | 9.39.5 | openapi-typescript | 7.13.0 |
| @testing-library/jest-dom | 6.9.1 | @testing-library/react | 16.3.3 |
| @types/react | 18.3.31 | @types/react-dom | 18.3.7 |
| @vitejs/plugin-react | 4.7.0 | eslint | 9.39.5 |
| eslint-plugin-react-hooks | 5.2.0 | eslint-plugin-react-refresh | 0.4.26 |
| globals | 15.15.0 | jsdom | 25.0.1 |
| typescript | 5.4.5 | typescript-eslint | 8.69.0 |
| vite | 5.4.21 | vitest | 2.1.9 |

## Container images

Every reference uses a readable exact tag plus a content-addressed digest.
The digest, not the mutable registry tag, determines downloaded content.

| Image | Exact tag | Immutable digest |
|---|---:|---|
| python | 3.12.14-slim | `sha256:78387bc3881b8273120a12ebe6c1ab22b018ccc2c9adf565ae1ac9b536e184ea` |
| node | 20.20.2-alpine | `sha256:fb4cd12c85ee03686f6af5362a0b0d56d50c58a04632e6c0fb8363f609372293` |
| nginx | 1.27.5-alpine | `sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10` |
| postgres | 16.15 | `sha256:f1c3376c26f2609ab9f29f71f824103fe2fcd8ee0346485cb6122a4f93df6f94` |
| rabbitmq | 3.13.7-management | `sha256:e582c0bc7766f3342496d8485efb5a1df782b5ce3886ad017e2eaae442311f69` |
| traefik | 3.7.13 | `sha256:f86a2cab1b5c649070c49f883c743dd32d8485a56e3368c5f93b9e91f1e91259` |
| adminer | 4.17.1 | `sha256:c1c24d89d06fcb8fc0fb5a049240dc292ff4a121ae05d7e74d64b58c421d2921` |
| mailhog/mailhog | v1.0.1 | `sha256:8d76a3d4ffa32a3661311944007a415332c4bb855657f4f6c57996405c009bea` |

## CI actions

| Action | Release | Full commit SHA |
|---|---:|---|
| actions/checkout | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |

All future third-party `uses:` entries must follow the same full-SHA rule, with
the human-readable release in a comment.

## Tested host (documented, not package-pinned)

| Component | Tested version |
|---|---:|
| Ubuntu | 26.04 x86_64 |
| Git | 2.53.0 |
| Docker Engine | 29.8.0 |
| Docker Compose | 5.5.1 |
| uv | 0.12.10 |
| pip (host user site) | 26.2.1 |

These host tools intentionally use minimum-supported constraints in setup docs.
Exact apt packages are not held because security updates must remain available.
Application output is controlled by runtime selectors, lockfiles, and image
digests rather than the host package patch level.

The GitHub-hosted runner label (`ubuntu-24.04`) is also a supported platform
selector rather than an immutable artifact. Third-party Actions executed on it
remain full-SHA pinned.

## Updating the baseline

1. Update one dependency family deliberately:
   - Python: edit the exact requirement, then run `uv lock`.
   - Frontend: run `npm install --save-exact <package>@<version>`.
   - Image: resolve the exact tag with
     `docker buildx imagetools inspect <image>:<tag>`, then update tag and digest.
   - Action: resolve the trusted release tag to its full upstream commit SHA.
2. Update every copied service/template occurrence and this document.
3. Run:

   ```bash
   uv run --project /home/oswa/atlas-repos/platform-outerloop \
     python /home/oswa/atlas-repos/platform-outerloop/scripts/check_pins.py \
     /home/oswa/atlas-repos
   ```

4. Run all repository quality gates, rebuild images without cache, start the
   complete estate, and run `scripts/seed.py` plus `scripts/smoke.py`.
5. Review and merge; automated dependency PRs must never bypass these checks.
