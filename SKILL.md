---
name: soybean-admin-salvo
description: "Implement or debug a Rust + Salvo backend that serves a soybean-admin (Vue3 + Naive UI) frontend. USE whenever the user wires soybean-admin to a Salvo backend, writes the /auth/login | /auth/refreshToken | /auth/getUserInfo endpoints, hits soybean-admin login failing / requests going to undefined / token not refreshing / a 'code 0000' or envelope-shape question that is explicitly about soybean-admin, configures .env.development for the soybean dev proxy, or asks what JSON a backend must return for soybean-admin. Even casual mentions count ('我的 soybean-admin 登录连不上', 'soybean 后端要返回什么', 'salvo 接 soybean admin'). Covers the {code,msg,data} response envelope, success-code 0000 + logout/modal/expired-token code segments, JWT access+refresh flow (and the force_passed trick that makes auto-refresh work), the Res<T> Writer + JwtAuth wiring, and dev-proxy-vs-CORS / serving the built dist. SKIP when soybean-admin is not in the picture: a generic Salvo custom Writer / unified {code,msg,data} envelope with no soybean-admin context goes to salvo-skill; a backend on axum/actix/rocket/other framework is not mine; pure frontend work (tweaking Vue/Naive UI styles, components, routing) is not mine. Pairs with salvo-skill for Salvo 0.93.0 framework specifics."
---

# soybean-admin ↔ Salvo backend

You're helping the user connect a **soybean-admin** frontend (Vue3 + Naive UI admin template, `soybeanjs/soybean-admin`) to a **Rust + Salvo** backend. soybean-admin is a pure SPA; it talks to the backend over JSON with a fixed contract baked into its `src/service` layer. Your job is to make the Salvo side satisfy that contract exactly, so login → token refresh → authorized requests all work end to end.

The frontend code is the source of truth for the contract — it's not negotiable on the backend's side. Match it precisely and the integration "just works"; deviate on the envelope shape or the codes and you get silent failures (blank screen, infinite refresh loop, login that never completes).

**This skill is Salvo-specific.** For any non-trivial Salvo API (middleware constructors, `Writer`/`Scribe`, router syntax, features), defer to the **salvo-skill** and its "verify with Context7" rule — Salvo's surface shifts across minor versions and your training data lags. Target Salvo **0.93.0**.

## The contract soybean-admin expects

### 1. Every response is a uniform envelope, always HTTP 200

soybean-admin's axios layer reads success/failure from a **`code` field in the JSON body**, not the HTTP status. Return **HTTP 200** for business responses (including business "errors" like wrong password or expired token) and put the real status in `code`. The frontend's `transform` hands business code only `data` — so the real payload must live under `data`.

```jsonc
{ "code": "0000", "msg": "success", "data": { /* the real payload */ } }
```

- `code` is a **string**. The frontend tests `String(response.data.code) === VITE_SERVICE_SUCCESS_CODE`, i.e. the code must equal `"0000"` *as a whole string*. Numeric `0` stringifies to `"0"` (and even string `"0"`) won't match — it must be exactly `"0000"`.
- `msg` is shown to the user on non-success codes (the frontend dedups identical messages).

### 2. Code segments drive frontend behavior

These come from the frontend `.env` and decide what happens on each non-`0000` code:

| Codes | `.env` key | Frontend behavior |
|---|---|---|
| `0000` | `VITE_SERVICE_SUCCESS_CODE` | success → unwrap `data` |
| `9999`, `9998`, `3333` | `VITE_SERVICE_EXPIRED_TOKEN_CODES` | **auto-call `/auth/refreshToken`, then replay the original request** |
| `8888`, `8889` | `VITE_SERVICE_LOGOUT_CODES` | silently log out → redirect to login |
| `7777`, `7778` | `VITE_SERVICE_MODAL_LOGOUT_CODES` | show a modal with `msg`, then log out on confirm |
| anything else | — | reject → toast the `msg` |

### 3. Auth flow

- Frontend sends `Authorization: Bearer <token>` on every request (when a token is stored).
- On an **expired-token code** (`9999`/`9998`/`3333`), the frontend automatically POSTs the stored refresh token to `/auth/refreshToken`, stores the new pair, and replays the original request. It de-dupes concurrent refreshes.
- **CRITICAL:** `/auth/refreshToken` must **never** itself return an expired-token code — that causes an infinite refresh loop. When the refresh token is invalid/expired, return a **logout code** (`8888`) instead, so the frontend logs the user out.

## Required endpoints

soybean-admin has two route modes (`VITE_AUTH_ROUTE_MODE`). The default clone is **`static`**, which needs only three endpoints:

```
POST /auth/login          body {userName, password}  -> data {token, refreshToken}
POST /auth/refreshToken   body {refreshToken}         -> data {token, refreshToken}
GET  /auth/getUserInfo    header Bearer               -> data {userId, userName, roles[], buttons[]}
```

