# Deploy through the Back4app MCP — lit-container-starter — 2026-11-10

Repo commit uploaded: `aa767da` (branch `main`, clean tree).

## Context (same for the 9 starters measured this day)

- Date: 2026-11-10 (all times UTC).
- Account kind: permanent, signed in (`whoami` -> `accountKind: "permanent"`, `limits: {freeAppsUsed: 19, maxFreeApps: 50}` before this run). Apps created on the Free plan.
- Client: Claude Code driving the Back4app MCP connector; terminal on macOS (Node v25.8.1, curl 8.7.1); validation requests left from Sao Paulo (CloudFront POP `GRU1-P5`).
- Upload CLI: `@back4app/cli` 0.3.2, run as `npx -y @back4app/cli upload --url <upload-link>` inside the starter folder. The upload link carries a one-time token and is not recorded here. `create_upload` returned `expiresAt` 15 minutes after the call.
- "create -> READY" below = deployment `createdAt` to the `updatedAt` of the `get_deployment` answer that first said `READY`. It is the platform's own clock, not the wall time of the polling.

## Tools called, in order

Part A: `create_upload` (no appId) -> CLI upload in the folder -> `create_web_app` -> `get_web_app` -> `get_deployment` -> `get_runtime_logs`

Part B3: `create_upload` (no appId) -> CLI upload in the folder -> `create_web_app` (`port: 3000`) -> `get_web_app` -> `get_deployment` -> `get_web_app` -> `get_build_logs`

Dry run: `update_web_app` WITHOUT confirmationId on `mcp-lit` -> `get_web_app` (to check nothing changed)

## Part A — first deploy

### Upload (CLI output)

```text
Packing .../lit-container-starter ...
24 file(s), 141 KB (2 left out)
Sending 12 new file(s) ...
Upload ready.
uploadId: ff8c6123-0164-4a83-993c-f27caefc768f
manifestSha256: 07e86af6f8e5fcf5d279da3d346a63149edf932b1563a7795917709f49a893be
```

- Wall time: 7 s. No "may hold secrets" message for this folder.
- The repo tracks 17 files; the CLI packed 24, which is every file in the folder outside `node_modules/` and `.git/` (local `dist/` included: the CLI does not read `.gitignore`).

### App and deployment

- `create_web_app` arguments: `name: "mcp-lit"`, `port: 8080`, `uploadId`, `manifestSha256`; `builder` not specified.
- appId `c44bec93-7be0-483f-83ca-73c52ffaa384` · deploymentId `603d76cd-1669-4e0b-ab8d-affa720db282`
- App `createdAt` 12:55:34.384Z · deployment `createdAt` 12:55:34.531Z
- Status seen: `BUILDING` / `BUILDING_IMAGE` (12:55:34) -> `READY` (`updatedAt` 12:55:57.939Z) -> **23.4 s**
- Public URL: 404 at 12:55:43, first 200 at 12:55:57.

Settings reported by `get_web_app`:

- `builder`: `DOCKERFILE` · `port`: `8080` · `startCommand`: `null` · `healthCheckPath`: `"/"` · `isStaticSite`: `false` · `rootDir`: `""` · `runtimeVersion`: `null` · `envVarNames`: `[]` · `plan`: `Free` · `source.kind`: `UPLOAD`
- `url`: https://mcplit-pmzngefc.back4.app
- `urlExpiresAt`: 2026-11-10T13:55:34.471Z (60 minutes after the app was created)
- `get_deployment` returned no `suggestions` and no `portDetected` on this deployment.

### Validation on the public URL (12:56Z)

- `GET /` -> 200, `content-type: text/html`, `server: nginx/1.27.5`, `via: 1.1 ... .cloudfront.net (CloudFront)`, no `content-encoding` (507 B document).
- JS referenced by the HTML: `/assets/index-DsTlOOtu.js` -> 200, **11,867 B** transferred with `Accept-Encoding: gzip` (26,134 B uncompressed). Headers: `content-encoding: gzip`, `server: nginx/1.27.5`, `cache-control: public, max-age=31536000, immutable`, `via: ... (CloudFront)`.
- Deep route: `GET /some/deep/route` -> 200, 507 B (`index.html`).
- Missing asset: `GET /assets/nope.js` -> 404.

### Who compressed the JS: the platform edge, not the starter's nginx

