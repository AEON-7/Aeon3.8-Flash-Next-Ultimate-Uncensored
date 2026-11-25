# AGENTS.md: how to serve Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS

You are an AI agent, and a person has asked you to bring this model up on their hardware. This file is the whole procedure. Work through it in order.

Every command below was copied from a run we measured between 21 and 25 November 2026. Treat the commands as fixed. Your job is to pick the right one for the hardware in front of you, fill in the few values that depend on the site, check the result against the numbers given here, and report honestly.

- Model card and all measurements: <https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS>
- Quick-start index for people: <https://github.com/AEON-7/Aeon3.8-Flash-Next-Ultimate-Uncensored>
- Full-precision master (not what you serve): <https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-Ultimate-Uncensored-BF16>

## 0. Rules that override everything else

1. **Use only a configuration from this file.** If the hardware matches none of them, stop and tell the person. Do not invent flags, and do not try another engine: SGLang and TensorFold cannot load this checkpoint.
2. **Keep the pins.** Strata 0.2.18 (engine 0.1.64) and the AEON vLLM Ultimate image by its digest. Never run a plain `strata update`. Strata 0.2.19 garbles this model's output and 0.2.20 is slower.
3. **Do not attempt Seat 2 (AEON vLLM Ultimate on one Spark).** The public image does not fit on one box. On one Spark, use Seat 1.
4. **Draft tokens are 3 on AEON vLLM Ultimate.** 8 stops the engine at startup. 11 starts and is slower. Do not tune this.
5. **Do not raise `--gpu-memory-utilization`.** A DGX Spark has one memory pool shared by CPU and GPU. The value is 0.80 on two Sparks and 0.74 on one. Higher values stall or kill the box.
6. **Never pass `--moe-backend`.** `flashinfer_b12x` consumed 34 GiB in 20 seconds and the out-of-memory killer took the whole machine. `flashinfer_cutlass` refuses this checkpoint.
7. **Watch free memory.** Read `MemAvailable` in `/proc/meminfo`. Under 5 GiB, back off the load. Under 4 GiB, stop the serve and tell the person.
8. **The machine is not yours.** Do not stop, remove or reconfigure any other container or service to make room. If something else is using the GPU or the memory, say so and ask.
9. **Do not widen network exposure on your own.** Seat 1 listens on `127.0.0.1`. The AEON vLLM Ultimate recipes listen on `0.0.0.0`. Tell the person which, and leave firewalling and any public exposure to them.
10. **This model is uncensored.** It makes no judgement about whether to comply. The person operating it is the safety layer, and the responsibility clause on the model card applies. Do not describe it to them as safe by default.
11. **Report what you measured, not what this file says you should get.** If a number is far off, say so and stop tuning.

## 1. Find out what you are on

Run these and read the output. Do not guess from the hostname.

```sh
nvidia-smi --query-gpu=name --format=csv,noheader     # a DGX Spark prints: NVIDIA GB10
grep -E 'MemTotal|MemAvailable' /proc/meminfo          # a DGX Spark has about 127,500,000 kB in total
df -h .                                                # free disk where the weights will go
docker --version                                       # needed for AEON vLLM Ultimate, not for Strata
uname -m                                               # aarch64 on a DGX Spark
rdma link show                                         # multi-Spark only: the RoCE link between boxes
```

Then choose:

| What you found | Seat | Engine | Context | Sessions |
|---|---|---|---|---|
| 1 DGX Spark | **Seat 1** (section 2) | Strata 0.2.18 + DFlash2 | 131K, images | 4 |
| 2 DGX Sparks joined by a RoCE link, general use | **Seat 3** (section 3) | AEON vLLM Ultimate, tensor-parallel | 131K, images | 16 |
| 2 DGX Sparks, and the person wants more than 131K of context or long reasoning | **Seat 4** (section 4) | AEON vLLM Ultimate, tensor-parallel | 256K, images | 4 |
| 4 DGX Sparks as two linked pairs | **Seat 5** (section 5) | two Seat 3 serves and a balancer | 131K, images | 32 |
| 3 DGX Sparks | Seat 3 on a linked pair, and Seat 1 on the third if wanted | | | |
| Anything else: other GPUs, less memory, no RoCE link between two Sparks | **Stop.** Not validated. Tell the person. | | | |

