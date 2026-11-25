# Seat 3: two Sparks, team server

[Quick-start index](../README.md) · [Sampling settings](sampling.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

> **Hardware** 2 DGX Sparks linked over RoCE · **Download** 108 GB (`vllm/`), shared over NFS or fetched on both · **Context** 131K · **Images** on, two per request · **Engine** AEON vLLM Ultimate, tensor-parallel

| | one user | 16 at once (total) |
|---|---|---|
| MTP head, greedy, thinking off (23 Nov) | 58.2 tok/s | **358.6 tok/s** |
| MTP head, recommended sampling, thinking off (25 Nov) | 58.4 | 329.9 |
| MTP head, recommended sampling, thinking on (25 Nov) | 47.3 | 279.0 |
| DFlash2, greedy, thinking off (23 Nov) | **65.1** | 357.8 |

*Single-user figures from 23 November are 11-workload means. The 25 November rows, and every 16-at-once figure, average four workloads.*

Sixteen people get about 22 tok/s each, all at once. Short answers start in 0.12 s. A 96K-token prompt, roughly a 250-page book, is read in about 41 s, and the reply then decodes at 59 tok/s, as fast as a short chat. The thinking-on loop check at 2,048 new tokens was 5 of 5 clean.

*Tensor-parallel means each Spark holds half of every layer and the two compute each token together over the fast link.*

Image: `ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-23-v0.33.0-omni-dflash2`. Pin it by digest: `sha256:71c2b82b3b0c97ddca9ef35b279fb745ee0abe29b762a24c8c0771cf930f11ab`. This seat runs on the public image as it is.

**1. Download** on one node and share it over NFS, or on both:

```sh
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --include 'vllm/*' --local-dir /models/aeon-xs
```

**2. Run on both nodes**, with `RANK=0` on the head node and `RANK=1` on the other. Set the interface and HCA names to match your RoCE link.

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
  --max-model-len 131072 --max-num-seqs 16 --gpu-memory-utilization 0.80 \
  --max-num-batched-tokens 16384 \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}' \
  --override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}' \
  $HEADLESS
```

- **Images.** `--limit-mm-per-prompt` allows up to two images per request, with video off. Send them as `image_url` parts in the usual OpenAI format. For a text-only server, use `--language-model-only` in its place. The drafter sees only the text of an image prompt, and vLLM says so in its log.
- **`NCCL_SOCKET_IFNAME==$IF` is not a typo.** NCCL reads a leading `=` as an exact interface match, so the value passed in is `=$IF`.
- **Draft tokens: use 3** with either drafter.

**Variant: DFlash2**, for the fastest single user and the snappiest short answers. Download the drafter, mount it, change one flag:

```sh
huggingface-cli download incoai/Qwen3.8-Flash-Next-DFlash2 --local-dir /models/dflash2
# add:     -v /models/dflash2:/draft:ro
# replace: --speculative-config '{"method":"dflash","model":"/draft","num_speculative_tokens":3}'
```

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
