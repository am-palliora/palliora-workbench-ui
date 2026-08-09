# Connecting to palliora-adapter

This fork of [Open WebUI](https://github.com/open-webui/open-webui) exists
specifically to run against
[`palliora-adapter`](https://github.com/am-palliora/palliora-adapter), an
OpenAI-API-compatible HTTP backend that runs chat completions as
threshold-encrypted compute agreements on the Palliora blockchain.

**This is otherwise an unmodified fork.** Connecting to
`palliora-adapter` needed zero source changes here — Open WebUI's own
backend makes the request to whatever OpenAI-compatible API you point it
at server-side, not from the browser, so `palliora-adapter`'s lack of any
CORS configuration is a non-issue. The only additions in this fork are
this file and `docker-compose.palliora.yml`, an optional overlay that
sets the two env vars below — everything else is upstream Open WebUI,
kept mergeable with `upstream/main` going forward.

## 1. Start palliora-adapter

In a sibling checkout of `palliora-adapter` (see that repo's own
`infra/docker-compose.yml`, proven working):

```bash
cd path/to/palliora-adapter/infra
docker compose up -d
curl http://localhost:3000/health   # expect {"status":"ok"}
```

## 2. Get a bearer token

`palliora-adapter`'s only real auth path today is a Magic email-OTP
login flow, and no real Magic credentials exist for this project yet
(see that repo's `docs/agents/architecture.md`). For now, use its
dev-token script instead — it onboards (and funds, on first run) a real
identity against the actually-running server and prints a usable bearer
token:

```bash
docker compose exec api npx tsx apps/api/scripts/issue-dev-token.ts
```

Save the printed token — that's your `OPENAI_API_KEY` below. Re-running
the script is safe (it reuses the same identity without re-funding it)
and just issues a fresh token each time.

## 3. Start Open WebUI, pointed at palliora-adapter

```bash
PALLIORA_ADAPTER_TOKEN=<token from step 2> OPEN_WEBUI_PORT=8081 \
  docker compose -f docker-compose.yaml -f docker-compose.palliora.yml up -d
```

`OPEN_WEBUI_PORT=8081` avoids a collision with `palliora-adapter`'s own
compose stack, which already binds host port 3000.

Open `http://localhost:8081`. The model dropdown should list the 4
models `palliora-adapter` currently supports (`gemma3:1b`,
`deepseek-r1:1.5b`, `llama3.2:1b`, `qwen3:0.6b`) — anything else will
fail on-chain with a fast 404 from the guardian's Ollama backend.

Prefer configuring by hand instead of the overlay file? Settings → Admin
→ Connections → add an OpenAI API connection with:

- **API Base URL**: `http://host.docker.internal:3000/v1` (or your
  `palliora-adapter` host's real address)
- **API Key**: the token from step 2

## What doesn't (and doesn't need to) go through palliora-adapter

- **Document upload / RAG**: Open WebUI's own document-upload feature
  uses its own local embeddings by default, not the connected
  provider's Files API — so it never touches `palliora-adapter`'s
  differently-shaped `/v1/files` endpoint. Nothing to configure either
  way for basic use.
- **Regenerate**: Open WebUI's own regenerate button just resends a
  normal `/v1/chat/completions` call, not `palliora-adapter`'s dedicated
  `/v1/chat/completions/regenerate` endpoint. It still works correctly —
  `palliora-adapter` lazily reuses the same on-chain agreement either
  way — it just doesn't exercise that specific optimization endpoint.

## Known limitations of this integration

- The dev-token flow above is exactly that — a **dev/demo convenience**,
  not real authentication. There's currently no way for a real end user
  to sign in through this UI and get their own `palliora-adapter`
  identity; everyone using a given token shares one on-chain identity.
  Real Magic-based login, if wired up on the `palliora-adapter` side
  later, would still need a small custom Open WebUI auth plugin to
  actually surface in this UI — not attempted here.
- `palliora-adapter`'s streaming responses are simulated (the guardian
  returns one complete result; the adapter chunks it into an SSE stream
  with a small artificial delay) rather than true token-by-token
  generation. It renders correctly in the UI, just isn't "real" streaming
  under the hood.