- The nginx access log (`$body_bytes_sent`, counted after nginx's own compression) shows 26,134 B sent for the JS file, which is its uncompressed size, while the client received 11,867 B with `content-encoding: gzip`. nginx sent it uncompressed; the gzip was applied in front of the container.
- Most likely cause, with the mechanism checked locally on 2026-11-10 with the preact starter (same `nginx.conf` gzip lines, `nginx:1.27-alpine`, Docker 29.5.2): a request with `Accept-Encoding: gzip` gets 7,335 B gzip; the same request plus a `Via:` header gets 17,970 B and no `content-encoding`. nginx's `gzip_proxied` defaults to `off`, which disables compression for requests that arrive through a proxy, and the Back4app edge is one.
- Consequence: `gzip on; gzip_comp_level 6;` in the starter's `nginx.conf` does nothing on this platform as written. The reader still gets gzip (from the edge), at a weaker ratio than nginx level 6 would give.
- The fix would be `gzip_proxied any;` in `nginx.conf`. NOT applied and NOT tested on Back4app.

### Runtime logs

`get_runtime_logs`: 27 lines (nginx entrypoint, nginx notices, access log). First three:

```text
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
```

## Part B3 — wrong port (`port: 3000`, image listens on 8080): corrected by the platform

Same commit, same folder, uploaded again (`Back4App already had every file: nothing to send.`, 4 s, uploadId `98f8249d-5660-46b2-8627-a902edf46187`, same `manifestSha256`).

- `create_web_app` arguments: `name: "mcp-lit-wrongport"`, `port: 3000`, `uploadId`, `manifestSha256`; `builder` not specified.
- appId `95470f79-7cb1-4b7b-85bb-82ea6dbc7221` · deploymentId `523c6e4e-4dc6-4fa8-ae33-972a56addd65`
- `get_web_app` right after creation: `builder: "DOCKERFILE"`, `port: 3000`.
- App `createdAt` 13:01:25.838Z · deployment `createdAt` 13:01:25.983Z -> `READY` (`updatedAt` 13:01:44.016Z) -> **18.0 s**. It did not fail.
- `get_deployment` at READY carried:

```json
"portDetected": { "expected": 3000, "listening": 8080 }
```

- `get_web_app` after READY: `port: 8080`. The setting was rewritten.
- Build log, verbatim:

```text
LAUNCHING CONTAINER...
Using port 3000 (from the Port setting)
App is not listening on port 3000, but on port 8080 — switching to it (auto-detected)
Port 8080 saved to the app settings for future deploys
CHECKING HEALTH...
trying to hit the 8080 port using http
DEPLOYMENT READY
```

- `url`: https://mcplitwrongport-po9ctt5t.back4.app (`urlExpiresAt` 14:01:25.922Z): `GET /` 200, `GET /some/deep/route` 200.
- The detection cost about 3 s (13:01:39.133 -> 13:01:42.134).

## `update_web_app` without confirmationId (dry run, on `mcp-lit`)

Arguments: `appId`, `envVars: [{ key: "LAB_NOTE", value: "x" }]`. Answer:

```json
{
  "status": "CONFIRMATION_REQUIRED",
  "confirmationId": "<not recorded>",
  "expiresAt": "2026-11-10T13:07:58.586Z",
  "action": "UPDATE_APP",
  "summary": {
    "settings": {},
    "envVars": { "added": ["LAB_NOTE"], "changed": [], "removed": [] },
    "newCode": false
  },
  "nextAction": "Nothing changed yet. Show this summary to the person and, if they agree, call update_web_app again with the same arguments and this confirmationId within 5 minutes."
}
```

The summary names the variable and never shows its value. The confirmation was valid for 5 minutes and was NOT sent. `get_web_app` afterwards: `envVarNames: []`, same deployment active: nothing changed.

## Not tested

- The second `update_web_app` call with the confirmationId (it needs the account owner's agreement), so: no data on how long an env-var change takes or whether the URL survives it.
- `gzip_proxied any;` on Back4app.
- A wrong port on an image that listens on no port at all, or on several.
- Not tested anywhere in this series: `rollback_web_app`, `confirm_action`, custom domains, env vars at creation, GitHub as the code source through MCP, behaviour after `urlExpiresAt`, a temporary (not signed in) account, the `files` upload path for clients with no terminal.

## Compression fix verified on the public URL (2026-11-10)

- Cause: requests reach the container through the platform's CDN with a `Via` header, and nginx's `gzip_proxied` is off by default, so nginx answered them uncompressed and the CDN compressed the response itself.
- Fix: `gzip_proxied any;` in nginx.conf (commit f81532b), redeployed to the same app with `deploy_web_app`.
- Main JS transferred from the public URL (`Accept-Encoding: gzip`): before 11,859 B, after 8,763 B. The "after" figure equals this bundle compressed by nginx at gzip_comp_level 6.