In `static` mode the frontend does **not** call any `/route/*` endpoint — don't implement them unless the user switches to `dynamic` mode. (`roles` gates menus/pages; the built-in super role is `R_SUPER`. `buttons` gates button-level permissions.) For `dynamic` mode (`/route/getConstantRoutes`, `/route/getUserRoutes`, `/route/isRouteExist`), see `references/dynamic-routes.md`.

## Frontend wiring — the dev gotcha (check which env file `dev` actually loads)

The #1 trap: **don't assume `pnpm dev` runs in `development` mode.** Read `package.json` first. Current soybean-admin (v2.x) ships:

```jsonc
"dev": "vite --mode test"      // NOT plain `vite`
```

Vite's `loadEnv(mode, ...)` (see `vite.config.ts`) then loads **`.env.test`**, whose `VITE_SERVICE_BASE_URL` points at an Apifox online mock (`https://mock.apifox.cn/...`). So a fresh clone's `pnpm dev` proxies to **that mock** — requests aren't hitting "nothing". And `.env.development` is **never read** under `--mode test` (it usually doesn't even exist in the repo). Creating `.env.development` therefore does nothing here — that's the trap.

To point dev at your local Salvo, pick one:
- **A — smallest change:** edit `.env.test`'s `VITE_SERVICE_BASE_URL` to your Salvo address (e.g. `http://localhost:5800`). Caveat: `.env.test` is also what `build:test` uses.
- **B — clean separation:** change the script to plain `"dev": "vite"` (mode `development`), then create `.env.development` with `VITE_SERVICE_BASE_URL=http://localhost:5800` and a valid `VITE_OTHER_SERVICE_BASE_URL` (json5). Only this form actually reads `.env.development`.

