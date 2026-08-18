# llama.cpp — nudge fork (Strix Halo + reasoning budget steering)

A llama.cpp fork for AMD Strix Halo (gfx1151) that adds **mid-thinking budget
steering** for hybrid-reasoning models (Qwen3.5/3.6/3.8 lineage): instead of only
a hard cutoff when the thinking budget expires, the server can inject first-person
nudge messages into the model's reasoning stream at budget fractions, at clean
paragraph boundaries, so the model converges on its own.

Built on top of:
- [Nathanw1014/llama.cpp `strix-halo-vulkan`](https://github.com/Nathanw1014/llama.cpp/tree/strix-halo-vulkan)
  — Strix Halo Vulkan backend work (thanks!)
- [ggml-org/llama.cpp PR #25961](https://github.com/ggml-org/llama.cpp/pull/25961)
  by laurencehardman — the reasoning-budget soft-warning mechanism this fork ports
  and extends
- upstream [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp) master

## What this fork adds

**Multiple soft warning points** — up to two nudges before the hard cutoff, each
firing once per thinking block at the next newline boundary:

```
--reasoning-budget-enable
--reasoning-budget 12288
--reasoning-budget-soft-ratio 0.5
--reasoning-budget-soft-message 'I have used about half of my thinking budget. Let me focus on the main line of reasoning and stop exploring side branches.'
--reasoning-budget-soft2-ratio 0.75
--reasoning-budget-soft2-message 'I have spent most of my thinking budget. I should stop exploring new approaches and converge on the final answer now.'
--reasoning-budget-message 'Considering the limited time by the user, I have to give the solution based on the thinking directly now.\n</think>'
--reasoning-budget-grace-tokens 256
```

**Intro-mode** — the budget announcement normally re-fires on every thinking block,
which spams multi-turn agent histories. `--reasoning-budget-intro-mode once`
suppresses it when its text already appears in the conversation prompt (deduped at
slot launch, matching on decoded text so BPE re-tokenization doesn't defeat it).

**Reasoning-token telemetry** — responses now report the reasoning-channel token
count: `usage.reasoning` (what Anthropic-style harnesses read) and
`completion_tokens_details.reasoning_tokens` (OpenAI-style). llama.cpp previously
folded reasoning silently into `completion_tokens`.

Soft messages are injected in the model's own voice as forced tokens (logit
forcing, speculative-decoding safe); the model keeps reasoning after them. The
hard cutoff waits for a paragraph boundary (bounded by the grace window) and the
hard message must include the closing tag (e.g. `\n</think>`).

## Build (Linux, Vulkan)

```
cmake -B build -DGGML_VULKAN=ON -DCMAKE_BUILD_TYPE=Release -DGGML_NATIVE=OFF \
  -DLLAMA_BUILD_SERVER=ON -DLLAMA_BUILD_WEBUI=OFF
cmake --build build -j (nproc) --target llama-server
LD_LIBRARY_PATH=build/bin ./build/bin/llama-server ...
```

HIP builds work with the usual `-DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1151`.

Everything else is stock llama.cpp — see [README-upstream.md](README-upstream.md).
Unit tests for the budget sampler: `--target test-reasoning-budget`.

## License

MIT, as upstream. The upstream readme and license are preserved in this repo.
