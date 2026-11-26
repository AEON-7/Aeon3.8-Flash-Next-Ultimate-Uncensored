# Seat 3: two Sparks, team server

[Quick-start index](../README.md) · [Sampling settings](sampling.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

> **Hardware** 2 DGX Sparks linked over RoCE · **Download** 108 GB (`vllm/`) plus the DFlash2 drafter, shared over NFS or fetched on both · **Context** 131K · **Images** on, two per request · **Engine** AEON vLLM Ultimate, tensor-parallel, DFlash2

| | one user | 16 at once (total) |
|---|---|---|
| DFlash2, greedy, thinking off (23 Nov) | **65.1 tok/s** | 357.8 tok/s |
| MTP head, greedy, thinking off (23 Nov) | 58.2 | **358.6** |
| MTP head, recommended sampling, thinking off (25 Nov) | 58.4 | 329.9 |
| MTP head, recommended sampling, thinking on (25 Nov) | 47.3 | 279.0 |

*Single-user figures from 23 November are 11-workload means. The 25 November MTP sampling rows, and every 16-at-once figure in this table, average four workloads.*

The default recipe is DFlash2 with 3 draft tokens and 32 sequences. The built-in MTP head, at 16 sequences, is the alternate. A later same-harness ladder the same day, at the recommended sampling and 256 new tokens, compared them on the same two boxes:

| setup | think | 1 | 4 | 8 | 16 | 32 |
|---|---|---|---|---|---|---|
| DFlash2 k=3, 32 sequences | off | 59.7 (accept 0.515) | 141.5 (0.573) | 183.0 (0.514) | 247.9 (0.519, 1/16 loops) | 242.9 (0.501, 3/32 loops) |
| MTP, 16 sequences | off | 35.0 (0.429) | 97.1 (0.593) | 166.3 (0.545) | 230.0 (0.524, 1/16 loops) | not run (cap 16) |
| DFlash2 k=3, 32 sequences | on | 56.8 (0.469) | 131.0 (0.459) | 175.2 (0.449) | 235.1 (0.456, 0 loops) | 312.5 (0.460, 0 loops) |
| MTP, 16 sequences | on | 33.4 (0.402-0.501) | 88.2 | 168.0 | 231.9 (0 loops) | not run |

Figures are aggregate tok/s. Accept is the fraction of drafted tokens the big model kept, about 0.45 to 0.57 on DFlash2. Head-node memory stayed at or above 9.16 GiB free. KV cache on the DFlash2 serve was 1,688,812 tokens (12.88x at 131K), so 32 full 131K contexts still do not fit; the 32-session rows are short sessions. DFlash2 k=3 was faster than MTP at every overlapping concurrency on this short-prompt sampling ladder, especially for one user (59.7 vs 35.0, thinking off).

Sixteen people get about 22 tok/s each, all at once. Short answers start in 0.12 s. A 96K-token prompt, roughly a 250-page book, is read in about 41 s, and the reply then decodes at 59 tok/s, as fast as a short chat. The thinking-on loop check at 2,048 new tokens was 5 of 5 clean.

*Tensor-parallel means each Spark holds half of every layer and the two compute each token together over the fast link.*

Image: `ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-23-v0.33.0-omni-dflash2`. Pin it by digest: `sha256:71c2b82b3b0c97ddca9ef35b279fb745ee0abe29b762a24c8c0771cf930f11ab`. This seat runs on the public image as it is.

**1. Download** the weights on one node and share them over NFS, or on both. Also fetch the DFlash2 drafter, which the default recipe mounts at `/draft`:

```sh
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --include 'vllm/*' --local-dir /models/aeon-xs
huggingface-cli download incoai/Qwen3.8-Flash-Next-DFlash2 --local-dir /models/dflash2
```

**2. Run on both nodes**, with `RANK=0` on the head node and `RANK=1` on the other. Set the interface and HCA names to match your RoCE link.

```sh
IMG=ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-23-v0.33.0-omni-dflash2
MODEL=/models/aeon-xs/vllm
DRAFT=/models/dflash2
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
  -e VLLM_HOST_IP=$MYIP -v $MODEL:/model:ro -v $DRAFT:/draft:ro \
  --entrypoint vllm $IMG serve /model --trust-remote-code --served-model-name aeon-xs \
  --tensor-parallel-size 2 --nnodes 2 --node-rank $RANK \
  --master-addr $HEAD --master-port 29531 --distributed-executor-backend mp \
  --limit-mm-per-prompt '{"image":2,"video":0}' --host 0.0.0.0 --port 8000 \
  --reasoning-parser qwen3 --tool-call-parser qwen3_coder --enable-auto-tool-choice \
  --enable-prefix-caching \
  --max-model-len 131072 --max-num-seqs 32 --gpu-memory-utilization 0.80 \
  --max-num-batched-tokens 16384 \
  --speculative-config '{"method":"dflash","model":"/draft","num_speculative_tokens":3}' \
  --override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}' \
  $HEADLESS
```

- **Images.** `--limit-mm-per-prompt` allows up to two images per request, with video off. Send them as `image_url` parts in the usual OpenAI format. For a text-only server, use `--language-model-only` in its place. The drafter sees only the text of an image prompt, and vLLM says so in its log.
- **`NCCL_SOCKET_IFNAME==$IF` is not a typo.** NCCL reads a leading `=` as an exact interface match, so the value passed in is `=$IF`.
- **Draft tokens: use 3** with either drafter.

**Variant: the built-in MTP head**, for multilingual traffic or when you do not want a separate drafter. Omit the `/draft` mount and use these two lines in place of the matching ones:

```sh
# same docker flags as the default recipe, except:
  --max-num-seqs 16 \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}' \
```

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
