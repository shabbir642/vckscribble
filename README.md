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

M0 (scaffold) and M1 (dictionary) complete. M2 (rooms) is next.
