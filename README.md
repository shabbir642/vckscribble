# vckscribble

A Skribbl.io-style drawing-and-guessing game where the dictionary is **corporate
slang**, and the drawer can write their own word when none of the three on offer
appeal. Web only.

| Repo | Contents |
|---|---|
| **vckscribble** (here) | [PLAN.md](./PLAN.md) — architecture, decisions, timeline |
| [vckscribble-be](https://github.com/shabbir642/vckscribble-be) | Node + Express + Socket.IO + SQLite. Owns the socket contract. |
| [vckscribble-fe](https://github.com/shabbir642/vckscribble-fe) | React + Vite. Vendors the contract. |

## Local setup

```bash
git clone git@github.com:shabbir642/vckscribble-be.git be
git clone git@github.com:shabbir642/vckscribble-fe.git fe

cd be && pnpm install && cp .env.example .env && pnpm dev   # :3001
cd fe && pnpm install && pnpm contract:sync && pnpm dev     # :5173
```

`be/` and `fe/` are gitignored here — they are independent repos that happen to
sit side by side, which is what lets `pnpm contract:sync` find the backend
contract locally instead of fetching it from GitHub.

## Status

All milestones complete (M0–M8). Playable end to end; backend verified in its
production container with 77 passing socket tests. Socket contract is at v3.

One play session's worth of fixes has landed on top — real flood fill, legible
multi-word masks, a working invite link, a pinned light theme, coloured guess
feedback and an end-of-game celebration. See the post-release section of
`PLAN.md`.

**Before deploying:** add a `BE_REPO_TOKEN` secret to the frontend repo (a PAT
with read access to the backend repo) or its CI contract check will fail.
