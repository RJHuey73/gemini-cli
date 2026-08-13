# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What This Is

**Gemini CLI** is Google's open-source, terminal-first AI agent
(`google-gemini/gemini-cli`; this repo, `RJHuey73/gemini-cli`, is a fork).
It gives a terminal-native interface to Gemini models with built-in tools
(file ops, shell, web fetch, Google Search grounding) and MCP (Model Context
Protocol) support for custom integrations. Apache-2.0 licensed.

Note: the repo also ships its own `GEMINI.md` at the root — that file is
project context read by *Gemini CLI itself* when it works on this codebase
(dogfooding). This `CLAUDE.md` is the equivalent context file for Claude
Code; keep the two in sync when project facts change, but don't assume
Gemini-CLI-specific tooling mentioned there (e.g. its `.gemini/skills/`
skill set) applies here.

## Layout

npm workspaces monorepo (`workspaces: ["packages/*"]`), Node >=20.0.0
(`.nvmrc` pins `20`), TypeScript, ESM (`"type": "module"`).

| Path | Package | Purpose |
|------|---------|---------|
| `packages/cli` | `@google/gemini-cli` | User-facing terminal UI (React/Ink), input handling, display rendering. `src/ui`, `src/config`, `src/commands`, `src/services`, `src/acp`, `src/core`. |
| `packages/core` | `@google/gemini-cli-core` | Backend: Gemini API orchestration, prompt construction, tool execution, agents, routing, MCP client, policy/safety, telemetry. Large `src/` with `tools/`, `agents/`, `mcp/`, `routing/`, `skills/`, `hooks/`, `prompts/`, `code_assist/`, `scheduler/`, `billing/`, `ide/`. |
| `packages/a2a-server` | `@google/gemini-cli-a2a-server` | Experimental Agent-to-Agent server. |
| `packages/sdk` | `@google/gemini-cli-sdk` | Programmatic SDK for embedding Gemini CLI capabilities. |
| `packages/devtools` | `@google/gemini-cli-devtools` | Integrated developer tools (network/console inspector); has its own `client/`. |
| `packages/test-utils` | `@google/gemini-cli-test-utils` | Shared test utilities used across workspaces. |
| `packages/vscode-ide-companion` | `gemini-cli-vscode-ide-companion` | VS Code extension pairing with the CLI. |
| `docs/` | — | User/dev docs (get-started, cli, core, tools, extensions, ide-integration, reference, admin, changelogs); `docs/CONTRIBUTING.md` mirrors root `CONTRIBUTING.md`. |
| `integration-tests/` | — | E2E tests, run via Vitest against a built CLI, optionally sandboxed (Docker/Podman). |
| `memory-tests/`, `perf-tests/` | — | Nightly-only regression suites (memory usage, CPU perf) against committed baselines — not part of `preflight`. |
| `evals/` | — | Behavioral eval suite (Vitest-driven). |
| `scripts/` | — | Build/release/lint/tooling scripts (`build.js`, `start.js`, `clean.js`, `lint.js`, settings/keybindings doc generators, etc.), invoked by root `package.json` scripts. |
| `.gemini/skills/` | — | Gemini CLI's own skills (docs-writer, pr-creator, ci, code-reviewer, ...) — used when *Gemini CLI* works on this repo, not a Claude Code mechanism. |
| `sea/` | — | Single-executable-application launch path/tests. |
| `third_party/` | — | Vendored third-party code. |

## Commands

All run from the repo root unless noted.

```bash
npm install                 # install deps (workspaces)
npm run build                # build all packages
npm run build:all            # build + sandbox + VS Code companion
npm run start                 # run in development
npm run debug                  # run with Node inspector (--inspect-brk)
npm run bundle                # esbuild bundle -> bundle/gemini.js

npm run test                  # unit tests, all workspaces (--if-present) + sea-launch test
npm run test:ci                # CI variant of the above + scripts tests
npm test -w <pkg> -- <path>    # single workspace, e.g.:
npm test -w @google/gemini-cli-core -- src/routing/modelRouterService.test.ts
npm run test:e2e                # integration tests, no sandbox
npm run test:integration:sandbox:docker   # integration tests, Docker sandbox
npm run test:memory             # nightly-only; run locally only if touching that area
npm run test:perf               # nightly-only; ditto

npm run lint                   # eslint (cache, --max-warnings 0)
npm run lint:fix                # eslint --fix + format
npm run format                  # prettier --write .
npm run typecheck                # tsc across workspaces + evals/integration-tests/memory-tests

npm run preflight                # clean + ci install + format + build + lint:ci + typecheck + test:ci
                                   # heaviest check — run once at the END of a task, not iteratively
```