Ask the person only what you cannot determine: Seat 3 or Seat 4 on two Sparks, and whether the port should be reachable from other machines. Everything else has a default below.

Disk needed: 98 GiB for Seat 1 (the repo root without `vllm/`). 108 GB for Seats 3 to 5 (`vllm/`), on storage both nodes can read. The repo holds both builds of the same weights. Download only the one the seat uses.

## 2. Seat 1: one Spark, Strata

Target: 1 DGX Spark. One person at about 55 tok/s with the default sampling (60 greedy), or four at once sharing about 66 to 73 tok/s. Strata decodes at most 8 streams in parallel, and this recipe runs 4.

**2.1 Download** everything except the `vllm/` folder:

```sh
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --exclude 'vllm/*' --local-dir aeon-xs
cd aeon-xs
```

**2.2 Get the pinned engine**, from the repo root, and check the hashes. If a hash does not match, stop.

```sh
U=https://github.com/Niko1221/Strata/releases/download/v0.2.18
T=strata-engine-0.1.64-linux-aarch64-cuda-cuda13.tar.gz
C=strata-control-0.1.64-linux-aarch64.tar.gz
curl -fLO $U/$T
curl -fLO $U/$C
echo "8e3aaf70a27a26de04bab62e42d151eaa01c33b8c9bdd9839018fd5c54f4778b  $T" | sha256sum -c -
echo "97f47f123d4a5af09a72fe0a3fa43c4a5d0b4f13ddc11d856b0d7f27b64558c4  $C" | sha256sum -c -
mkdir -p strata-engine-0.1.64 && tar -xzf $T -C strata-engine-0.1.64
mkdir -p strata-terminal-0.1.64 && tar -xzf $C -C strata-terminal-0.1.64
ENGINE=$PWD/strata-engine-0.1.64/strata
$ENGINE --version    # strata 0.1.64
```

**2.3 Write the config and start the server.** Run this from the repo root, in the same shell as 2.2 so that `$ENGINE` is set.

```sh
cat > aeon-xs-dflash2.json <<JSON
{
  "exe": "$ENGINE",
  "cwd": "$PWD",
  "tokenizer": "$PWD/serve/tokenizer",
  "model_name": "aeon-xs",
  "env": {"STRATA_PREFILL_NVFP4": "w4a8"},
  "open_browser": false,
  "drafting": "dflash2",
  "parallel": 4,
  "args": ["--pack", "$PWD/pack",
    "--native", "$PWD/dense.gguf", "--native-dense-gguf", "$PWD/dense.gguf",
    "--ple-gguf", "$PWD/ple-nvfp4.gguf",
    "--expert-profile", "$PWD/expert-profile.bin",
    "--expert-cache", "auto", "--prefill", "auto:32768",
    "--max-context", "131072", "--kv", "int8", "--kv-resident", "32768",
    "--mmap-experts", "--conversation-cache-mib", "4096", "--vision"],
  "vision": {"exe": "$PWD/strata-engine-0.1.64/strata-vision",
    "mmproj": "$PWD/mmproj-Aeon3.8-Flash-Next-BF16.gguf",
    "model": "$PWD/dense.gguf", "gpu": true, "max_tokens": 1024},
  "user_args": ["--draft-model", "$PWD/strata/qwen3.8-flash-next-dflash2.strata1.q4_0-selq4.pack",
    "--spec", "6", "--draft-conf", "0.5"]
}
JSON
./strata-terminal-0.1.64/bin/strata serve --config aeon-xs-dflash2.json --host 127.0.0.1 --port 8300 --parallel 4
```

