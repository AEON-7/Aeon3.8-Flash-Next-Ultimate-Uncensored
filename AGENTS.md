# AGENTS.md: serve Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS

You are an AI agent. A person asked you to serve this model on their hardware. Do the sections in order. Run every command exactly as written. Change only values written `<like-this>`.

Run each code block as ONE shell call, whole. Every block is complete on its own: it sets its own folder and its own variables, so it works in a fresh shell.

- What it is: uncensored Qwen3.8-Flash-Next, 180B parameters, 4-bit NVFP4. Text and images in, text out.
- What you end with: an OpenAI-compatible server. Model name in requests: `aeon-xs`.
- Validated hardware: NVIDIA DGX Spark only, as 1, 2 or 4 units. Nothing else has a recipe.
- Weights: <https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS> (public, no token needed)
- Guides for people: <https://github.com/AEON-7/Aeon3.8-Flash-Next-Ultimate-Uncensored>

## Rules

1. Use only a recipe from this file. If the hardware matches none, stop and tell the person.
2. Do not change flags, versions, image tags or sampling values. Section 9 lists the only changes allowed.
3. Do not stop, remove or reconfigure anything else on the machine. If the GPU or the memory is already in use, stop and ask the person.
4. If a command fails, look in section 11. If the failure is not listed, stop and report the exact error. Do not improvise.
5. Watch memory with `grep MemAvailable /proc/meminfo`. Under 5,200,000 kB (5 GiB): send no more load. Under 4,200,000 kB (4 GiB): stop the server and report.
6. No seat sets an API key. Seat 1 listens on `127.0.0.1` only. Seats 2 to 5 listen on `0.0.0.0`, so other machines on the network can reach them. Tell the person. Do not open firewalls or expose ports yourself.
7. The model is uncensored. Tell the person they are responsible for how it is used. Never describe it as safe.
8. Report what you measured, not what this file says to expect.
9. When the test passes, leave the server running. Stop it only if the person asks, or if rule 5 says so. Section 9 has the stop commands.
10. If `sudo` asks for a password, stop and ask the person to run that line.

## 1. Identify the hardware

```sh
nvidia-smi --query-gpu=name --format=csv,noheader   # DGX Spark prints: NVIDIA GB10
uname -m                                            # DGX Spark prints: aarch64
grep MemTotal /proc/meminfo                         # DGX Spark: about 127,500,000 kB
df -h ~ /                                           # free disk: Seat 1 downloads into ~, Seats 2 to 5 into /models
docker ps                                           # must work without error for Seats 2 to 5
rdma link show                                      # two or more Sparks: lists the RoCE link
```

| You have | The person wants | Use | Go to |
|---|---|---|---|
| 1 DGX Spark | the fastest single stream, at most 4 sessions at once | **Seat 1**: Strata | section 3 |
| 1 DGX Spark | more than 4 sessions at once | **Seat 2**: AEON vLLM Ultimate | section 4 |
| 2 DGX Sparks joined by a RoCE link | a shared server, up to 32 sessions | **Seat 3** | section 5 |
| 2 DGX Sparks joined by a RoCE link | more than 131K of context, or long reasoning | **Seat 4** | section 6 |
| 4 DGX Sparks, as two linked pairs | the most throughput | **Seat 5** | section 7 |
| 3 DGX Sparks | | Seat 3 on a linked pair. Seat 1 on the third if wanted. | sections 5 and 3 |
| anything else: other GPUs, less memory, two Sparks with no RoCE link | | **Stop.** Not validated. Tell the person. | |

If one Spark and the person did not say: use Seat 1. If two Sparks and the person did not say: use Seat 3.

| Seat | Free disk needed | Download | Port | Listens on |
|---|---|---|---|---|
| 1 | 110 GB | repo without `vllm/` | 8300 | 127.0.0.1 |
| 2 | 140 GB | `vllm/` + drafter, plus 29 GB written at first boot | 8000 | 0.0.0.0 |
| 3 | 110 GB, readable at the same path on both nodes | `vllm/` + drafter | 8000 | 0.0.0.0 |
| 4 | 110 GB, readable at the same path on both nodes | `vllm/` | 8000 | 0.0.0.0 |
| 5 | as Seat 3, for each pair | `vllm/` + drafter | 8000 per pair, 8080 on the balancer | 0.0.0.0 |

