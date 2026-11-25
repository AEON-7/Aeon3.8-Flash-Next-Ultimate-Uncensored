# Sampling settings, for every seat

[Quick-start index](../README.md) · [Sampling settings](sampling.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

Use these values for chat and for long reasoning. They are the settings that stopped the thinking-mode loops we saw at temperature 0. Keep `repetition_penalty` at 1.0, because values above it break tool-call XML.

```
temperature 0.6
top_p 0.95
top_k 20
presence_penalty 0.5
repetition_penalty 1.0
```

On AEON vLLM Ultimate, set them as the server default, so clients that send only `temperature` still get the rest:

```
--override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}'
```

On Strata, send the same fields with each chat-completions request. On AEON vLLM Ultimate, leave prefix caching on.

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
