# almide/llm

Streaming clients for LLM providers, for [Almide](https://github.com/almide/almide): the OpenAI-compatible `/chat/completions` API and the Anthropic Messages API. Each call streams the reply, hands every text fragment to a callback as it arrives, and returns the whole reply as one JSON string with the same shape for both providers.

Pure Almide, built on the stdlib's `http.request_stream` and `http.request_stream_with_limits`. No native code.

## Install

```bash
almide add almide/llm@v0.1.0
```

This adds the dependency to your `almide.toml`:

```toml
[dependencies]
llm = { git = "https://github.com/almide/llm", tag = "v0.1.0" }
```

## Usage

```almide
import llm

effect fn main() -> Unit = {
  let body = "{\"model\":\"gpt-4o-mini\",\"stream\":true,\"messages\":[{\"role\":\"user\",\"content\":\"Hi\"}]}"
  let reply = llm.openai_streaming_call("https://api.openai.com/v1", "sk-...", body, (delta) => println(delta))!
  println(reply)
}
```

```almide
import llm

effect fn main() -> Unit = {
  let body = "{\"model\":\"claude-sonnet-4-5\",\"max_tokens\":1024,\"stream\":true,\"messages\":[{\"role\":\"user\",\"content\":\"Hi\"}]}"
  // base_url defaults to https://api.anthropic.com
  let reply = llm.anthropic_streaming_call("sk-ant-...", body, (delta) => println(delta))!
  // a proxy or a compatible server: pass base_url, positionally or by name
  let proxied = llm.anthropic_streaming_call("key", body, (delta) => println(delta), base_url: "http://127.0.0.1:8080")!
  println(reply + proxied)
}
```

`body_json` is sent as is and must already ask for a stream (`"stream": true`). Parse the answer with `json.parse`.

### Per-call limits

The `_with_limits` twins take the stdlib's `HttpLimits` record, in milliseconds, 0 meaning no limit: `total_ms` bounds the whole call (connecting, the first byte and the body all count), `idle_ms` the longest silence between bytes.

```almide
import llm

effect fn main() -> Unit = {
  let body = "{\"model\":\"gpt-4o-mini\",\"stream\":true,\"messages\":[]}"
  let limits = { total_ms: 60000, idle_ms: 10000 }
  match llm.openai_streaming_call_with_limits("http://127.0.0.1:8080/v1", "key", body, limits, (delta) => println(delta)) {
    ok(reply) => println(reply),
    err(e) => println(e),   // e.g. "request timeout: idle_ms 10000 exceeded"
  }
}
```

## The answer

Both providers answer the same LLM-response JSON:

```json
{"content":"...","model":"...","usage":{"prompt_tokens":0,"completion_tokens":0,"total_tokens":0},"finish_reason":"...","tool_calls":[{"id":"...","name":"...","arguments":"..."}]}
```

- `content` is the concatenated text deltas; each non-empty one also went to the callback.
- `tool_calls[i].arguments` is the concatenated argument fragments, as a string of JSON.
- `total_tokens` is the reported total, or prompt + completion when none was reported (Anthropic never reports one). A Float count is truncated.
- OpenAI: `data: [DONE]` ends the stream; nothing after it is applied. Payloads that are not JSON are skipped.
- Anthropic: `input_tokens` / `output_tokens` become `prompt_tokens` / `completion_tokens`, `stop_reason` becomes `finish_reason`. A `tool_use` block is the tool call at the block's `index`, so a text block before it leaves an empty call (`{"id":"","name":"","arguments":""}`) in its slot. An `error` event goes to the callback as `[anthropic stream error] <message>` and is not added to `content`.

A non-2xx status or a transport failure is the err of the underlying stdlib client (`HTTP 404: Not Found: <body>`, `could not connect to ...`). A fired limit is `request timeout: total_ms <n> exceeded` or `request timeout: idle_ms <n> exceeded`.

## API

| Function | Signature |
|---|---|
| `llm.openai_streaming_call` | `effect (base_url: String, api_key: String, body_json: String, on_text_delta: (String) -> Unit) -> String` |
| `llm.openai_streaming_call_with_limits` | `effect (base_url: String, api_key: String, body_json: String, limits: HttpLimits, on_text_delta: (String) -> Unit) -> String` |
| `llm.anthropic_streaming_call` | `effect (api_key: String, body_json: String, on_text_delta: (String) -> Unit, base_url: String = "https://api.anthropic.com") -> String` |
| `llm.anthropic_streaming_call_with_limits` | `effect (api_key: String, body_json: String, limits: HttpLimits, on_text_delta: (String) -> Unit, base_url: String = "https://api.anthropic.com") -> String` |

OpenAI requests go to `<base_url>/chat/completions` with `Authorization: Bearer <api_key>`; Anthropic requests go to `<base_url>/v1/messages` with `x-api-key` and `anthropic-version: 2023-06-01`. Trailing slashes on `base_url` are dropped.

## Targets

Native (`almide run`, `almide build`) serves all four. `almide run --target wasm` serves the two `_with_limits` twins, which stream through the http call handle; the plain calls stream through `http.request_stream`, which has no wasm host yet, so a wasm program that calls them is refused with E081 at compile time. The stock wasm artifact (`almide build --target wasm`) has no http streaming host at all.

## Migrating from the stdlib (dialect epoch 14)

These four functions were part of the stdlib's `http` module until dialect epoch 14, which removed them with no deprecation window. Add the package, `import llm`, and rename the call:

| Before (stdlib) | After (this package) |
|---|---|
| `http.openai_streaming_call(base_url, key, body, cb)` | `llm.openai_streaming_call(base_url, key, body, cb)` |
| `http.openai_streaming_call_with_limits(base_url, key, body, limits, cb)` | `llm.openai_streaming_call_with_limits(base_url, key, body, limits, cb)` |
| `http.anthropic_streaming_call(key, body, cb)` | `llm.anthropic_streaming_call(key, body, cb)` |
| `http.anthropic_streaming_call_with_limits(key, body, limits, cb)` | `llm.anthropic_streaming_call_with_limits(key, body, limits, cb)` |

```diff
+import llm
 import http   // still needed only if you use other http functions

-let reply = http.anthropic_streaming_call(key, body, (d) => println(d))!
+let reply = llm.anthropic_streaming_call(key, body, (d) => println(d))!
```

The parameters, their order, the answer and the error strings are unchanged. The one addition: the Anthropic calls take an optional trailing `base_url` (default `https://api.anthropic.com`, the URL the stdlib hardcoded), so existing calls migrate by the rename alone. The Anthropic `_with_limits` twin now also runs under `almide run --target wasm`, where the stdlib's native intrinsic was refused.

## License

Dual-licensed under either of [MIT](LICENSE-MIT) or [Apache 2.0](LICENSE-APACHE), at your option.
