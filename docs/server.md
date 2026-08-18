# Server

The server target is OpenAI-compatible local serving, with Anthropic
Messages compatibility available for coding harness smoke tests.

```bash
mtplx serve --host 127.0.0.1 --port 8000 --no-stats-footer
```

See [Concurrency modes](concurrency.md) for scheduler selection, ownership
rules, and model/backend-specific implementations.

Endpoints:

- `GET /health`
- `GET /metrics`
- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/completions`
- `POST /v1/messages`
- `GET /admin/sessions`
- `POST /admin/cache/clear`

## Sharing on your network (other devices, Parallels/VM guests)

The default bind is `127.0.0.1`: only this Mac can connect. To reach MTPLX
from other devices — or from a Windows VM in Parallels/VMware/UTM on the same
Mac, which arrives over the virtual network rather than loopback — bind all
interfaces. Non-localhost binds require an API key; if the key file doesn't
exist yet it is created with a fresh key and printed once:

```bash
mtplx serve --host 0.0.0.0 --port 8000 --api-key-file ~/.mtplx/api-key
```

Startup prints a `Network OpenAI API Base URL` (your Mac's LAN address, e.g.
`http://192.168.1.20:8000/v1`). On the other machine, point any
OpenAI-compatible client at that base URL with the printed key as the API
key (sent as a Bearer token). Parallels shared networking reaches the Mac's
LAN address directly; macOS may ask once to allow incoming connections —
click Allow. To pass the key inline instead of a file:

```bash
mtplx serve --host 0.0.0.0 --port 8000 --api-key "$MTPLX_API_KEY"
```

For Open WebUI, set the OpenAI-compatible base URL to:

```text
http://127.0.0.1:8000/v1
```

For Dockerized Open WebUI, the container must use the host gateway URL, not the host's loopback URL:

```bash
mtplx openwebui docker-command
```

That helper disables Open WebUI's Ollama probe and background task generations
so MTPLX only serves visible chat turns by default.

For Anthropic Messages-compatible clients, point the client base URL at the
bare server root — no `/v1` suffix:

```text
http://127.0.0.1:8000
```

The Anthropic SDK appends `/v1/messages` itself; a `/v1` base would request
`/v1/v1/messages`, which is not a registered route.

## Android Studio

Android Studio's external model provider should use the OpenAI-compatible URL
schema and the MTPLX `/v1` base URL:

```text
URL: http://127.0.0.1:8008/v1
URL schema: OpenAI-compatible
API key: leave blank for localhost unless MTPLX was started with --api-key
```

Refresh the model list after the server starts. MTPLX supports the OpenAI chat,
streaming, and tool-call request shape used by local coding clients; Gemini-only
proprietary behavior is outside that compatibility contract. To verify a local
setup, run:

```bash
mtplx doctor android-studio --port 8008
```

Since 2.5.3 the stats footer only appears on MTPLX-owned surfaces (the app
and the built-in browser chat); API clients such as Open WebUI, Claude Code,
and OpenCode never receive it, so no flag is needed for them.
`--no-stats-footer` still turns it off everywhere, and
`MTPLX_STATS_FOOTER_SCOPE=all` restores the pre-2.5.3 behavior. Metrics
remain available at `/metrics`.

## Transparent agent middleware

Agent middleware is enabled by default for compatibility with existing MTPLX
agent integrations. For a transparent OpenAI-compatible agent bridge (for
example OpenCode with a Qwen template that supports native tools), start the
forked server with:

```bash
uv run mtplx serve --host 127.0.0.1 --port 8006 \
  --agent-middleware off \
  --chat-template-profile froggeric_v22_1 \
  --reasoning-mode on \
  --reasoning-effort medium \
  --reasoning-parser qwen3
```

In this mode MTPLX preserves the incoming system, developer, user, assistant,
and tool history (with the protocol-required developer-to-system conversion for
Qwen templates) and passes the complete incoming `tools` array to the native
chat template. `froggeric_v22_1` is the bundled Qwen 3.8 template with native
tool support and request-selectable `low`, `medium`, and `xhigh` reasoning.
It does not canonicalize or compact the transcript, replace the tool inventory
with a text contract, or inject MTPLX agent prompts and retry reminders. If a
selected template cannot render native tools, the request fails explicitly
instead of silently dropping them. Session-bank postcommit rewriting and its
history-reconstruction cache reuse are disabled in transparent mode. For long
text prompts, MTPLX does retain a clone-only KV snapshot at the exact rendered
token boundary: a later request can reuse only the literal rendered-token
prefix it shares with the new request, with matching model/template policy
identity. This avoids a full re-prefill of an unchanged OpenCode prologue
without rewriting, compacting, or approximating the client transcript. Image
prompts currently cold-prefill in transparent mode.

### Transparent-mode reasoning effort

With `--agent-middleware off`, an OpenAI-compatible caller owns only its
reasoning controls; MTPLX still owns sampling, MTP depth, and generation mode.
This applies to OpenCode and to other callers without an opt-in header. Send an
effort either at top level or in Qwen's vLLM/SGLang-style template kwargs:

```json
{"reasoning_effort": "xhigh"}
```

```json
{"chat_template_kwargs": {"reasoning_effort": "low"}}
```

The top-level field wins when both are present. Qwen aliases `minimal` to
`low`, and OpenCode's `high` to `xhigh`; `none` disables thinking. Combining
`reasoning_effort: "none"` with `enable_thinking: true` is rejected as a
conflicting request. The response's `mtplx_stats` records the original request
value/source and the resolved effort.

`froggeric_v21_3` supports on/off thinking only; use `froggeric_v22_1` for
effort switching. A custom Qwen 3.8 template must render distinct `low` and
`xhigh` prompts or an explicit non-default effort request fails with a clear
400 instead of being silently ignored. Legacy `--agent-middleware on` keeps its
server-owned reasoning policy unchanged.
