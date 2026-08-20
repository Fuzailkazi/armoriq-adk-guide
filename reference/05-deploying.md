# 5. Deploying

Two services, deployed separately. The MCP server first, because you need its URL.

## Why two services

| | |
|---|---|
| **The MCP server** | Must be publicly reachable over HTTPS — the ArmorIQ platform calls it directly. Holds no secrets beyond its own upstream credentials. |
| **The agent** | Holds your ArmorIQ key and your model key. Needs no inbound access unless you are exposing a chat endpoint. |

Different exposure, different scaling, different deploy cadence. Keep them apart.

## The packaging mistake that stops a container booting

If you run TypeScript directly with `tsx` rather than compiling first — a
reasonable choice for a small service — then **`tsx` is a runtime dependency**.

```json
{
  "dependencies": { "tsx": "^4.19.2" },
  "devDependencies": { "typescript": "^5.7.2" }
}
```

With `tsx` in `devDependencies`, any `npm install --omit=dev` — which is what
Docker, Render and Cloud Run all do in production — produces a deploy with no
`tsx`, and the service cannot start. The failure arrives at runtime, after a
successful build, which makes it maddening to diagnose.

Verify it before you deploy:

```bash
mkdir /tmp/prodtest && cd /tmp/prodtest
cp -r /path/to/service/{package.json,package-lock.json,src} .
npm install --omit=dev
npm start
```

If that boots, your deploy will boot.

If you would rather compile properly, add a `tsc` build step and run
`node dist/index.js` — then `tsx` and `typescript` are both dev-only, as normal.

## Render

### One repo per service

`render.yaml` at the repo root, no `rootDir` needed:

```yaml
services:
  - name: ops-mcp
    type: web
    runtime: node
    plan: free
    region: oregon
    buildCommand: npm install
    startCommand: npm start
    healthCheckPath: /health
```

Then **New > Blueprint** and point Render at the repo.

### Both services in one repo

Set `rootDir` per service, and add a build filter so each rebuilds only when its
own folder changes:

```yaml
services:
  - name: ops-mcp
    type: web
    runtime: node
    rootDir: ./mcp-server
    buildCommand: npm install
    startCommand: npm start
    healthCheckPath: /health
    buildFilter:
      paths: [mcp-server/**]

  - name: ops-copilot
    type: web
    runtime: node
    rootDir: ./agent
    buildCommand: npm install
    startCommand: npm start
    healthCheckPath: /health
    buildFilter:
      paths: [agent/**]
```

Render runs every command relative to `rootDir`, so there is no `cd` anywhere. The
blueprint must live at the **repo root** — Render will not find it in a
subdirectory.

Within a single blueprint you can also wire one service to another, so the agent
learns the MCP's hostname without you copying a URL:

```yaml
    envVars:
      - key: MCP_HOST
        fromService:
          name: ops-mcp
          type: web
          property: host
```

Across separate repos that does not apply — set `MCP_URL` by hand instead.

### Free plan: the thing that will embarrass you

Free services **spin down when idle**. The first request after a quiet period
wakes the MCP server and can take about thirty seconds. From the agent's side that
looks exactly like a hang, and it will happen the moment you present.

Use a paid instance for anything you are demonstrating live.

## Cloud Run, Fly, and anything Docker

```bash
gcloud run deploy ops-mcp --source . --region us-central1 --allow-unauthenticated

gcloud run deploy ops-copilot --source . --region us-central1 \
  --set-env-vars MCP_URL=https://ops-mcp-xxxx.run.app/mcp
```

Read `PORT` from the environment — every platform sets it — and bind to all
interfaces, which Express does by default.

## Secrets

Set `ARMORIQ_API_KEY` and your model key as **secrets**, not plain environment
variables. Use `sync: false` in a Render blueprint so the value is prompted for
rather than read from a committed file:

```yaml
    envVars:
      - key: ARMORIQ_API_KEY
        sync: false
```

Never commit a key. If one lands in a repo or a chat transcript, rotate it —
deleting the commit is not enough, because it is already in clones and caches.

## Register after deploying

Both halves have to be registered on platform.armoriq.ai, and both names have to
match your environment variables exactly:

| Register | As | Env var |
|---|---|---|
| The MCP server URL, with `/mcp` on the end | `ops-mcp` | `ARMORIQ_MCP_NAME` |
| The agent | `ops-copilot` | `ARMORIQ_AGENT_NAME` |

A mismatch does not error. It silently means no policy applies, which surfaces
much later as "my policies are not firing".

## Verify the deployment

```bash
curl https://ops-mcp.onrender.com/health

curl -s -X POST https://ops-mcp.onrender.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'

curl -X POST https://ops-copilot.onrender.com/ask \
  -H 'Content-Type: application/json' \
  -d '{"question": "look up acme@corp.com", "user": "support-t1@example.com"}'
```

Then run your setup check against production and confirm the registered names
match.

---

Next: **[6. Troubleshooting](./06-troubleshooting.md)**