**2.4 Wait** for the line `ready: http://127.0.0.1:8300/v1`, then test from a second shell:

```sh
curl -s http://127.0.0.1:8300/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"aeon-xs","messages":[{"role":"user","content":"Name three prime numbers."}],"max_tokens":64,"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}'
```

Do not change: the engine version, `STRATA_PREFILL_NVFP4`, `--kv int8`, `--max-context 131072`, the drafter flags (`--spec 6 --draft-conf 0.5`), or `--parallel 4`.

You may change: remove `--vision` and the whole `"vision"` section for a text-only server (same text speed). Use `--parallel 1` for a single user (same speed for that user). Change `--host` and `--port` only if the person asks.

Do not raise `--parallel` above 4. Strata never decodes more than 8 streams in parallel, a 32-session launch at 131K does not keep 5 GiB free, and in our runs free memory kept falling from one pass to the next.

Variant, for mostly non-English traffic: use the built-in MTP head in place of DFlash2. In the config set `"drafting": "mtp"`, delete `"user_args"`, and add these flags to `"args"`, with `./mtp/rt` written as a full path like the others:

```sh
  ... --spec 4 --spec-min-p 0.5 --mtp ./mtp/rt --mtp-q4 all --mtp-hnorm stream --batch-mtp
```

Expected, and what to do if you do not see it:

| Check | Expected | If not |
|---|---|---|
| Output reads as normal text | yes | Garbled text means the wrong engine version. Check `$ENGINE --version` prints `strata 0.1.64`. |
| First token | about 0.5 s | |
| Decode, one user, default sampling | about 55 tok/s (45 with thinking on) | Around 30 tok/s means the drafter did not load. Check the `strata/` pack path in `user_args`. |
| `MemAvailable` under load | 16 to 19 GiB | Much lower: something else is using memory. Rule 8. |

## 3. Seat 3: two Sparks, AEON vLLM Ultimate, 131K, 16 sequences

Target: 2 DGX Sparks joined by a RoCE link. One user at 58 to 65 tok/s. Sixteen at once sharing 330 to 359 tok/s.

Image: `ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-23-v0.33.0-omni-dflash2`, digest `sha256:71c2b82b3b0c97ddca9ef35b279fb745ee0abe29b762a24c8c0771cf930f11ab`. This seat runs on the public image as it is.

**3.1 Download** the `vllm/` build on one node and share it read-only over NFS, or download it on both. The path must be the same on both nodes.

```sh
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --include 'vllm/*' --local-dir /models/aeon-xs
```

**3.2 Fill in five values per node.** These are the only site-specific parts.

| Variable | What it is | How to find it |
|---|---|---|
| `HEAD` | the head node's IP address on the fast link | the address on the RoCE interface of the node you choose as head |
| `MYIP` | this node's IP address on the fast link | `ip -br addr show` on this node |
| `RANK` | `0` on the head node, `1` on the other | |
| `IF` | the network interface that carries the fast link | the `netdev` column of `rdma link show`. Ours was `enp1s0f0np0`. |
| `HCA` | the RDMA devices of that port, comma-separated | the `link` names from `rdma link show` whose `netdev` is that port. Ours were `rocep1s0f0,roceP2p1s0f0`. |

**3.3 Run this on both nodes**, with `RANK=0` on the head and `RANK=1` on the other:

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

The doubled `=` in `NCCL_SOCKET_IFNAME==$IF` and `NCCL_IB_HCA==$HCA` is correct. NCCL reads a leading `=` as an exact match. Do not "fix" it.

**3.4 Wait and test.** Loading takes several minutes. Do not restart the container while `docker logs aeon-xs` is still advancing. When the head node answers on `/health`, send the Seat 1 test request to `http://<head>:8000/v1/chat/completions`. The model name is the same, `aeon-xs`.

Variant, DFlash2 in place of the MTP head: faster for one user (65 tok/s) and for short answers, slower behind very long prompts. Download the drafter, mount it, change one flag:

