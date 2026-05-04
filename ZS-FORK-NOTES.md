# wiggdevin/ruflo-zs — Zero Sum fork notes

This is a private fork of [`ruvnet/ruflo`](https://github.com/ruvnet/ruflo) maintained by Devin Wiggins ([@wiggdevin](https://github.com/wiggdevin)).

## Why fork

Upstream ruflo calls Anthropic / OpenAI APIs directly via `@anthropic-ai/sdk` and `fetch`, which requires per-token API billing. Zero Sum's hard rule is **"never use API billing — only Claude Max subscription."** This fork makes ruflo's LLM calls redirectable so they route through [`zs-anthropic-proxy`](https://github.com/wiggdevin/zs-anthropic-proxy), which translates Anthropic Messages API calls into local `claude` CLI invocations on the macOS keychain OAuth subscription.

## What changed

Every Anthropic call site now honors `ANTHROPIC_BASE_URL` (env). Set it to `http://127.0.0.1:11455` to route through the proxy; leave it unset to keep upstream behavior.

| File | Line(s) | Change |
| --- | --- | --- |
| `v3/@claude-flow/providers/src/anthropic-provider.ts` | ~140 | `apiUrl` now falls back to `process.env.ANTHROPIC_BASE_URL` before the hardcoded default |
| `v3/@claude-flow/cli/src/mcp-tools/agent-execute-core.ts` | ~117, ~345 | Replaced two hardcoded `fetch('https://api.anthropic.com/v1/messages', …)` with `${ANTHROPIC_BASE_URL}/messages` |
| `v3/@claude-flow/mcp/src/sampling.ts` | ~324 | Same fix inside `createAnthropicProvider()` |
| `v3/@claude-flow/cli/src/commands/providers.ts` | ~75 | Connectivity test endpoint now reads from env |
| `v2/src/sdk/sdk-config.ts` | ~37 | `baseURL` honors `ANTHROPIC_BASE_URL` |
| `v2/src/api/claude-client.ts` | ~307, ~381 | Hardcoded fallbacks now read env |

Each patch is marked with a `// ZS patch:` comment explaining the redirect intent so they're easy to find on rebase.

## What is intentionally NOT changed

- **Embeddings (`v3/@claude-flow/embeddings/src/embedding-service.ts`)** — defaults to OpenAI's embedding endpoint. Not patched yet because the proxy is Anthropic-shape, not OpenAI-shape. Use the existing Transformers.js local embedding provider in ruflo if you want zero-cost embeddings, or set `OPENAI_API_BASE` to a separate local OpenAI proxy.
- **Plugin-internal calls** — none found. Plugins go through the framework's provider manager, which is patched.

## How to use it

1. Run `zs-anthropic-proxy` (default `http://127.0.0.1:11455`).
2. Export env before running ruflo:
   ```bash
   export ANTHROPIC_BASE_URL=http://127.0.0.1:11455
   export ANTHROPIC_API_KEY=local-proxy   # any non-empty placeholder
   ```
3. Run ruflo as normal — agents, swarms, hooks, MCP tools all flow through the proxy.

## Upstream sync

```bash
git remote add upstream https://github.com/ruvnet/ruflo.git
git fetch upstream
git merge upstream/main    # resolve conflicts on the patched files only
```

The patched files are stable provider plumbing. Expected merge tax: 1–2 hours/month.

## Spike status (as of 2026-05-04)

- ✅ `zs-anthropic-proxy` v0.1 built and smoke-tested (non-streaming + streaming both pass against Claude Max subscription)
- ✅ Fork created, six call sites patched
- ⏭️ End-to-end ruflo agent spawn against the proxy (next step)
- ⏭️ Real-world workflow trial on ZS-TARE recipe pipeline (week 1 milestone)

## Related

- Plan: `~/.claude/plans/https-github-com-ruvnet-ruflo-git-take-a-ethereal-muffin.md`
- Proxy: `https://github.com/wiggdevin/zs-anthropic-proxy`
- ZeroForge precedent: `~/tools/zeroforge/` (subscription-mode fork of AutoForge)
