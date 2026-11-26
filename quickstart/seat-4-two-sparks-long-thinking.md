# Seat 4: two Sparks, long thinking

[Quick-start index](../README.md) · [Before you start](before-you-start.md) · [Sampling](sampling.md) · [Use it](use-it.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

> First time? Do the three steps in [Before you start](before-you-start.md), then come back here.

> **Hardware** 2 DGX Sparks linked over RoCE · **Download** 108 GB (`vllm/`), no drafter · **Context** **256K** · **Images** on · **Engine** AEON vLLM Ultimate, tensor-parallel, MTP head

| measured 25 November 2026, recommended sampling | one user | four at once (total) |
|---|---|---|
| Thinking off | 51.3 tok/s | **144.8 tok/s** |
| Thinking on | 43.4 | 100.2 |

This is the serve for whole books in context and for long chains of reasoning. It uses four streams and twice the context window. Speeds average four workloads: chat, reasoning, math and code. Chat at four streams was measured twice, at 93 and 108 tok/s, and the table uses the mean of the two. The thinking-on loop check at 2,048 new tokens was 5 of 5 clean. A 24-case long-think quality suite on this exact serve scored 14 of 24. We have not run the stock model through that suite yet, so read it as a first data point and not as a comparison. Memory stayed above 6.9 GiB free on the head node. Back off if it falls under 5.

Same image, same weights and same two nodes as Seat 3. It drafts with the built-in MTP head, so there is no drafter to download. Fill in the same four values as Seat 3 and run this on both nodes, with `RANK=0` on the head node and `RANK=1` on the other:

```sh
IMG=ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-23-v0.33.0-omni-dflash2
MODEL=/models/aeon-xs/vllm
HEAD=<head-node-ip>   # head node address on the fast link
MYIP=<this-node-ip>   # this node's address on the fast link
RANK=0                # 1 on the second node
IF=<roce-if>
HCA=<roce-hca>        # comma-separated HCA names for your link
HEADLESS=$([ $RANK = 0 ] || echo --headless)

docker run -d --name aeon-xs --network host --ipc host --shm-size 16g --gpus all \
  --device /dev/infiniband --ulimit memlock=-1 \
  -e NCCL_SOCKET_IFNAME==$IF -e GLOO_SOCKET_IFNAME=$IF -e TP_SOCKET_IFNAME=$IF \
  -e NCCL_IB_HCA==$HCA -e NCCL_IB_GID_INDEX=3 -e NCCL_IB_DISABLE=0 \
  -e NCCL_IB_ROCE_VERSION_NUM=2 -e NCCL_IB_ADDR_FAMILY=AF_INET \
  -e VLLM_HOST_IP=$MYIP -v $MODEL:/model:ro \
  --entrypoint vllm $IMG serve /model --trust-remote-code --served-model-name aeon-xs \
  --tensor-parallel-size 2 --nnodes 2 --node-rank $RANK \
  --master-addr $HEAD --master-port 29531 --distributed-executor-backend mp \
  --limit-mm-per-prompt '{"image":2,"video":0}' --host 0.0.0.0 --port 8000 \
  --reasoning-parser qwen3 --tool-call-parser qwen3_coder --enable-auto-tool-choice \
  --enable-prefix-caching \
  --max-model-len 262144 --max-num-seqs 4 --gpu-memory-utilization 0.80 \
  --max-num-batched-tokens 16384 \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}' \
  --override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}' \
  $HEADLESS
```

Against Seat 3 only three things differ: `--max-model-len 262144`, `--max-num-seqs 4`, and the MTP head in place of DFlash2, which is why there is no `/draft` mount. Wait and check it exactly as in Seat 3.

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