## 2. Install the downloader (every seat)

```sh
python3 -m venv ~/hf-cli
~/hf-cli/bin/pip install -U huggingface_hub
export PATH="$HOME/hf-cli/bin:$PATH"
hf --version
```

If `python3 -m venv` fails: `sudo apt-get install -y python3-venv`, then repeat. The download blocks below set `PATH` themselves. Do not use `huggingface-cli`: it no longer exists.

For Seats 2 to 5, make the folder the recipes use:

```sh
sudo mkdir -p /models && sudo chown $USER /models
```

## 3. Seat 1: one Spark, Strata

**3.1 Download** into `~/aeon-xs` (about 106 GB):

```sh
export PATH="$HOME/hf-cli/bin:$PATH"
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --exclude 'vllm/*' --local-dir ~/aeon-xs
```

**3.2 Get the pinned engine.** Both hash checks must print `OK` and the last line must print `strata 0.1.64`. If not, stop.

```sh
cd ~/aeon-xs
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

**3.3 Check the files and write the config.** It must print `FILES-OK`. If it does not, a download is incomplete: repeat 3.1 or 3.2.

```sh
cd ~/aeon-xs
ENGINE=$PWD/strata-engine-0.1.64/strata
ls -d pack dense.gguf ple-nvfp4.gguf expert-profile.bin serve/tokenizer mmproj-Aeon3.8-Flash-Next-BF16.gguf strata/qwen3.8-flash-next-dflash2.strata1.q4_0-selq4.pack strata-engine-0.1.64/strata strata-engine-0.1.64/strata-vision strata-terminal-0.1.64/bin/strata >/dev/null && echo FILES-OK
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
```

**3.4 Start the server in the background and wait for it.** The server writes `ready: http://127.0.0.1:8300/v1` to `strata-serve.log` when it is up, and the loop prints `READY` when it sees that line. The server keeps running after the command returns.

```sh
cd ~/aeon-xs
setsid nohup ./strata-terminal-0.1.64/bin/strata serve --config aeon-xs-dflash2.json --host 127.0.0.1 --port 8300 --parallel 4 > strata-serve.log 2>&1 < /dev/null &
for i in $(seq 1 120); do grep -q 'ready: http' strata-serve.log && echo READY && break; sleep 5; done
```

If `READY` is not printed within 10 minutes: `tail -40 ~/aeon-xs/strata-serve.log`, stop, report.

**3.5 Test.**

```sh
curl -s http://127.0.0.1:8300/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"aeon-xs","messages":[{"role":"user","content":"Name three prime numbers."}],"max_tokens":64,"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}'
```

Pass: the reply contains normal text naming prime numbers. Then do section 8.

| Check | Expected | If not |
|---|---|---|
| `$ENGINE --version` | `strata 0.1.64` | wrong package. Repeat 3.2. |
| reply text | readable | garbled text means another engine version is running |
| speed, one request | about 55 tok/s, 45 with thinking on | about 30 means the drafter did not load: check the `strata/` path in `user_args` |
| `MemAvailable` under load | 17,000,000 to 20,000,000 kB (16 to 19 GiB) | much lower: something else is using memory. Rule 3. |

## 4. Seat 2: one Spark, AEON vLLM Ultimate

Image: `ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-25-v0.33.0-omni-dflash2-ngram-disk` (digest `sha256:4749bd067524ce41a4a6d59625f6b6c8531d08a14739bd0d416e935b9fc3a1ad`). Only this image fits on one Spark. The `2026-11-23` image does not.

**4.1 Download** (about 109 GB):

```sh
export PATH="$HOME/hf-cli/bin:$PATH"
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --include 'vllm/*' --local-dir /models/aeon-xs
hf download incoai/Qwen3.8-Flash-Next-DFlash2 --local-dir /models/dflash2
```

