# wiggdevin/ruflo-zs — Zero Sum fork notes

This is a private fork of [`ruvnet/ruflo`](https://github.com/ruvnet/ruflo) maintained by Devin Wiggins ([@wiggdevin](https://github.com/wiggdevin)).

## Why fork

Upstream ruflo calls Anthropic / OpenAI APIs directly via `@anthropic-ai/sdk` and `fetch`, which requires per-token API billing. Zero Sum's hard rule is **"never use API billing — only Claude Max subscription."** This fork makes ruflo's LLM calls redirectable so they route through [`zs-anthropic-proxy`](https://github.com/wiggdevin/zs-anthropic-proxy), which translates Anthropic Messages API calls into local `claude` CLI invocations on the macOS keychain OAuth subscription.

## What changed

Every Anthropic call site now honors `ANTHROPIC_BASE_URL` (env). For the proxy, set it to `http://127.0.0.1:11455/v1` (note the `/v1` suffix — see "URL convention" below). Leave unset to keep upstream behavior.

| File | Line(s) | Change | Verified |
| --- | --- | --- | --- |
| `v3/@claude-flow/cli/src/mcp-tools/agent-execute-core.ts` | ~117, ~345 | Replaced two hardcoded `fetch('https://api.anthropic.com/v1/messages', …)` with `${ANTHROPIC_BASE_URL}/messages` | ✅ Step 1 + Step 2 (3-agent parallel) |
| `v3/@claude-flow/cli/src/commands/providers.ts` | ~75 | Connectivity test endpoint now reads from env | ✅ `providers test` PASS |
| `v3/@claude-flow/cli/src/types/optional-modules.d.ts` | end of file | New TS stub for `@ruvector/sona` so the CLI builds without the native binary (which has no macOS-arm64 prebuilt). The runtime import is wrapped in `try/catch`, so the stub is build-time only. | ✅ `tsc` clean |
| `v3/@claude-flow/providers/src/anthropic-provider.ts` | ~140 | `apiUrl` now falls back to `process.env.ANTHROPIC_BASE_URL` before the hardcoded default | ⏭️ Untested in this spike |
| `v3/@claude-flow/mcp/src/sampling.ts` | ~324 | Same fix inside `createAnthropicProvider()` | ⏭️ Untested (MCP sampling flow not exercised) |
| `v2/src/sdk/sdk-config.ts` | ~37 | `baseURL` honors `ANTHROPIC_BASE_URL` | ⏭️ Untested (v2 codebase) |
| `v2/src/api/claude-client.ts` | ~307, ~381 | Hardcoded fallbacks now read env | ⏭️ Untested (v2 codebase) |

Each patch is marked with a `// ZS patch:` comment explaining the redirect intent so they're easy to find on rebase.

## URL convention (read this before debugging 404s)

The patches concatenate `${ANTHROPIC_BASE_URL}/messages` and the upstream default is `'https://api.anthropic.com/v1'` — meaning **`/v1` is expected to live in the base URL, not be appended by the patch**. This differs from the `@anthropic-ai/sdk` convention (SDK base = `https://api.anthropic.com`, SDK appends `/v1/messages` itself).

Implications:
- For ruflo via proxy: `ANTHROPIC_BASE_URL=http://127.0.0.1:11455/v1` ← required
- For raw `@anthropic-ai/sdk` via proxy: `ANTHROPIC_BASE_URL=http://127.0.0.1:11455` ← no `/v1`
- The proxy serves `POST /v1/messages` and `GET /v1/models` — both conventions reach the same endpoints once the URL is fully resolved

If you forget the `/v1` for ruflo, you get a `404 endpoint not implemented` from the proxy's not-found handler — the request lands on `/messages`, which is unrouted.

## What is intentionally NOT changed

- **Embeddings (`v3/@claude-flow/embeddings/src/embedding-service.ts`)** — defaults to OpenAI's embedding endpoint. Not patched yet because the proxy is Anthropic-shape, not OpenAI-shape. Use the existing Transformers.js local embedding provider in ruflo if you want zero-cost embeddings, or set `OPENAI_API_BASE` to a separate local OpenAI proxy.
- **Plugin-internal calls** — none found. Plugins go through the framework's provider manager, which is patched.

## How to use it

1. Run `zs-anthropic-proxy` (default `http://127.0.0.1:11455`).
2. Build the workspace once: `cd v3 && pnpm install && pnpm -r build` (the repo is a pnpm workspace at `v3/`, not a plain npm package — `npm install` fails with `EUNSUPPORTEDPROTOCOL` because some sibling deps use `workspace:*`).
3. Export env before running ruflo:
   ```bash
   export ANTHROPIC_BASE_URL=http://127.0.0.1:11455/v1   # /v1 required — see "URL convention"
   export ANTHROPIC_API_KEY=local-proxy                  # any non-empty placeholder
   ```
4. Pre-warm Claude Max OAuth before parallel work: `claude -p "ping"` (avoids OAuth refresh races).
5. Run ruflo. The CLI exposes the patched code path via two routes:
   - **Direct register-then-execute**: `agent spawn -t coder --name probe`, then `mcp exec --tool agent_execute --params '{"agentId":"<id>","prompt":"..."}'`
   - **Note**: `agent spawn` only registers the agent — it does **not** trigger an LLM call. Use `mcp exec --tool agent_execute` to actually invoke the model. Also note: `-p` on `agent spawn` is `--provider`, not prompt.

## Upstream sync

```bash
git remote add upstream https://github.com/ruvnet/ruflo.git
git fetch upstream
git merge upstream/main    # resolve conflicts on the patched files only
```

The patched files are stable provider plumbing. Expected merge tax: 1–2 hours/month.

## Spike status (as of 2026-05-04)

- ✅ `zs-anthropic-proxy` v0.1 built and smoke-tested (non-streaming + streaming both pass against Claude Max subscription)
- ✅ Fork created, six call sites patched + one TS stub added (seven total, see table)
- ✅ **Step 1 (single-agent probe) — PASS.** `agent_execute` returned exact expected output `RUFLO PROXY OK` via `claude-sonnet-4-6`, 7.6s, zero outbound to api.anthropic.com from ruflo itself.
- ✅ **Step 2 (3-agent parallel swarm) — PASS.** All three concurrent `agent_execute` calls succeeded in 7s wall (vs 21s sequential), per-call latency 6.3–7.0s (tight cluster, no OAuth refresh stall), zero proxy warn/error log entries, zero unhandled stream-json event types.
- ⏭️ **Step 3 — pending.** Real-workflow trial on ZS-TARE recipe pipeline (`~/Desktop/ZS-TARE/`, see `recommendation_engine.py` Steps 1 and 7 for the LLM hot spots). Coordinate one full recipe-generation cycle through ruflo with AgentDB trajectory persistence; document whether the trajectory memory adds value over ZeroForge + the LLM router.
- ⏭️ Day-7 go/no-go gate hangs on Step 3 outcome.

## Related

- Plan: `~/.claude/plans/https-github-com-ruvnet-ruflo-git-take-a-ethereal-muffin.md`
- Proxy: `https://github.com/wiggdevin/zs-anthropic-proxy`
- ZeroForge precedent: `~/tools/zeroforge/` (subscription-mode fork of AutoForge)