For fast iteration during a code change, prefer targeted commands
(`npm run test`, `npm run lint`, workspace-scoped `npm test -w <pkg> -- <path>`)
over `preflight`, and only run `preflight` before finishing/submitting. Skip
`preflight` entirely for docs-only or prompt-text-only changes.

## Conventions

- **Imports**: use specific imports; ESLint (`no-restricted-imports`,
  configured per-workspace in `eslint.config.js`) blocks certain relative
  imports *between* packages — go through a package's public exports rather
  than reaching into another workspace's internals.
- **License headers**: every new `.ts`/`.tsx`/`.js` source file needs an
  Apache-2.0 header with the current year, e.g.:
  ```
  /**
   * @license
   * Copyright 2026 Google LLC
   * SPDX-License-Identifier: Apache-2.0
   */
  ```
  This is enforced by ESLint — check an existing neighboring file for the
  exact format rather than retyping from memory.
- **Commit messages**: [Conventional Commits](https://www.conventionalcommits.org/).
- **Contributions**: follow `CONTRIBUTING.md` (Google CLA required). Keep PRs
  small, focused, and linked to an existing issue where one exists.
- **Environment-variable tests**: use `vi.stubEnv('NAME', 'value')` in
  `beforeEach` / `vi.unstubAllEnvs()` in `afterEach` — not direct
  `process.env` mutation (leaks across tests). To unset a variable, stub it
  to `''`.
- **Docs**: live under `docs/`; when a code change makes existing docs stale
  or incomplete, call that out / propose the doc update alongside the code
  change.

## Testing

- **Framework**: Vitest everywhere (unit, integration, evals, memory, perf,
  scripts tests all have their own Vitest configs).
- **Unit tests** run per-workspace (`npm run test --workspaces --if-present`);
  colocated with source (`*.test.ts(x)` next to the file it tests, e.g.
  `packages/cli/src/gemini.test.tsx`).
- **Integration/E2E** (`integration-tests/`) runs against a *built* CLI and
  can run unsandboxed or sandboxed via Docker/Podman
  (`GEMINI_SANDBOX=false|docker|podman`); see `docs/integration-tests.md`.
- **Memory/perf tests** compare against committed baselines and are
  deliberately excluded from `preflight` — they run nightly in CI. Only run
  them locally when your change actually touches memory or performance
  behavior; otherwise rely on CI.
- **Flaky tests**: `npm run deflake -- --command="<cmd>"` reruns a command
  repeatedly to help diagnose flakiness (see `test:integration:flaky` and the
  `deflake:*` scripts for prewired examples).

## Gotchas

- **Don't confuse `GEMINI.md` with this file.** `GEMINI.md` is read by
  Gemini CLI when it operates on its own codebase; it references Gemini-CLI
  skills (`docs-writer`, `pr-creator`, etc.) under `.gemini/skills/` that
  have no Claude Code equivalent here — don't try to invoke them as Claude
  Code skills.
- **`preflight` is expensive** (`clean && npm ci && format && build &&
  lint:ci && typecheck && test:ci`) — it's the full pre-submit gate, not a
  quick sanity check. Use targeted `test`/`lint`/`typecheck` commands while
  iterating.
- **Workspace-scoped test paths are relative to the workspace root**, not
  the repo root — e.g. `npm test -w @google/gemini-cli-core -- src/foo.test.ts`,
  not `packages/core/src/foo.test.ts`.
- **Sandbox modes matter for integration tests**: `GEMINI_SANDBOX=docker`
  rebuilds the sandbox image first (`test:integration:sandbox:docker` chains
  `build:sandbox`); running the wrong sandbox target without the image built
  will fail confusingly.
- **`npm install` alone doesn't rebuild the bundle** — `prepare` (husky +
  `npm run bundle`) runs on install via the `prepare` lifecycle script, but
  after further changes you generally need an explicit `npm run build` (or
  `npm run bundle` for the esbuild single-file output) before `npm start`
  reflects them, depending on what you changed.
- This is a **fork** (`RJHuey73/gemini-cli` of `google-gemini/gemini-cli`).
  Be mindful that `README.md`, `ROADMAP.md`, and issue/PR automation docs
  describe the upstream project's process (Google CLA, upstream CI badges,
  upstream release cadence), which may not all apply identically to work
  done in this fork.
