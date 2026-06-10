# soybean-admin-salvo skill

A Claude Code skill for implementing/debugging a **Rust + Salvo backend that serves a soybean-admin (Vue3 + Naive UI) frontend**. Install as a plugin (see below) or by symlinking into `~/.claude/skills/`.

## What it does

Makes the Salvo backend satisfy soybean-admin's fixed frontend service-layer contract so login → token refresh → authorized requests work end to end.

- Hub: `SKILL.md` — the contract (the `{code,msg,data}` envelope always at HTTP 200, success code string `"0000"`, logout/modal/expired-token code segments), required endpoints (static mode = `/auth/login` + `/auth/refreshToken` + `/auth/getUserInfo`), main-vs-example branch shapes, negative contracts (logout is frontend-only; no change-password endpoint; the captcha is fake), how to add custom APIs (incl. the differently-shaped `demoRequest` second instance), the dev-env gotcha, and a paste-ready Salvo impl (`Res<T>` Writer, JWT access+refresh, the `force_passed(true)` trick that makes the frontend auto-refresh fire).
- `references/system-manage.md` — the **`example`-branch** `/systemManage/*` contracts: `getUserList`/`getRoleList`/`getAllRoles`/`getMenuList/v2` (yes, the `/v2` suffix is real)/`getAllPages`/`getMenuTree`, the `{records, current, size, total}` pagination envelope, `createBy`/`createTime` audit fields, string enums (`status`/`userGender`/`menuType`/`iconType` as `"1"`/`"2"`), and a Salvo pagination handler sketch.
- `references/dynamic-routes.md` — the `/route/*` endpoints, only needed in `dynamic` auth-route mode.
- `evals/` — output evals + trigger-eval set.

## Key gotchas it encodes

- `code` must be the **string** `"0000"` (not numeric `0`), payload under `data`, **HTTP 200 even on business errors**.
- `/auth/refreshToken` must return a **logout** code on failure, never an expired-token code — an expired code there makes the refresh response `await` its own in-flight refresh promise: a silent deadlock where every request just hangs (no retry, no logout).
- Protected routes use `JwtAuth::force_passed(true)` + return `code "9999"` on missing/expired token, so the frontend's auto-refresh (which lives in the HTTP-200 path) fires.
- **Dev env trap:** soybean-admin v2.x `"dev": "vite --mode test"` loads `.env.test` (Apifox mock), **not** `.env.development`. Point dev at local Salvo by editing `.env.test`, or switch the script to plain `vite` + create `.env.development`.
- **Menu list 404 trap:** the example-branch menu page calls `GET /systemManage/getMenuList/v2` — a plain `getMenuList` route 404s. `getMenuList/v2`, `getAllRoles`, `getAllPages`, `getMenuTree` are called with **no parameters**; only `getUserList`/`getRoleList` send `current`/`size` + filters.

## Division of labor

| Layer | Owner |
|---|---|
| soybean-admin ↔ backend **contract** (envelope, codes, auth flow, endpoints, dev proxy) | **soybean-admin-salvo** (this skill) |
| **Salvo framework** API specifics (Writer/Scribe, JwtAuth signatures, routing, OpenAPI) | `salvo-skill` (target 0.93.0 + Context7 rule) |
| Rust **language** layer (Send across .await, error design, lifetimes) | `rust-idioms` |

## Properties & verification

Zero hooks / zero MCP / zero extra permissions — pure reference text. The Salvo code in `SKILL.md` was `cargo check`-verified against salvo 0.93.0 (0 errors) by an independent review (codex + agy), the frontend contract was cross-checked against the real soybean-admin v2.2.0 source, and the dev-gotcha fix was behavior-verified by re-running the eval. Built/maintained 2026-06 via the skill-creator flow.

The v0.1.3 round (2026-06-10) added an **adversarial review**: 10 claim groups attacked against the real soybean-admin `main` + `example` branch sources — all core contracts survived with line-level evidence; two descriptions were precision-fixed (the refresh-expired-code failure mode is a silent promise self-deadlock, not a retry loop; the demo's create/edit/delete buttons are stubs that re-fetch, not local-state mutations). A/B eval (menu-manage 404 scenario, new vs 0.1.1): both pass the contract assertions, but the 0.1.1 run accepted refresh tokens on protected routes (no `kind` check) and skipped the user/role endpoints — the new version closes both.

## Changelog

- **0.1.3** (2026-06-10) — adversarial-review precision pass (refresh deadlock semantics, stub buttons, no-params note for menu endpoints).
- **0.1.2** (2026-06-10) — new `references/system-manage.md` (`/systemManage/*` paginated contracts, `getMenuList/v2`), main-vs-example branch guidance, negative contracts, `demoRequest` envelope warning, pagination checklist items.
- **0.1.1** — `Res<T>` defensive `status_code(OK)`, MenuRoute `id` field; cargo-verified; packaged as installable plugin.

## Install

- **As a plugin (marketplace):** `/plugin marketplace add waydone/soybean-admin-salvo`, then `/plugin install soybean-admin-salvo@soybean-admin-salvo`.
- **Local (symlink):** from the repo root, `ln -sfn "$PWD/skills/soybean-admin-salvo" ~/.claude/skills/soybean-admin-salvo`.

## Updating after an edit

Bump `version` in `.claude-plugin/{plugin,marketplace}.json`, `git push`, then `claude plugin marketplace update soybean-admin-salvo && claude plugin update soybean-admin-salvo@soybean-admin-salvo`, and restart. Without a version bump, `plugin update` reports "already at latest" and won't pull the new content.
