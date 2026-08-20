# llama.cpp nudge fork (Strix Halo + reasoning budget steering)

A llama.cpp fork for AMD Strix Halo (gfx1151) that adds mid-thinking budget steering for hybrid-reasoning models in the Qwen3.5/3.6/3.8 lineage. Instead of only applying a hard cutoff when the thinking budget expires, the server can inject first-person nudge messages into the model's reasoning stream at budget fractions and clean paragraph boundaries so the model can converge on its own.

Built on top of:

- [Nathanw1014/llama.cpp `strix-halo-vulkan`](https://github.com/Nathanw1014/llama.cpp/tree/strix-halo-vulkan), including the current Vulkan and DeepSeek V4 sparse-prefill work
- [ggml-org/llama.cpp PR #25961](https://github.com/ggml-org/llama.cpp/pull/25961) by laurencehardman, which introduced the reasoning-budget soft-warning mechanism this fork ports and extends
- Upstream [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) master

## What this fork adds

### Reasoning-budget controls

The server supports a master switch, a hard reasoning cutoff, up to two soft warning points, an optional intro message, an intro-once mode, and a grace window for reaching a paragraph boundary.

```sh
--reasoning-budget-enable
--reasoning-budget 12288
--reasoning-budget-soft-ratio 0.5
--reasoning-budget-soft-message 'I have used about half of my thinking budget. Let me focus on the main line of reasoning and stop exploring side branches.'
--reasoning-budget-soft2-ratio 0.75
--reasoning-budget-soft2-message 'I have spent most of my thinking budget. I should stop exploring new approaches and converge on the final answer now.'
--reasoning-budget-message 'Considering the limited time by the user, I have to give the solution based on the thinking directly now.\n</think>'
--reasoning-budget-intro-mode once
--reasoning-budget-grace-tokens 256
```

Soft messages are injected in the model's own voice as forced tokens and are safe to use with speculative decoding. The hard cutoff waits for a paragraph boundary within the configured grace window. The hard message must include the model's closing tag when required by its template.

`--reasoning-budget-intro-mode once` suppresses the intro message when its decoded text already appears in the conversation prompt. The prompt scan skips multimodal `LLAMA_TOKEN_NULL` placeholders before detokenization.

OpenAI-compatible HTTP requests can set `reasoning_budget_enabled` explicitly. Omitting the field keeps the server or CLI default.

### Reasoning-token telemetry

Responses report the reasoning-channel token count through `usage.reasoning` and `completion_tokens_details.reasoning_tokens` on OpenAI-compatible paths, and `output_tokens_details.reasoning_tokens` on the Anthropic-compatible path.

## Validation

The rebased fork was validated on 2026-08-20:

- Vulkan build completed successfully.
- `test-reasoning-budget` passed all 23 tests and UTF-8 checks.
- `test-chat` passed, including explicit true, explicit false, and omitted HTTP master-switch cases.
- Ornith vision and Qwen3.8 Q6 vision smoke tests returned HTTP 200.
- Qwen3.8 Q6 at 262K context passed all three needle retrieval checks.
- Pi-bench `shift-calendar` passed with an automated grade of 91/100.

The Qwen3.8 Q6 validation server used a 262144-token context, Vulkan flash attention, q8 KV cache, and MTP draft depth 3.

## Build (Linux, Vulkan)

```sh
cmake -B build -DGGML_VULKAN=ON -DCMAKE_BUILD_TYPE=Release -DGGML_NATIVE=OFF -DLLAMA_BUILD_SERVER=ON -DLLAMA_BUILD_WEBUI=OFF
cmake --build build -j "$(nproc)" --target llama-server
LD_LIBRARY_PATH=build/bin ./build/bin/llama-server ...
```

HIP builds work with the usual `-DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1151` configuration.

Everything else is stock llama.cpp. See [README-upstream.md](README-upstream.md) for the upstream documentation. Unit tests for the budget sampler are available through the `test-reasoning-budget` target.

## License

MIT, as upstream. The upstream README and license are preserved in this repository.
