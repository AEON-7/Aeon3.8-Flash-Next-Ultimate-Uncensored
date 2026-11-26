# Use it: addresses, Python, network

[Quick-start index](../README.md) · [Before you start](before-you-start.md) · [Sampling](sampling.md) · [Use it](use-it.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

Every seat speaks the OpenAI chat API. Point any OpenAI-compatible app or library at the base URL and use the model name `aeon-xs`. The recipes set no API key. If an app insists on one, type anything.

| Seat | Base URL |
|---|---|
| 1 | `http://127.0.0.1:8300/v1`, reachable from the Spark itself only, as launched |
| 2 | `http://<spark-ip>:8000/v1` |
| 3 and 4 | `http://<head-node-ip>:8000/v1` |
| 5 | `http://<balancer-ip>:8080/v1` |

From Python, after `pip install openai`:

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8000/v1", api_key="none")  # Seat 1: port 8300
reply = client.chat.completions.create(
    model="aeon-xs",
    messages=[{"role": "user", "content": "Explain what a mixture-of-experts model is in three sentences."}],
    temperature=0.6, top_p=0.95, presence_penalty=0.5,
    extra_body={"top_k": 20, "repetition_penalty": 1.0,
                "chat_template_kwargs": {"enable_thinking": False}},
)
print(reply.choices[0].message.content)
```

Images go in as `image_url` parts of a message, in the usual OpenAI format.

**Mind the network.** Seats 2 to 5 listen on every network interface with no password, so anyone who can reach the port can use the model. Keep them on a network you trust. Seat 1 listens on the Spark alone.

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
