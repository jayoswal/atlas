# Atlas HRMS — Repository and Path Tracker

This file is the authoritative map from Atlas components to local folders and
public GitHub repositories. Code repositories are intentionally independent
siblings; `atlas` contains documentation only.

## Repository inventory

| Repository | Purpose | Local path | GitHub remote | Git | Current phase/status |
|---|---|---|---|---|---|
| `atlas` | Architecture/specification center | `/home/oswa/atlas` | `https://github.com/jayoswal/atlas` (connected) | Existing | Active; plan/tracker added |
| `platform-outerloop` | Compose, gateway, contracts, seed/smoke | `/home/oswa/atlas-repos/platform-outerloop` | `https://github.com/jayoswal/platform-outerloop` | Connected/pushed | P0 scaffolded; compose config valid |
| `svc-identity` | Identity, org, auth/RBAC | `/home/oswa/atlas-repos/svc-identity` | `https://github.com/jayoswal/svc-identity` | Connected/pushed | Skeleton ready; P1 pending |
| `svc-time` | Time, attendance, PTO | `/home/oswa/atlas-repos/svc-time` | `https://github.com/jayoswal/svc-time` | Connected/pushed | Skeleton ready; P2 pending |
| `svc-expense` | Expense reports, FX, reimbursement | `/home/oswa/atlas-repos/svc-expense` | `https://github.com/jayoswal/svc-expense` | Connected/pushed | Skeleton ready; P2 pending |
| `svc-workflow` | Approvals, policy, notifications | `/home/oswa/atlas-repos/svc-workflow` | `https://github.com/jayoswal/svc-workflow` | Connected/pushed | Skeleton ready; P3 pending |
| `hrms-web` | React SPA | `/home/oswa/atlas-repos/hrms-web` | `https://github.com/jayoswal/hrms-web` | Connected/pushed | App shell ready; P1 pending |

All six code repositories are public, use `origin` in the `jayoswal` namespace,
and track the pushed `main` branch.

## Workspace invariant

Keep the six code repositories as siblings:

```text
/home/oswa/atlas-repos/
├── hrms-web/
├── platform-outerloop/
├── svc-expense/
├── svc-identity/
├── svc-time/
└── svc-workflow/
```

The compose build contexts rely on this relationship. The layout is portable
to an Ubuntu VM; clone all six into the same parent path (the parent directory
name itself may differ).

## Toolchain status

| Tool | Required | Installed/status |
|---|---|---|
| Git | 2.40+ | 2.53.0 |
| Python runtime | 3.12 | Managed automatically per project by `uv` |
| pip | Current | 26.2.1, user site |
| uv | Current | 0.12.10, `~/.local/bin/uv` |
| Node | 20 | 20.20.2, managed by `~/.nvm` |
| npm | Node 20 bundled | 10.8.2 |
| Docker Engine | 26+ | 29.8.0; daemon and test container verified |
| Docker Compose | v2 | v5.5.1; Atlas compose config verified |

## Docker status

Docker is installed, the WSL session has inherited the `docker` group, and
daemon access works without `sudo`. Verification commands:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

## Identity and remote policy

- Git author: `jayoswal <jayumeshoswal2001@gmail.com>`.
- GitHub CLI active account: `jayoswal`.
- Do not use another username, email, owner namespace, or commit author.
- All six code repositories are public and connected to their `jayoswal`
  remotes.
