# Seat 4: two Sparks, long thinking

[Quick-start index](../README.md) · [Sampling settings](sampling.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

> **Hardware** 2 DGX Sparks linked over RoCE · **Download** as Seat 3 · **Context** **256K** · **Images** on · **Engine** AEON vLLM Ultimate, tensor-parallel, MTP head

| measured 25 November 2026, recommended sampling | one user | four at once (total) |
|---|---|---|
| Thinking off | 51.3 tok/s | **144.8 tok/s** |
| Thinking on | 43.4 | 100.2 |

This is the serve for whole books in context and for long chains of reasoning. It gives up twelve of Seat 3's sixteen streams for twice the context window. Speeds average four workloads: chat, reasoning, math and code. Chat at four streams was measured twice, at 93 and 108 tok/s, and the table uses the mean of the two. The thinking-on loop check at 2,048 new tokens was 5 of 5 clean. A 24-case long-think quality suite on this exact serve scored 14 of 24. We have not run the stock model through that suite yet, so read it as a first data point and not as a comparison. Memory stayed above 6.9 GiB free on the head node. Back off if it falls under 5.

Same image, same weights, same two nodes as Seat 3:

```sh
# same docker flags as the throughput recipe, except:
  --enable-prefix-caching \
  --max-model-len 262144 --max-num-seqs 4 --gpu-memory-utilization 0.80 \
  --max-num-batched-tokens 16384 \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}' \
  --override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}' \
  $HEADLESS
```

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
