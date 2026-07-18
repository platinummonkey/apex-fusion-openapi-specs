# apex-specs

Unofficial, community-maintained **OpenAPI 3.1** description of the
[Neptune Systems](https://www.neptunesystems.com/) **Apex Fusion** cloud API
(`https://apexfusion.com`).

> ⚠️ **Not affiliated with or endorsed by Neptune Systems.** This description is
> reverse-engineered from live network captures of the Fusion web app and from
> community projects. It documents only observed behavior and is intentionally
> conservative: field lists are incomplete and **hardware-dependent** — a
> controller with different modules will expose different inputs, outputs, and
> config entries.

## Files

| File | Purpose |
|------|---------|
| `openapi.yaml` | The OpenAPI 3.1 description. |
| `redocly.yaml` | Lint configuration. |
| `package.json` | `lint` / `preview` / `bundle` scripts. |
| `captures/` | Local-only raw capture payloads (git-ignored; may contain secrets). |

## Usage

```bash
npm install        # or: npx @redocly/cli@latest ...
npm run lint       # validate the spec
npm run preview    # interactive API docs (live server)
npm run docs       # build static HTML docs to dist/index.html
npm run bundle     # bundle to dist/openapi.bundled.yaml
```

## Authentication

Fusion uses **session-cookie auth** with **CSRF protection**:

1. `GET` a Fusion page — the HTML carries `<meta name="csrf-token">` and a
   pre-auth `connect.sid` cookie is set.
2. `POST /login` with JSON credentials and the `csrf-token` header. Success sets
   an authenticated `connect.sid` cookie (HttpOnly, Secure).
3. Subsequent `/api/**` calls carry that cookie; mutating calls also send
   `csrf-token`.

All XHRs send `X-Requested-With: XMLHttpRequest` and `Accept-Version: 4`.

## Coverage status

Documentation is being built up incrementally through guided live capture
against a real controller.

### Documented

- `POST /login` — authenticate
- `GET /api/user` — current user / session bootstrap
- `GET /api/apex` — list controllers (paginated `[pagination, controllers]`)
- `GET /api/apex/{apexId}` — controller document
- `GET /api/apex/{apexId}/status` — live inputs / outputs / modules
- `GET /api/apex/{apexId}/ilog` — input/probe history snapshots
- `GET /api/apex/{apexId}/dlog` — device data log
- `GET /api/apex/{apexId}/tlog` — Trident measurement log
- `GET /api/apex/{apexId}/dose` — dosing log
- `GET /api/apex/{apexId}/oday` — output state-change log
- `GET /api/apex/{apexId}/olog` — output log (paginated)
- `GET /api/apex/{apexId}/alog` — alarm log (paginated; entry shape TBD)
- `GET /api/apex/{apexId}/config/inputs` — input calibration/alarm config
- `GET /api/apex/{apexId}/config/outputs` — output programs (`prog`) + `ctype`
- `GET /api/apex/{apexId}/config/modules` — AquaBus module config
- `GET /api/apex/{apexId}/config/profiles` — reusable control profiles
- `GET /api/apex/{apexId}/config/network` — network + email/SMTP config
- `GET /api/apex/{apexId}/config/network/nsi` — firmware/AOS release info
- `GET /api/apex/{apexId}/config/heartbeat` — heartbeat config
- `GET /api/apex/{apexId}/config/flo` — flow-monitoring config
- `GET /api/user/profile` — account profile
- `GET /api/user/notification` — alert-delivery targets
- `GET /api/scheme` — named schemes (e.g. UI colors)
- `POST /api/user/notification/{id}/test` — send test alert *(inferred; not exercised)*
- `GET /api/apex/{apexId}/calendar` — reminders/events (`before` filter)
- `GET /api/apex/{apexId}/diag` — diagnostics (entry shape TBD)
- `GET /api/apex/{apexId}/config/inputs/{did}` — single input config
- `GET /api/apex/{apexId}/config/outputs/{did}` — single output config (by `did`)
- `GET /api/apex/{apexId}/config/profiles/{profileId}` — single profile (numeric id)
- `GET /api/apex/{apexId}/config/modules/{abaddr}` — single module (by AquaBus addr)
- `PUT /api/apex/{apexId}/status/outputs/{did}` — **set live output state** (verified)
- `PUT /api/apex/{apexId}/config/outputs/{did}` — **update output program/config** (verified; feed repeat interval lives here too)
- `PUT /api/apex/{apexId}/config/inputs/{did}` — **update input calibration/config** (verified)
- `PUT /api/apex/{apexId}/config/profiles/{profileId}` — **update profile** (verified)

The full config tree (including `clock`, `display`, `misc`, `ebg`, which are not
individually GET-able) is modeled as `ApexConfig`, embedded in the controller
document.

### Known but not yet documented (roadmap)

Endpoints identified from the Fusion web bundle and community clients, to be
captured and modeled in later increments:

- Session: `GET /logout`, `POST /api/signup`
- Logs / history: `/api/apex/{id}/mlog` (manual test-kit measurements — empty on
  the sampled tank). Note: `/api/apex/{id}/json` returns `404` on the cloud
  (local-controller-only route).
- Account writes: `PUT /api/user/profile`, `PUT /api/user/password`,
  notification target create/update, `POST /api/link` (link a controller)
- Further control writes: feed cycles (`PUT …/status/feed/{did}`), module and
  network config (`PUT …/config/{modules/{abaddr},network}`)
- Misc: `/api/aos`, `/api/support`

Note: per-feeder repeat interval and dosing/flow/light schedules are all encoded
in an output's `prog` string (updated via `PUT …/config/outputs/{did}`), not
discrete fields.

## Contributing

Because equipment changes, captures should note the controller model and
firmware they came from. Never commit raw payloads (cookies, serials, IPs) — the
`captures/` directory is git-ignored for this reason.