```sh
huggingface-cli download incoai/Qwen3.8-Flash-Next-DFlash2 --local-dir /models/dflash2
# add:     -v /models/dflash2:/draft:ro
# replace: --speculative-config '{"method":"dflash","model":"/draft","num_speculative_tokens":3}'
```

Do not change: `--tensor-parallel-size 2`, `--max-model-len 131072`, `--max-num-seqs 16`, `--gpu-memory-utilization 0.80`, `--max-num-batched-tokens 16384`, `--enable-prefix-caching`, the draft-token count, the parsers, or the sampling override.

You may change: `--port`, `--served-model-name` (then use that name in requests), and `--limit-mm-per-prompt` to `--language-model-only` for a text-only server.

| Check | Expected | If not |
|---|---|---|
| Engine starts | yes | `QSA ring capacity ... must divide the attention block size`: someone changed the draft-token count. Put it back to 3. |
| First token on a short prompt | 0.1 to 0.2 s | |
| Decode, one user, default sampling | about 58 tok/s (47 with thinking on) | About 34 tok/s means no drafter is active. Check `--speculative-config`. |
| 16 at once, total | 330 to 359 tok/s | |
| Nodes cannot find each other | | Check `IF`, `HCA`, `HEAD`, `MYIP`, and that `/dev/infiniband` exists on both nodes. |

## 4. Seat 4: two Sparks, 256K context, 4 sequences

Target: the same two Sparks as Seat 3, for whole books in context and long reasoning. One user at about 51 tok/s, four at once sharing about 145 tok/s, at the default sampling.

Use the Seat 3 command with these lines in place of the matching ones:

```sh
# same docker flags as the throughput recipe, except:
  --enable-prefix-caching \
  --max-model-len 262144 --max-num-seqs 4 --gpu-memory-utilization 0.80 \
  --max-num-batched-tokens 16384 \
  --speculative-config '{"method":"mtp","num_speculative_tokens":3}' \
  --override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}' \
  $HEADLESS
```

Everything else in section 3 applies. In our runs the head node kept at least 6.9 GiB of memory free. Rule 7 applies.

## 5. Seat 5: four Sparks, two replicas

Target: 4 DGX Sparks as two linked pairs. 32 at once sharing about 748 tok/s.

Bring up Seat 3 twice, once on nodes 1 and 2 and once on nodes 3 and 4. Each pair has its own `HEAD`. The replicas share nothing. Then put any round-robin HTTP proxy in front. With nginx:

```nginx
upstream aeon_xs { server <replica-a>:8000; server <replica-b>:8000; }
server { listen 8080; location / { proxy_pass http://aeon_xs; proxy_buffering off; proxy_read_timeout 3600s; } }
```

Do not start one job with `--tensor-parallel-size 4`. That was not measured on this build.

## 6. Seat 2: one Spark, AEON vLLM Ultimate. Blocked.

This seat was measured (38.9 tok/s for one user, 97.5 across four) with a patch that keeps the 28.8 GB n-gram table on NVMe. That patch is not in the public image. The public image keeps the table in pinned host memory, which is the same pool as the weights on a Spark, and it does not fit.

**Do not run this seat until the model card names a tagged image that includes the offload.** If the person asks for many sessions on one Spark today, give them Seat 1 and say why. For the record, these are the flags we measured with:

```sh
IMG=ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-23-v0.33.0-omni-dflash2
MODEL=/models/aeon-xs/vllm
DRAFT=/models/dflash2

docker run -d --name aeon-xs --ipc host --shm-size 16g --gpus all \
  -v $MODEL:/model:ro -v $DRAFT:/draft:ro \
  --entrypoint vllm $IMG serve /model --trust-remote-code --served-model-name aeon-xs \
  --host 0.0.0.0 --port 8000 \
  --engram-config '{"cpu_offload":true}' \
  --limit-mm-per-prompt '{"image":2,"video":0}' \
  --reasoning-parser qwen3 --tool-call-parser qwen3_coder --enable-auto-tool-choice \
  --enable-prefix-caching \
  --max-model-len 131072 --max-num-seqs 4 --gpu-memory-utilization 0.74 \
  --max-num-batched-tokens 16384 \
  --speculative-config '{"method":"dflash","model":"/draft","num_speculative_tokens":3}' \
  --override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}'
```

