# Atlas HRMS — Repository and Path Tracker

This file is the authoritative map from Atlas components to local folders and
planned GitHub repositories. Code repositories are intentionally independent
siblings; `atlas` contains documentation only.

## Repository inventory

| Repository | Purpose | Local path | Planned GitHub remote | Git | Current phase/status |
|---|---|---|---|---|---|
| `atlas` | Architecture/specification center | `/home/oswa/atlas` | `https://github.com/jayoswal/atlas` (connected) | Existing | Active; plan/tracker added |
| `platform-outerloop` | Compose, gateway, contracts, seed/smoke | `/home/oswa/atlas-repos/platform-outerloop` | `https://github.com/jayoswal/platform-outerloop` (not connected) | Initialized | P0 scaffolded; Docker validation pending |
| `svc-identity` | Identity, org, auth/RBAC | `/home/oswa/atlas-repos/svc-identity` | `https://github.com/jayoswal/svc-identity` (not connected) | Initialized | Skeleton ready; P1 pending |
| `svc-time` | Time, attendance, PTO | `/home/oswa/atlas-repos/svc-time` | `https://github.com/jayoswal/svc-time` (not connected) | Initialized | Skeleton ready; P2 pending |
| `svc-expense` | Expense reports, FX, reimbursement | `/home/oswa/atlas-repos/svc-expense` | `https://github.com/jayoswal/svc-expense` (not connected) | Initialized | Skeleton ready; P2 pending |
| `svc-workflow` | Approvals, policy, notifications | `/home/oswa/atlas-repos/svc-workflow` | `https://github.com/jayoswal/svc-workflow` (not connected) | Initialized | Skeleton ready; P3 pending |
| `hrms-web` | React SPA | `/home/oswa/atlas-repos/hrms-web` | `https://github.com/jayoswal/hrms-web` (not connected) | Initialized | App shell ready; P1 pending |

No remote is added and no code is pushed for the six new repositories. The
owner will create/connect each remote after reviewing the local repositories.

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
| Docker Engine | 26+ | **Manual installation pending** |
| Docker Compose | v2 | **Manual installation pending** |

## Manual host prerequisite

Docker needs root access. The automation environment cannot open a sudo
password prompt, so run this directly in the Ubuntu/WSL terminal:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker "$USER"
```

Restart the WSL/Ubuntu login session, then verify:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

Docker Desktop with WSL integration is also valid; do not install both daemon
approaches unless intentionally managing the conflict.

## Identity and remote policy

- Git author: `jayoswal <jayumeshoswal2001@gmail.com>`.
- GitHub CLI active account: `jayoswal`.
- Do not use another username, email, owner namespace, or commit author.
- Do not create/add remotes or push the six new repositories until the owner
  explicitly connects them.

