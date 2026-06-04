# Dynamic route mode (`VITE_AUTH_ROUTE_MODE=dynamic`)

Only relevant when the user switches soybean-admin from `static` to `dynamic` route mode. In `dynamic` mode the **backend owns the menu/route tree** and the frontend fetches it after login. The default clone is `static` — skip all of this unless the user explicitly uses dynamic mode.

All three responses use the same `{code,msg,data}` envelope (HTTP 200, success `"0000"`) as the auth endpoints.

```
GET  /route/getConstantRoutes                  -> data: MenuRoute[]            (constant/login-free routes)
GET  /route/getUserRoutes                      -> data: { routes: MenuRoute[], home: string }
GET  /route/isRouteExist?routeName=<name>      -> data: boolean
```

- `getUserRoutes.home` is the route **name** to redirect to after login (e.g. `"home"`); it must exist in `routes`.
- `isRouteExist` is used when adding routes by name; return whether a route with that name exists.

## MenuRoute shape

soybean-admin's route objects follow its `@elegant-router` types. **Verify the exact fields against the frontend's `src/typings/elegant-router.d.ts` and `src/typings/api/route.d.ts`** before finalizing — the shape evolves with the template version. A representative node:

```jsonc
{
  "name": "manage",                 // unique route name
  "path": "/manage",                // path
  "component": "layout.base",       // layout / view ref (elegant-router convention)
  "meta": {
    "title": "manage",
    "i18nKey": "route.manage",      // i18n key
    "icon": "carbon:cloud-service-management",
    "order": 1,
    "roles": ["R_SUPER"],           // role gate (optional)
    "hideInMenu": false,
    "constant": false               // true => login-free (constant route)
  },
  "children": [
    {
      "name": "manage_user",
      "path": "/manage/user",
      "component": "view.manage_user",
      "meta": { "title": "manage_user", "i18nKey": "route.manage_user", "icon": "ic:round-manage-accounts" }
    }
  ]
}
```

## Salvo sketch

Reuse the `Res<T>` writer from SKILL.md; serve the route tree from config or DB. Protect `/route/getUserRoutes` with the same JWT hoop + `force_passed(true)` + `9999`-on-unauthorized pattern as `getUserInfo`.

```rust
#[handler]
async fn get_constant_routes() -> Res<Vec<serde_json::Value>> {
    Res::ok(vec![ /* constant MenuRoute nodes: 403/404/500/login etc. */ ])
}

#[handler]
async fn get_user_routes(depot: &mut Depot) -> Res<serde_json::Value> {
    match depot.jwt_auth_state() {
        JwtAuthState::Authorized => Res::ok(serde_json::json!({
            "routes": [ /* MenuRoute[] gated by this user's roles */ ],
            "home": "home"
        })),
        _ => Res::fail("9999", "token expired"),
    }
}

#[handler]
async fn is_route_exist(req: &mut Request) -> Res<bool> {
    let name = req.query::<String>("routeName").unwrap_or_default();
    Res::ok(/* lookup */ !name.is_empty())
}
```

Build these with strongly-typed structs (deriving `Serialize`) instead of `serde_json::Value` once you've pinned the exact `MenuRoute` fields from the frontend types.
