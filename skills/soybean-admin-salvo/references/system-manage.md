# System-manage endpoints (`example` branch) + pagination contract

The soybean-admin **`main` branch is a slim template** — only `_builtin` + `home` views, only `/auth/*` (+ `/route/*` in dynamic mode). The **`example` branch** (what the online demo and Apifox mock correspond to) adds the system-manage pages (`src/views/manage/*`) and with them a second API family: `/systemManage/*`. If the user's frontend has 用户管理/角色管理/菜单管理 pages, they're on the example branch (or copied those pages) and the backend needs these endpoints.

All endpoints below are **GET**, all params arrive as **query string**, all responses use the same `{code,msg,data}` envelope as auth. Field names verified against `example` branch `src/service/api/system-manage.ts` + `src/typings/api/system-manage.d.ts`.

## Pagination contract (applies to every list endpoint)

Request: `?current=1&size=10` (+ optional search filters, all nullable — receive as `Option<...>`).
Response `data`:

```jsonc
{
  "records": [ /* the page of rows */ ],
  "current": 1,    // page number, 1-based
  "size": 10,      // page size
  "total": 57      // total row count (NOT total pages)
}
```

Every record also carries the common audit fields (note: `createBy`/`createTime`, **not** createdAt/updatedAt):

```jsonc
{ "id": 1, "createBy": "admin", "createTime": "2024-01-01 10:00:00",
  "updateBy": "admin", "updateTime": "2024-01-01 10:00:00", "status": "1" }
```

`status` is an **enum string**: `"1"` = enabled, `"2"` = disabled (nullable). Same convention for `userGender` (`"1"` male / `"2"` female), `menuType` (`"1"` directory / `"2"` menu), `iconType` (`"1"` iconify / `"2"` local).

## The endpoints

```
GET /systemManage/getRoleList?current=&size=&roleName=&roleCode=&status=   -> paginated Role
GET /systemManage/getAllRoles                                              -> [{id, roleName, roleCode}]
GET /systemManage/getUserList?current=&size=&userName=&userGender=&nickName=&userPhone=&userEmail=&status=
                                                                           -> paginated User
GET /systemManage/getMenuList/v2                                           -> paginated Menu (NOTE the /v2 suffix!)
GET /systemManage/getAllPages                                              -> ["home", "manage_user", ...]
GET /systemManage/getMenuTree                                              -> [{id, label, pId, children?}]
```

Gotcha: the menu list URL really is `getMenuList/v2` — a plain `/systemManage/getMenuList` route 404s.

Entity fields (camelCase, on top of the common audit fields):

- **User**: `userName`, `userGender` (`"1"|"2"|null`), `nickName`, `userPhone`, `userEmail`, `userRoles: string[]` (role codes)
- **Role**: `roleName`, `roleCode`, `roleDesc`
- **Menu**: `parentId: number` (0 = root), `menuType` (`"1"|"2"`), `menuName`, `routeName`, `routePath`, `component?`, `icon`, `iconType` (`"1"|"2"`), `buttons?: {code, desc}[]`, `children?`, plus route-meta props (`i18nKey`, `order`, `hideInMenu`, `keepAlive`, …) mirroring the route system

The template only ships **read** endpoints — the demo's create/edit/delete buttons mutate local table state. When the user wants real CRUD, design the write endpoints freely (the frontend code for them is the user's to write), but keep the envelope and field-name conventions.

## Salvo sketch

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
struct UserSearchParams {
    current: Option<u64>,            // default 1
    size: Option<u64>,               // default 10
    user_name: Option<String>,
    user_gender: Option<String>,
    nick_name: Option<String>,
    user_phone: Option<String>,
    user_email: Option<String>,
    status: Option<String>,
}

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct Page<T: Serialize> { records: Vec<T>, current: u64, size: u64, total: u64 }

#[handler]
async fn get_user_list(req: &mut Request) -> Res<Page<User>> {
    // parse_queries is sync — no .await (see salvo-skill data-extraction.md)
    let p: UserSearchParams = match req.parse_queries() {
        Ok(p) => p, Err(_) => return Res::fail("5000", "bad query"),
    };
    let (current, size) = (p.current.unwrap_or(1), p.size.unwrap_or(10));
    // ... filter + page your store with the Option fields ...
    Res::ok(Page { records, current, size, total })
}

let manage = Router::with_path("systemManage")
    .hoop(jwt())                                  // same force_passed(true) guard as getUserInfo
    .push(Router::with_path("getUserList").get(get_user_list))
    .push(Router::with_path("getRoleList").get(get_role_list))
    .push(Router::with_path("getAllRoles").get(get_all_roles))
    .push(Router::with_path("getMenuList/v2").get(get_menu_list))
    .push(Router::with_path("getAllPages").get(get_all_pages))
    .push(Router::with_path("getMenuTree").get(get_menu_tree));
```

These routes are JWT-protected in spirit (admin pages), so reuse the `force_passed(true)` + `9999` pattern from SKILL.md — a hard 401 here would break the auto-refresh on the manage pages too.
