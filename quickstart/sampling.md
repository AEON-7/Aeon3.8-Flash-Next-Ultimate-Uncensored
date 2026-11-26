# Sampling settings, for every seat

[Quick-start index](../README.md) · [Before you start](before-you-start.md) · [Sampling](sampling.md) · [Use it](use-it.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

Sampling settings decide how the model picks each next word. They matter more on this model than on most: with the wrong ones, long reasoning can fall into a loop and tool calls can break. Use these five values everywhere, for chat and for long reasoning. Every figure on this page marked "recommended sampling" was measured with them.

```
temperature 0.6
top_p 0.95
top_k 20
presence_penalty 0.5
repetition_penalty 1.0
```

| Setting | What it does | Why this value |
|---|---|---|
| `temperature 0.6` | How much chance goes into each choice. At 0 the model always takes its single most likely word. | At 0, long thinking fell into loops on our two-Spark serve. At 0.6, every loop check since has come back clean. |
| `top_p 0.95` | Cuts off the unlikely tail. Only the most likely words that together make up 95% of the probability stay in play. | Qwen's own value for this model's thinking mode. |
| `top_k 20` | Never considers more than the 20 most likely words. | Qwen's own value, in both of its modes. |
| `presence_penalty 0.5` | Makes a word a little less likely once it has already appeared. It is the brake on repeating. | Qwen's card allows 0 to 2 and warns that high values can mix languages and cost some quality. 0.5 is the value every recipe here was validated with. |
| `repetition_penalty 1.0` | A second, blunter brake on repeating. 1.0 switches it off. | Leave it off. Values above 1.0 break tool-call XML. |

**What it costs.** These settings decode a little slower than greedy decoding (temperature 0): 55.2 against 60.6 tok/s for one user on Seat 1, and 7 to 28% lower across the seats. Each seat lists both.

**How to send them.**

- **AEON vLLM Ultimate (Seats 2 to 5).** The recipes set all five as the server default with the flag below, so an app that sends only `temperature`, or no settings at all, still gets them. Leave prefix caching on as well.
- **Strata (Seat 1).** There is no server default. Send the five fields with every chat-completions request, as the check command in Seat 1 does.

```
--override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}'
```

**Thinking** is switched per request. Add `"chat_template_kwargs": {"enable_thinking": true}` or `false` to the request body. With thinking on, the model reasons before it answers, the reasoning counts against `max_tokens`, and decoding is slower (45 against 55 tok/s for one user on Seat 1).

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
