# StubServer Backend — Postman Automation Collection

Complete automation collection covering all 48 endpoints across 13 modules.

---

## Files

| File | Purpose |
|------|---------|
| `StubServer-Backend-Automation.postman_collection.json` | Main collection (100+ requests) |
| `QA.postman_environment.json` | QA environment variables |
| `UAT.postman_environment.json` | UAT environment variables |

---

## Prerequisites

```bash
npm install -g newman newman-reporter-htmlextra
```

---

## Import into Postman

1. Open Postman → **Import**
2. Drop in `StubServer-Backend-Automation.postman_collection.json`
3. Drop in the environment file for your target (`QA.postman_environment.json` or `UAT.postman_environment.json`)
4. Select the imported environment from the top-right dropdown
5. Fill in real values for `admin_password` and `test_password` in the environment editor

---

## Running via Newman

```bash
cd postman/
newman run StubServer-Backend-Automation.postman_collection.json \
  -e QA.postman_environment.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export report.html
```

Pass a different env file to target UAT:
```bash
newman run StubServer-Backend-Automation.postman_collection.json \
  -e UAT.postman_environment.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export report.html
```

---

## Seed `test_username` Before Running Chained Flows

The chained flows (User Lifecycle, Password Cycle) require `test_username` to exist.

**Option A — let the Flow_User_Lifecycle create it** (it's step UL-1)

**Option B — pre-seed manually:**
```bash
curl -s -X POST http://your-server:9092/backend/api/signUp \
  -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -d '{"username":"autotest_user","email":"autotest@corp.com","firstname":"Auto","lastname":"Test","requestedBy":"admin","userrole":"ApplicationUser"}'
```

---

## Chained Flows

| Flow | Steps | What it proves |
|------|-------|---------------|
| Flow_Password_Cycle | 7 | Full password change + reset cycle, reused-token 403 |
| Flow_User_Lifecycle | 7 | Create → verify → modify → verify → delete → verify |
| Flow_Group_Tag_Cycle | 8 | Mutate group/tags, assert, verify duplicate-tag 400, restore |
| Flow_LiveURL_Cycle | 9 | Add URL → set active → 409 delete → switch active → delete |
| Flow_AppPort_Cycle | 7 | Add port → 409 dup → modify → no-change → delete |
| Flow_ExecutionMode_Cycle | 6 | Failover → invalid 400 → Stand In → verify → restore |

All flows restore original state so the collection can run twice in a row (idempotent).

---

## Troubleshooting

### 401 on every request
- The `Admin Login` in `00_Setup` failed — check `admin_username` / `admin_password` in your environment.
- The collection-level pre-request script auto-attaches the token only after login succeeds.

### 403 on admin endpoints
- Your `admin_username` must have role `Admin` in the system.
- Check `USERROLE` via `GET /getUsersList`.

### Chained flow fails mid-way
- A collection variable (e.g., `created_portid`, `created_vsurlid`) wasn't saved because a prior step failed.
- Run the flow folder in isolation in Postman to debug step by step.

### `test_username` duplicate on re-run
- If the User Lifecycle flow failed before the delete step, the user may still exist.
- Delete it manually: `DELETE /delete` with `{"username":"autotest_user","requestedBy":"admin"}`.

### LiveURL Cycle leaves URL2 active
- By design — URL1 is always deleted. URL2 (autotest-url2-<timestamp>) may remain active.
- Each run uses a unique timestamp-based host so there is no collision on re-runs.

---

## Environment Variables Reference

| Key | Description |
|-----|-------------|
| `baseUrl` | API base URL, e.g. `http://host:9092/backend/api` |
| `admin_username` | Admin login username |
| `admin_password` | Admin login password (**never commit real values**) |
| `test_username` | Username for lifecycle/password tests |
| `test_email` | Email for `test_username` |
| `test_password` | Password for `test_username` |
| `test_service` | Service name that exists in the catalog |
| `test_serverIP` | Server IP for execution mode / metrics endpoints |
| `environment` | Environment label: `QA` or `UAT` |
| `default_app_name` | App name used in port management cycle (`AutoTestApp`) |
| `default_port_range` | Safe port range for tests (`9500-9510`) |
| `sample_log_file` | Log filename for download tests (optional) |
| `sample_reqresp_log` | ReqResp log filename for download tests (optional) |
| `sample_dataset_file` | Dataset filename for download tests (optional) |