(Older / forked soybean-admin where `dev` is already plain `vite` loads `.env.development` directly — then option B's file alone is enough.)

Keep `VITE_HTTP_PROXY=Y` (in base `.env`). With the proxy on, the request baseURL becomes `/proxy-default`, which Vite proxies **same-origin** to `VITE_SERVICE_BASE_URL` — so **no CORS in dev**. CORS only matters when the browser talks to Salvo directly (typically prod). The frontend also sends a harmless `apifoxToken` header; ignore it on the Salvo side.

## Salvo implementation

Features: `salvo = { version = "0.93.0", features = ["jwt-auth", "cors", "logging"] }` plus `serde`, `serde_json`, `jsonwebtoken`, `chrono`, `tokio`, `async-trait`. Add `serve-static` only when Salvo will host the built `dist/` in prod (don't carry it by default).

> `jsonwebtoken = "9"` coexists fine with the `10.x` that Salvo pulls internally — verified compiling. Use `"10"` if you prefer to unify; the `encode`/`decode`/`Validation::new(Algorithm::HS256)` calls are unchanged.

### The envelope as a `Writer`

Implement `Writer` once so handlers return domain types and the envelope is applied uniformly. This mirrors the `ApiResponse` pattern in salvo-skill's `handlers.md`.

```rust
use async_trait::async_trait;
use salvo::prelude::*;
use serde::Serialize;

/// Always renders HTTP 200 with { code, msg, data }.
pub struct Res<T: Serialize> { pub code: &'static str, pub msg: String, pub data: Option<T> }

impl<T: Serialize> Res<T> {
    pub fn ok(data: T) -> Self { Self { code: "0000", msg: "success".into(), data: Some(data) } }
    pub fn fail(code: &'static str, msg: impl Into<String>) -> Self {
        Self { code, msg: msg.into(), data: None }
    }
}

#[async_trait]
impl<T: Serialize + Send + 'static> Writer for Res<T> {
    async fn write(self, _req: &mut Request, _depot: &mut Depot, res: &mut Response) {
        // HTTP 200 always — soybean reads status from the `code` field, not the HTTP status.
        res.render(Json(serde_json::json!({
            "code": self.code, "msg": self.msg, "data": self.data
        })));
    }
}
```

### JWT: access + refresh, and the auto-refresh trick

The key insight: to trigger the frontend's **auto-refresh**, an expired access token must come back as **HTTP 200 + `code:"9999"`**, NOT as a 401. If you let `JwtAuth` hard-reject (its default), the browser gets a 401 → axios error path → the frontend just shows an error instead of refreshing. So run `JwtAuth` with `force_passed(true)` and check the auth state inside the handler, returning the `9999` envelope yourself when it's not authorized.

```rust
use salvo::jwt_auth::{ConstDecoder, HeaderFinder, JwtAuth, JwtAuthState};
use serde::{Deserialize, Serialize};
use jsonwebtoken::{encode, EncodingKey, Header, Algorithm};

const SECRET: &[u8] = b"change-me-in-config";

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims { sub: String, kind: String, exp: i64 } // kind = "access" | "refresh"

fn sign(sub: &str, kind: &str, hours: i64) -> String {
    let exp = (chrono::Utc::now() + chrono::Duration::hours(hours)).timestamp();
    encode(&Header::new(Algorithm::HS256),
           &Claims { sub: sub.into(), kind: kind.into(), exp },
           &EncodingKey::from_secret(SECRET)).unwrap()
}

// force_passed(true): never auto-reject; we decide the response code ourselves.
fn jwt() -> JwtAuth<Claims, ConstDecoder> {
    JwtAuth::new(ConstDecoder::from_secret(SECRET))
        .finders(vec![Box::new(HeaderFinder::new())])
        .force_passed(true)
}
```

### The three handlers

```rust
#[derive(Deserialize)] struct LoginReq { #[serde(rename = "userName")] user_name: String, password: String }
#[derive(Deserialize)] struct RefreshReq { #[serde(rename = "refreshToken")] refresh_token: String }
#[derive(Serialize)] struct Tokens { token: String, #[serde(rename = "refreshToken")] refresh_token: String }
#[derive(Serialize)] struct UserInfo { #[serde(rename = "userId")] user_id: String, #[serde(rename = "userName")] user_name: String, roles: Vec<String>, buttons: Vec<String> }

#[handler]
async fn login(req: &mut Request) -> Res<Tokens> {
    let body = match req.parse_json::<LoginReq>().await {
        Ok(b) => b, Err(_) => return Res::fail("5000", "bad request"),
    };
    // TODO: verify against your store. Demo accepts Super/123456.
    if body.user_name != "Super" || body.password != "123456" {
        return Res::fail("5001", "用户名或密码错误"); // non-success code -> frontend toasts msg
    }
    Res::ok(Tokens { token: sign(&body.user_name, "access", 2), refresh_token: sign(&body.user_name, "refresh", 168) })
}

#[handler]
async fn refresh_token(req: &mut Request) -> Res<Tokens> {
    let body = match req.parse_json::<RefreshReq>().await {
        Ok(b) => b, Err(_) => return Res::fail("8888", "invalid refresh token"),
    };
    use jsonwebtoken::{decode, DecodingKey, Validation};
    let claims = decode::<Claims>(&body.refresh_token, &DecodingKey::from_secret(SECRET), &Validation::new(Algorithm::HS256));
    match claims {
        Ok(c) if c.claims.kind == "refresh" =>
            Res::ok(Tokens { token: sign(&c.claims.sub, "access", 2), refresh_token: sign(&c.claims.sub, "refresh", 168) }),
        // refresh invalid/expired -> LOGOUT code, never an expired-token code (would loop forever)
        _ => Res::fail("8888", "refresh token expired"),
    }
}

#[handler]
async fn get_user_info(depot: &mut Depot) -> Res<UserInfo> {
    // force_passed(true): inspect state and emit 9999 ourselves so the frontend auto-refreshes.
    match depot.jwt_auth_state() {
        JwtAuthState::Authorized => {
            let data = depot.jwt_auth_data::<Claims>().unwrap();
            Res::ok(UserInfo { user_id: "1".into(), user_name: data.claims.sub.clone(),
                               roles: vec!["R_SUPER".into()], buttons: vec![] })
        }
        // `_` covers both Unauthorized and Forbidden (missing / malformed / bad-signature token).
        // Emitting HTTP 200 + 9999 is what makes the frontend refresh: its auto-refresh lives in the
        // onBackendFail path (HTTP 200 branch). A real 401/4xx goes to onError, which only toasts —
        // it will NOT refresh, even if the body carries 9999. A broken token logs out after one failed refresh.
        _ => Res::fail("9999", "token expired"), // -> frontend refreshes + replays
    }
}
```

### Router

```rust
let router = Router::new().push(
    Router::with_path("auth")
        .push(Router::with_path("login").post(login))
        .push(Router::with_path("refreshToken").post(refresh_token))
        .push(Router::with_path("getUserInfo").hoop(jwt()).get(get_user_info)),
);
```

### Dev vs prod serving

- **Dev:** nothing extra — Vite proxies `/proxy-default` to Salvo same-origin, so no CORS.
- **Prod, option A (recommended):** `pnpm build` the frontend and serve `dist/` from Salvo with `StaticDir` (`serve-static` feature) → same origin, still no CORS.
- **Prod, option B:** frontend and Salvo on different origins → add the `cors` middleware (`allow_credentials(true)`, allow `authorization` + `content-type`). See salvo-skill `auth-security.md`.

## Checklist before claiming it works

- `code` is the **string** `"0000"`, payload under `data`, HTTP 200 even on business errors.
- `/auth/refreshToken` returns a **logout** code on failure, never `9999/9998/3333`.
- Protected routes use `force_passed(true)` + return `9999` on missing/expired token (so auto-refresh fires).
- Confirm **which env file `pnpm dev` actually loads** (decided by the `--mode` in the `dev` script: `vite --mode test` → `.env.test`; plain `vite` → `.env.development`) and that its `VITE_SERVICE_BASE_URL` points at your local Salvo; `VITE_HTTP_PROXY=Y` (in base `.env`). Don't blindly create `.env.development` — under `--mode test` it's ignored.
- JSON field names match (`userName`, `refreshToken`, `userId`) — soybean uses camelCase.
- Verify non-trivial Salvo API against Context7 / salvo-skill; run `cargo check`. End to end: start Salvo, `pnpm dev`, log in, confirm `getUserInfo` succeeds and an expired access token triggers one refresh + replay.