When this seat is available, three more things will be true. `--gpu-memory-utilization` stays at 0.74. The first minutes after a boot run at about half speed while the table comes off disk. The serve generated for at most 12 sessions at once, whatever `--max-num-seqs` said.

## 7. Requests: sampling, thinking, images

**Sampling.** Use these values for chat and for long reasoning. Long thinking at temperature 0 can loop. `repetition_penalty` above 1.0 breaks tool-call XML.

```
temperature 0.6
top_p 0.95
top_k 20
presence_penalty 0.5
repetition_penalty 1.0
```

The AEON vLLM Ultimate recipes already set them as the server default with this flag, so clients that send only `temperature` still get the rest:

```
--override-generation-config '{"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}'
```

Strata has no server default. Send the fields with every request, as the test request in 2.4 does.

**Thinking** is switched per request with `"chat_template_kwargs": {"enable_thinking": true}` or `false`. With thinking on, expect roughly 15 to 20% lower decode speed and allow a larger `max_tokens`, because the reasoning counts against it.

**Images.** Both engines accept images as `image_url` parts in the usual OpenAI chat format. AEON vLLM Ultimate allows two images per request (`--limit-mm-per-prompt '{"image":2,"video":0}'`). Video is off in every recipe and was not tested.

**Tools.** The AEON vLLM Ultimate recipes start with `--reasoning-parser qwen3 --tool-call-parser qwen3_coder --enable-auto-tool-choice`. Leave them.

## 8. Before you report

Check each of these and give the person the result:

1. Which seat you brought up, on which hosts, and the base URL.
2. The test request returned normal text.
3. Decode speed you measured for one request, next to the expected figure for the seat.
4. `MemAvailable` on each node with the serve idle.
5. Whether the port is reachable from other machines.
6. Anything you changed from this file, and why. If the answer is "nothing", say so.
7. That the model is uncensored and the person is responsible for how it is used.

## 9. What goes wrong

| Symptom | Cause | Fix |
|---|---|---|
| Garbled text from the first token on Strata | engine 0.1.65 or later | reinstall the pinned 0.1.64 packages from 2.2 |
| Strata's template engine rejects the chat template | the config points at the wrong tokenizer | `"tokenizer"` must be `serve/tokenizer`, whose template has the `{% generation %}` tags removed |
| AEON vLLM Ultimate dies at startup with a `QSA ring capacity` assertion | draft tokens set to 7 or 8 | set `num_speculative_tokens` to 3 |
| The serve or the whole machine is killed during warm-up | `--moe-backend flashinfer_b12x`, or `--gpu-memory-utilization` raised | remove the flag, restore the value |
| One Spark, AEON vLLM Ultimate, out of memory at load | Seat 2 on the public image | not available yet. Use Seat 1. |
| Thinking answers run to the token limit repeating themselves | temperature 0 | use the sampling values in section 7 |
| Tool calls come back malformed | `repetition_penalty` above 1.0 | set it to 1.0 |
| Sessions past a point wait instead of running | the engine's parallel limit: 8 on Strata, 12 observed on one-Spark AEON vLLM Ultimate | expected. Add Sparks for more parallel sessions. |
| A long prompt takes tens of seconds before the first token | prompt reading, about 2,100 tokens a second | expected. A 96K-token prompt takes about 40 s on two Sparks. |

## 10. What this file does not cover

Fine-tuning, re-quantizing, serving the 360 GB BF16 master, hardware other than DGX Spark, and any engine other than Strata and AEON vLLM Ultimate. None of those has a validated recipe here. If the person asks for one, tell them so before you try anything.
