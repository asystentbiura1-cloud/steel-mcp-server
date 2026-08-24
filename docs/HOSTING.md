# Running the hosted endpoint

The stdio server in the README serves one person on one machine. `dist/hosted.js` serves the same
tools to many callers over Streamable HTTP, each bringing their own Steel key. This is how a team
shares one deployment, and how a client that cannot launch a local process reaches Steel.

## Dependencies

The hosted entrypoint needs two packages a default install deliberately leaves out, so that a
desktop or `npx` user never carries the hosted stack:

```bash
npm install ioredis @modelcontextprotocol/node
# and, only if you want OTLP tracing:
npm install @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http
```

They are declared as optional `peerDependencies`. A source checkout already has all four, and the
Docker image installs them itself.

## The endpoint

`node dist/hosted.js` (or `npm run start:hosted`) serves the tools at `POST /mcp`. Every caller
brings their own Steel key, as an `Authorization: Bearer` header or an `?apiKey=` query parameter
for hosts that cannot set headers; a handle minted by one request is usable only by the credential
that minted it. `GET /healthz` answers a load-balancer probe without consulting the Host allowlist.
`GET` and `DELETE` on `/mcp` answer `405`, as the 2026-07-28 spec requires.

| Variable | Default | Meaning |
|---|---|---|
| `STEEL_ALLOWED_HOSTS` | | **Required.** Comma-separated hostnames this endpoint answers on. Without it, DNS rebinding has nothing to stop it, so the server refuses to start |
| `STEEL_ALLOWED_ORIGINS` | | Comma-separated browser origins allowed to call it. Empty rejects every request that carries an `Origin`; requests without one still pass |
| `PORT` | `8080` | Port to bind. `0` asks the OS for a free one |
| `HOST` | `0.0.0.0` | Address to bind |
| `REDIS_URL` | | Shares handle records between replicas, so any replica can serve a handle another minted. Without it, records stay in the process, which is correct for exactly one replica |
| `REDIS_KEY_PREFIX` | `steel-mcp` | Key namespace, so one store can hold more than one deployment |
| `STEEL_REQUEST_STATE_SECRET` | per-process | HMAC key for human-in-the-loop handoff state. **Required with `REDIS_URL`**, and identical on every replica: without it a retried handoff lands on a replica that cannot verify state another one minted, after the person has already signed in. Generate with `openssl rand -base64 32` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | | Any standard `OTEL_*` variable turns on OTLP tracing; `OTEL_SERVICE_NAME` defaults to `steel-mcp`. Unset means no exporter is loaded at all |

The stdio variables from the README (`STEEL_BASE_URL`, `STEEL_PROFILE`, the timeouts,
`STEEL_MAX_SESSIONS`) apply here too. `STEEL_API_KEY` does not: the server never holds a Steel key
of its own, so it is the deployment's job to terminate TLS in front of it. Hosted logs are
structured JSON on stdout, and credentials are redacted before anything reaches them.

## Deploying with compose

`docker-compose.yaml` deploys the endpoint on any compose host, Coolify included:

```bash
STEEL_ALLOWED_HOSTS=mcp.example.com docker compose up -d --wait
```

It builds the image from this repository and names `dist/hosted.js`, because the image's own
default command is the stdio server, which binds no port; a platform that cannot override the
command would otherwise deploy a container that never turns healthy. Point the proxy at port 8080
rather than whatever it defaults to, and set `STEEL_ALLOWED_HOSTS` to the public hostname the proxy
forwards: any other `Host` is refused, while `/healthz` answers regardless so a probe on an IP still
passes.

## Connecting a client

Claude Code speaks Streamable HTTP itself:

```bash
claude mcp add steel --transport http https://mcp.example.com/mcp \
  --header "Authorization: Bearer $STEEL_API_KEY"
```

Claude Desktop does not. Its `claude_desktop_config.json` launches a program and speaks JSON-RPC over
that program's stdin and stdout, so a remote endpoint needs a local bridge:

```json
{
  "mcpServers": {
    "steel": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@0.1.38",
        "https://mcp.example.com/mcp",
        "--header",
        "Authorization:${STEEL_AUTH_HEADER}"
      ],
      "env": { "STEEL_AUTH_HEADER": "Bearer <your-steel-api-key>" }
    }
  }
}
```

Two details in that snippet look like mistakes and are not. The header has **no space** after the
colon, and the credential sits in `env` rather than inline, because some hosts do not escape a space
inside `args` and mangle the value. Prefer a header over the `?apiKey=` query parameter wherever the
client can set one: the query form is there for clients that cannot, and any proxy in front of this
server logs a query string before the server is reached.
