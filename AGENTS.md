# AGENTS.md

## Project Overview

This repository is a Four Color Cards single-player practice project built with:

- `client/`: Vue 3 + TypeScript + Vite frontend.
- `server/`: Colyseus + Express + TypeScript backend.
- `tests/e2e/`: Playwright browser regression tests.
- `docs/`: architecture, deployment, requirements, and known code/document gaps.

The current usable product scope is single-player practice: 1 human player enters the lobby and the server fills the other 3 seats with bots. Friend rooms, online matchmaking, accounts, leaderboards, multi-room lobby, and Redis persistence are reserved or partially scaffolded only. When docs and code disagree, prefer current code and tests.

## Commands

Run commands from the repository root unless noted otherwise.

```bash
npm install
npm run install:all
npm run dev
npm run build
npm run build:server
npm run build:client
npm --prefix server run test
npm run e2e
```

Useful endpoints while `npm run dev` is running:

- Frontend: `http://localhost:5173`
- Backend health: `http://localhost:2567/health`
- Colyseus monitor: `http://localhost:2567/colyseus`

Docker commands:

```bash
npm run compose:dev
npm run compose:prod
npm run compose:down
npm run compose:down:dev
```

## Validation Expectations

- For server rules, room flow, or schema changes, run `npm --prefix server run test`.
- For frontend, shared type, or build-affecting changes, run `npm run build` or the narrower `npm run build:client` / `npm run build:server`.
- For user-facing gameplay flow changes, run `npm run e2e` when practical.
- If a command cannot be run, state exactly which command was skipped and why.

## Architecture Pointers

- `server/src/index.ts`: Express/Colyseus startup and auxiliary HTTP endpoints such as `/health`, `/room-id`, `/reset-room`, and `/private-state`.
- `server/src/rooms/GameRoom.ts`: room lifecycle, message routing, seats, tokens, bots, declaration phase, and round result entry points.
- `server/src/rooms/flow/`: gameplay state machine and action execution flow.
- `server/src/rules/`: deck, action candidates, hu checking, and rule types.
- `server/src/schema/game-state.schema.ts`: public Colyseus room state.
- `client/src/composables/useRoom.ts`: room connection, state subscription, local token handling, and room helper calls.
- `client/src/App.vue`: top-level app flow, lobby, declarations, settlement, and rules modal.
- `client/src/components/GameBoard.vue`: main table display.
- `client/src/components/ActionPanel.vue`: action button presentation and submission.

## Gameplay Model Notes

- Server state is authoritative. Do not trust frontend-only checks for hu, kai, peng, chi, discard, or phase legality.
- Public room state does not contain private hands. Private hand data is sent through private messages and `/private-state`.
- Use seat IDs for gameplay logic. `sessionId` is connection-level and can change after reconnect.
- Token reconnect lets a human reclaim a bot-managed seat after disconnect.
- `responsePhase` values are `collective`, `local_upper`, and `local_draw`.
- There is no separate protocol action named `zhua`. In `local_upper`, a submitted `pass` is displayed and interpreted as "抓" in the UI.
- Bot decision priority is currently `hu > kai > peng > pass` for collective responses, and chi-first in local phases.

## Documentation Rules

- Keep `README.md`, `docs/ARCHITECTURE.md`, `docs/BUSINESS_ARCH.md`, `docs/DEPLOYMENT.md`, and `docs/DOCS_CODE_GAPS.md` aligned with code when behavior changes.
- Treat `docs/DOCS_CODE_GAPS.md` as the place for unresolved rule ambiguities instead of silently inventing behavior.
- The Chinese rule documents `四色牌游戏流程说明.md` and `四色牌操作说明.md` are useful references, but current code and tests remain the final source of truth.

## Coding Guidelines

- Keep TypeScript strictness intact. Prefer typed helpers over ad hoc `any`.
- Preserve the current server-authoritative architecture.
- Keep room flow changes small and explicit; the state machine is easier to validate when phase transitions remain localized.
- Do not add real product claims for unimplemented modes. Reserved modes should stay visibly unavailable unless backend support is actually implemented.
- Avoid introducing persistence assumptions. Redis settings exist for deployment structure, but app code currently does not persist room state to Redis.
- Do not commit generated logs such as `client-dev.log`, `server-dev.log`, or their `.err.log` variants.

## Frontend Guidelines

- The UI is game-first, landscape-oriented, and touch-friendly. Keep important controls large enough for mobile taps.
- Preserve the distinction between protocol actions and display language, especially `local_upper + pass` displayed as "抓".
- Keep action availability visually clear: disabled options may remain visible, but executable actions must be obvious.
- When changing visual layout, verify both desktop and mobile landscape behavior.

## Git Hygiene

- Check `git status --short` before editing if the task may overlap with user work.
- Do not revert unrelated user changes.
- Keep changes scoped to the requested behavior and update tests/docs when the behavior contract changes.
