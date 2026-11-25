# Seat 5: four Sparks

[Quick-start index](../README.md) · [Sampling settings](sampling.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

> **Hardware** 4 DGX Sparks, as two linked pairs · **Engine** AEON vLLM Ultimate, two Seat 3 replicas behind any round-robin HTTP proxy

| greedy, thinking off, both replicas loaded at the same moment | total tok/s |
|---|---|
| 8 at once | 313.6 |
| 32 at once | **748.2** |

Run Seat 3 twice, on nodes 1+2 and nodes 3+4, and spread requests across both. The replicas share nothing, so each keeps its full two-Spark speed. That is 23 tok/s each for 32 people, with 0 errors. We measured it with one DFlash2 replica (382 tok/s at 16) and one MTP replica (366 at 16). Use Seat 4's flags on both when the goal is long reasoning.

```nginx
upstream aeon_xs { server <replica-a>:8000; server <replica-b>:8000; }
server { listen 8080; location / { proxy_pass http://aeon_xs; proxy_buffering off; proxy_read_timeout 3600s; } }
```

A single AEON vLLM Ultimate job with tensor-parallel size 4 was not measured on this build and is not a published recipe.

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
