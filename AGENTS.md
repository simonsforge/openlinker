# AGENTS.md — OpenLinker

Open-source, self-hosted e-commerce orchestration platform (syncs orders, inventory, and listings between shops and marketplaces). pnpm monorepo, TypeScript strict, Hexagonal Architecture (ports and adapters).

A more detailed agent guide lives in `CLAUDE.md`; read it for the full rules. Start-of-session reads: `docs/lessons.md` (empirical regression ledger — review before working, add an entry after correcting a repeatable mistake), `docs/engineering-standards.md`, `docs/architecture-overview.md`.

## Layout

- `libs/core/` — domain logic, capability ports (`*.port.ts`), core services. Platform-agnostic.
- `libs/integrations/` — adapters implementing CORE ports, one package per platform (Allegro, PrestaShop, …).
- `libs/oms/`, `libs/plugin-sdk/`, `libs/shared/` (shared logging/DTOs), `libs/test-kit/`
- `apps/api/` — NestJS REST API, port 3000 (`@openlinker/api`)
- `apps/worker/` — background job worker (`@openlinker/worker`)
- `apps/web/` — React 19 + Vite admin SPA, port 4173 (`@openlinker/web`)
- `apps/e2e/` — Playwright black-box suite against a running stack (not in PR CI; read-only `smoke` project runs weekly via `.github/workflows/e2e.yml`)
- `apps/prestashop-module/` — PHP PrestaShop module (`pnpm test:php`)

## Setup

```bash
pnpm install                                  # Node 22+, pnpm 10+ (engines; pnpm pinned via packageManager)
cp apps/api/.env.example apps/api/.env.local
cp apps/worker/.env.example apps/worker/.env.local
pnpm dev:stack:up                             # Postgres 17, Redis, MySQL, PrestaShop, WooCommerce in Docker
pnpm --filter @openlinker/api migration:run
pnpm test
```

## Commands

| Task | Command |
|---|---|
| Build all | `pnpm build` |
| Unit tests | `pnpm test` (no Docker; fast) |
| Watch / coverage | `pnpm test:watch` / `pnpm test:cov` |
| Integration tests | `pnpm test:integration` (Testcontainers — real Postgres + Redis, needs Docker) |
| PHP module tests | `pnpm test:php` |
| Lint | `pnpm lint` (runs `pnpm -r lint` **plus `pnpm check:invariants`**) |
| Type check | `pnpm type-check` |
| Format | `pnpm format` (Prettier; check: `pnpm format:check`) |
| Affected tests only | `pnpm smart-test` (diff-driven jest selection; used by the pre-commit hook) |
| Scaffold a new integration | `pnpm create-adapter <name>` then follow `docs/plugin-author-guide.md` |
| Dev servers | `pnpm start:dev:api` / `start:dev:worker` / `start:dev:web` |
| Migrations | `pnpm --filter @openlinker/api migration:{generate,run,revert,show}` |

## Quality gate (before every commit)

```bash
pnpm lint && pnpm type-check && pnpm test
```

The Husky pre-commit hook runs `pnpm lint`, `pnpm type-check`, and `pnpm smart-test --no-integration` — a libs/core touch runs only related specs. Never skip hooks with `--no-verify`.

## Architecture rules

- **CORE ↔ Integration boundary is strict.** `libs/core` defines capability ports; `libs/integrations/*` implement them and never bleed domain logic back into CORE. Prefer reusing existing abstractions; justify any new pattern.
- **Frontend layering enforced:** `app` → `pages` → `features` → `shared`; `shared` must not import upward. Server state = TanStack Query, forms = React Hook Form, no general-purpose global store.
- Any ORM entity change in `libs/core/src/**/*.orm-entity.ts` requires a TypeORM migration (`docs/migrations.md`). No `synchronize: true`.

## Code style

- TypeScript strict; no `any` (use `unknown` + narrowing); explicit return types on public API.
- No `console.log` — use the shared `Logger` from `libs/shared/src/logging/`.
- Prettier: single quotes, semicolons, 2 spaces, 100 print width, trailing comma es5.
- Naming: ports `*.port.ts` / `{Capability}Port`, adapters `*-adapter.ts` / `{Platform}{Capability}Adapter`, services `*.service.ts` + `*.service.interface.ts`, entities `*.orm-entity.ts`, unit tests `*.spec.ts`, integration tests `*.int-spec.ts`; FE: components `PascalCase.tsx`, hooks `use-*.ts`, routes `*.route.tsx`.
- Backend user input validated with class-validator DTOs; TypeORM query builder, no interpolated raw SQL; `@UseGuards(JwtAuthGuard)` on non-public endpoints.

## Workflows & gotchas

- **Conventional Commits** (`feat`, `fix`, `docs`, …) with DCO sign-off: `git commit -s`. Branch from `main` named `{issue-number}-{short-kebab-description}`. PRs close issues via `Closes #N` — never close an issue manually.
- **`pnpm check:invariants`** runs dozens of `scripts/check-*-mirror.mjs` scripts that assert mirrored definitions (permissions, VAT, order lifecycle, UI vocabulary, etc.) stay in sync across packages. If a check fails after touching a mirrored type/constant, update **all** mirrors — the failing script names them.
- Migration files are timestamp-validated by `scripts/check-migration-timestamps.mjs`; generate migrations via the script, don't hand-number.
- Never commit `.env` / `.env.local` files (gitignored; `.worktreeinclude` excludes them from worktrees).
- Security vulnerabilities: never file publicly — follow `SECURITY.md`.
- Repo URL references in code/docs are checked by `scripts/check-repo-urls.mjs`; keep them pointing at the canonical repo.