**4.2 Start.** For up to 4 sessions at the lowest latency, run as written. For more sessions, change `--max-num-seqs 4` to `--max-num-seqs 32` and nothing else: the server then accepts 32 sessions, works on about 12 at a time, and queues the rest.

```sh
IMG=ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-25-v0.33.0-omni-dflash2-ngram-disk
MODEL=/models/aeon-xs/vllm
DRAFT=/models/dflash2
TABLE=/models/aeon-xs-ple   # NVMe dir for the n-gram table; created on first boot, reused later

docker run -d --name aeon-xs --ipc host --shm-size 16g --gpus all \
  -v $MODEL:/model:ro -v $DRAFT:/draft:ro -v $TABLE:/ple \
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

**4.3 Wait.** The first boot writes the 29 GB n-gram table to `/models/aeon-xs-ple` and is the slowest.

```sh
for i in $(seq 1 180); do curl -sf http://127.0.0.1:8000/health >/dev/null && echo READY && break; sleep 10; done
```

If `READY` is not printed within 30 minutes: `docker logs --tail 80 aeon-xs`, stop, report.

**4.4 Test.**

```sh
curl -s http://127.0.0.1:8000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"aeon-xs","messages":[{"role":"user","content":"Name three prime numbers."}],"max_tokens":64}'
```

Pass: the reply contains normal text. Then do section 8.

| Check | Expected | If not |
|---|---|---|
| speed, one request | about 36 tok/s, 30 with thinking on | the first minutes after a boot run at about half speed. Send 5 requests, then measure again. |
| 4 requests at once, total | about 79 tok/s | |
| sessions past 12 at once | they queue | expected. This serve ran at most 12 sessions at a time, whatever the cap. |

## 5. Seat 3: two Sparks, 131K context, up to 32 sessions

Image: `ghcr.io/aeon-7/aeon-vllm-ultimate:2026-11-23-v0.33.0-omni-dflash2` (digest `sha256:71c2b82b3b0c97ddca9ef35b279fb745ee0abe29b762a24c8c0771cf930f11ab`).

**5.1 Download.** `/models` must hold the same files on both nodes. If `/models` is one shared folder (NFS) mounted on both: run this once. If not: run it on each node.

```sh
export PATH="$HOME/hf-cli/bin:$PATH"
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --include 'vllm/*' --local-dir /models/aeon-xs
hf download incoai/Qwen3.8-Flash-Next-DFlash2 --local-dir /models/dflash2
```

Then check on each node. Both lines must list files:

```sh
ls /models/aeon-xs/vllm | head -3
ls /models/dflash2 | head -3
```

**5.2 Find four values on each node.**

| Value | What it is | How to find it |
|---|---|---|
| `HEAD` | the head node's IP address on the RoCE link | choose one node as head. Use its address on the RoCE interface. |
| `MYIP` | this node's IP address on the RoCE link | `ip -br addr show` |
| `IF` | the interface that carries the RoCE link | the `netdev` column of `rdma link show`. Example: `enp1s0f0np0`. |
| `HCA` | the RDMA devices of that port, comma-separated | the link names in `rdma link show` whose `netdev` is `IF`. Example: `rocep1s0f0,roceP2p1s0f0`. |

**5.3 Start on both nodes.** Start the head node first with `RANK=0`, then the other node straight away with `RANK=1`. Replace the four `<...>` values on each.

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

`NCCL_SOCKET_IFNAME==$IF` and `NCCL_IB_HCA==$HCA` have two `=` signs on purpose. Do not remove one.

**5.4 Wait**, on the head node. Loading takes several minutes.

```sh
for i in $(seq 1 180); do curl -sf http://127.0.0.1:8000/health >/dev/null && echo READY && break; sleep 10; done
```

If `READY` is not printed within 30 minutes: `docker logs --tail 80 aeon-xs` on both nodes, stop, report.

**5.5 Test**, on the head node.

```sh
curl -s http://127.0.0.1:8000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"aeon-xs","messages":[{"role":"user","content":"Name three prime numbers."}],"max_tokens":64}'
```

Pass: the reply contains normal text. Then do section 8.

| Check | Expected | If not |
|---|---|---|
| first token on a short prompt | 0.1 to 0.2 s | |
| speed, one request | about 60 tok/s, 57 with thinking on | about 35 means no drafter: check `--speculative-config` and the `/draft` mount |
| 16 requests at once, total | about 248 tok/s, 235 with thinking on | |
| 32 requests at once, total | about 243 tok/s, 312 with thinking on | short sessions only. 32 full 131K contexts do not fit. |

Variant, only if the person asks for the built-in MTP head (mostly non-English traffic, or very long prompts): remove ` -v $DRAFT:/draft:ro`, set `--max-num-seqs 16`, and set `--speculative-config '{"method":"mtp","num_speculative_tokens":3}'`.

## 6. Seat 4: two Sparks, 256K context, 4 sessions

Same image as Seat 3. No drafter download.

**6.1 Download** as in 5.1, but leave out the `incoai/...` line (the drafter), and skip the `/models/dflash2` check.

**6.2 Find the four values** as in 5.2.

**6.3 Start on both nodes.** Start the head node first with `RANK=0`, then the other node straight away with `RANK=1`. Replace the four `<...>` values on each.

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

**6.4 Wait and test** exactly as in 5.4 and 5.5.

| Check | Expected |
|---|---|
| speed, one request | about 51 tok/s, 43 with thinking on |
| 4 requests at once, total | about 145 tok/s, 100 with thinking on |
| `MemAvailable` on the head node | about 7,200,000 kB (6.9 GiB) at the lowest in our runs. For information: the limits are in rule 5. |

## 7. Seat 5: four Sparks, two replicas

1. Do section 5 on nodes 1 and 2. Do section 5 again on nodes 3 and 4. Each pair has its own `HEAD`. The pairs share nothing.
2. On any machine that can reach both head nodes, put a round-robin proxy in front. With nginx on Ubuntu: `sudo apt-get install -y nginx`, save the block below as `/etc/nginx/conf.d/aeon-xs.conf` with the two head-node addresses filled in, then `sudo nginx -t && sudo systemctl reload nginx`.

```nginx
upstream aeon_xs { server <replica-a>:8000; server <replica-b>:8000; }
server { listen 8080; location / { proxy_pass http://aeon_xs; proxy_buffering off; proxy_read_timeout 3600s; } }
```

3. Test with the 5.5 request against `http://127.0.0.1:8080/v1/chat/completions` on the proxy machine.

Expected: 32 requests at once share about 748 tok/s. Do not start one job with `--tensor-parallel-size 4`. It was not measured.

## 8. Measure one request (every seat)

Set `URL` to the seat's address, then run:

```sh
URL=http://127.0.0.1:8000/v1/chat/completions   # Seat 1: port 8300. Seat 5: port 8080.
python3 - "$URL" <<'PY'
import json, sys, time, urllib.request
body = {"model": "aeon-xs", "max_tokens": 400,
        "messages": [{"role": "user", "content": "Write a 300-word story about a lighthouse."}],
        "temperature": 0.6, "top_p": 0.95, "top_k": 20, "presence_penalty": 0.5, "repetition_penalty": 1.0,
        "chat_template_kwargs": {"enable_thinking": False}}
req = urllib.request.Request(sys.argv[1], json.dumps(body).encode(), {"Content-Type": "application/json"})
t = time.time()
r = json.load(urllib.request.urlopen(req, timeout=600))
dt = time.time() - t
n = (r.get("usage") or {}).get("completion_tokens")
print("tokens:", n, "| seconds: %.1f" % dt, "| tok/s: %.1f" % (n / dt) if n else "| no usage field in the reply")
PY
grep MemAvailable /proc/meminfo
```

The tok/s it prints includes the wait for the first token, so it reads a little under the seat's expected figure. Report the number you got.

## 9. What you may change

- `--port`, and on Seat 1 `--host`, only if the person asks.
- Text only, no images, Seats 2 to 5: replace `--limit-mm-per-prompt '{"image":2,"video":0}'` with `--language-model-only`. On Seat 1 leave images on: text speed is the same either way.
- Seat 1: leave `parallel` at 4, also for a single user. Never raise it.
- Seat 2: `--max-num-seqs 32` in place of `4`.

Stop commands, for when the person asks or rule 5 says so:

- Seat 1: `pkill -f strata-terminal-0.1.64/bin/strata`. Then `pgrep -fa strata-engine-0.1.64` must print nothing. If it prints a process: `pkill -f strata-engine-0.1.64/strata`.
- Seats 2 to 5: `docker rm -f aeon-xs`, on every node of the pair.

Never change: Strata 0.2.18 (engine 0.1.64), the image tags, `--gpu-memory-utilization` (0.74 on one Spark, 0.80 on two), `--max-num-batched-tokens 16384`, `--enable-prefix-caching`, `num_speculative_tokens` 3, Strata's `--spec 6 --draft-conf 0.5`, the parsers, the sampling values. Never add `--moe-backend`. Never run `strata update`.

## 10. How clients must call it

Tell the person these four things.

1. **Address.** `http://<host>:<port>/v1`, model name `aeon-xs`, no API key.
2. **Sampling.** Use exactly these values. Temperature 0 makes long reasoning loop. `repetition_penalty` above 1.0 breaks tool calls.

```
temperature 0.6
top_p 0.95
top_k 20
presence_penalty 0.5
repetition_penalty 1.0
```

   Seats 2 to 5 apply them by default. On Seat 1 the client must send all five with every request.
3. **Thinking.** Per request: `"chat_template_kwargs": {"enable_thinking": true}` or `false`. Thinking counts against `max_tokens` and is slower.
4. **Images.** As `image_url` parts in the OpenAI chat format. Seats 2 to 5 take up to two images per request. Video is off.

## 11. Failures

| Symptom | Cause | Fix |
|---|---|---|
| `hf: command not found` | section 2 was skipped or failed | do section 2 |
| `huggingface-cli: command not found` | that command was removed | use `hf download` with the same arguments |
| download fails with "permission denied" under `/models` | the folder belongs to root | `sudo mkdir -p /models && sudo chown $USER /models` |
| `docker: permission denied` | the user is not in the docker group | tell the person. Do not change group membership yourself. |
| Strata prints garbled text | engine 0.1.65 or later | repeat 3.2, check `strata 0.1.64` |
| Strata rejects the chat template | wrong tokenizer path | `"tokenizer"` must end in `serve/tokenizer` |
| Seat 2 runs out of memory while loading | wrong image, or `/ple` not mounted | use the `ngram-disk` image and the command in 4.2 unchanged |
| `QSA ring capacity ... must divide the attention block size` at startup | `num_speculative_tokens` is not 3 | set it to 3 |
| the server or the whole machine is killed during warm-up | `--moe-backend` was added, or `--gpu-memory-utilization` was raised | remove the flag, restore the value |
| two-Spark nodes never connect | a wrong `HEAD`, `MYIP`, `IF` or `HCA`, or `/dev/infiniband` missing | re-read 5.2 on both nodes |
| thinking answers repeat until the token limit | temperature 0 | use the section 10 values |
| tool calls come back malformed | `repetition_penalty` above 1.0 | set it to 1.0 |
| requests past 4 (Seat 1) or 12 (Seat 2) at once wait | the session limit: 4 in the Seat 1 recipe (Strata never decodes more than 8 at once at any setting), 12 observed on Seat 2 | expected. More Sparks give more parallel sessions. |
| a long prompt waits tens of seconds for its first token | prompt reading, about 2,100 tokens a second | expected. 96K tokens take about 40 s on two Sparks. |

## 12. Final report to the person

1. Seat, hosts, base URL.
2. The test reply, quoted.
3. The section 8 output: tok/s and `MemAvailable`.
4. Which addresses the port listens on, that there is no API key, and that the server was left running.
5. Every change you made to a command here, or "none".
6. "This model is uncensored. You are responsible for how it is used."

## Not covered

Hardware other than DGX Spark, other engines (SGLang and TensorFold cannot load this checkpoint), the 360 GB BF16 master, fine-tuning, re-quantizing. If the person asks for one of these, say there is no validated recipe before doing anything.
