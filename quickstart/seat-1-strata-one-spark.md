# Seat 1: Strata on one Spark

[Quick-start index](../README.md) · [Sampling settings](sampling.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

> **Hardware** 1 DGX Spark, 128 GB · **Download** 98 GiB · **Context** 131K · **Images** on · **Engine** Strata 0.2.18 with the DFlash2 drafter

| measured 25 November 2026 | one user | four at once (total) |
|---|---|---|
| Greedy, thinking off | **60.6 tok/s** | **73.0 tok/s** |
| Recommended sampling, thinking off | 55.2 | 66.4 |
| Recommended sampling, thinking on | 45.3 | 67.0 |

That is about 45 words a second, roughly ten times faster than you read. Answers start in about half a second. With no drafter the same box does 30 tok/s, so the drafter doubles it. A five-prompt thinking-on loop check came back clean. Between 16 and 19 GiB of memory stayed free throughout.

We ran this benchmark twice, three days apart, on two different Sparks. The 11-workload mean came out at 60.1 and then 60.6 tok/s. The second run used this recipe as written, release packages and all.

**1. Download.** [Strata](https://github.com/Niko1221/Strata) is an open-source local inference engine with a native NVFP4 path. The root of this repo is a Strata pack. Take everything except the vLLM folder:

```sh
hf download AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS --exclude 'vllm/*' --local-dir aeon-xs
cd aeon-xs
```

**2. Get the engine.** Pin **Strata 0.2.18** (engine 0.1.64 and Terminal 0.1.64, CUDA 13, linux-aarch64). Every Strata number here was measured on it. The next release garbles this model's output and the one after is slower (see [Strata versions](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS#strata-versions-on-dgx-spark)). The release has two packages: the engine, which runs the model, and the Terminal, which carries its own Python and the OpenAI-compatible server. Fetch both from the repo root and check their hashes:

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

**3. Write the config and serve.** This is the launch we benchmarked. The server starts the engine with these flags, adds `--draft dflash2 --draft-fallback none` and its own draft vocabulary, and listens on port 8300.

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

**4. Check it.** When it prints `ready: http://127.0.0.1:8300/v1`, from a second shell:

```sh
curl -s http://127.0.0.1:8300/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"aeon-xs","messages":[{"role":"user","content":"Name three prime numbers."}],"max_tokens":64,"temperature":0.6,"top_p":0.95,"top_k":20,"presence_penalty":0.5,"repetition_penalty":1.0}'
```

Things worth knowing:

- **Images.** `--vision` and the `vision` section turn on image input. The server hands each image to `strata-vision`, which encodes it with the bundled `mmproj-Aeon3.8-Flash-Next-BF16.gguf`. Leave both out for a text-only server. Text speed is the same either way.
- **Tokenizer.** The config points at `serve/tokenizer`, whose chat template has the Hugging Face `{% generation %}` tags removed, because Strata's template engine rejects them.
- **`--parallel 4`** is the launch behind the table. A single user is just as fast at `--parallel 1`.
- **Stay on 0.1.64.** Do not run a plain `strata update`: it installs the newest engine and switches to it. Strata never updates on its own, and the binary above sits outside Strata's component folder, so updates do not touch it. If you also use the Strata Terminal, `strata components use 0.1.64` switches back and `strata components engines` shows which engine is active.
- **The drafter** in `strata/` is Strata's tuned DFlash2 pack: [incoai/Qwen3.8-Flash-Next-DFlash2](https://huggingface.co/incoai/Qwen3.8-Flash-Next-DFlash2), fine-tuned by the Strata project and packed in its runtime format (SHA-256 `4df2e5ab2b47c064afbe662e16e4f06569877125c2d0e2123a4ac755266c664c`). It is Apache-2.0, with its license and notice beside it. `draft_vocab.bin` is byte-identical to the Terminal's default draft vocabulary.
- **Provenance of the numbers.** Our tables come from engine 0.1.64 built from the v0.2.18 source tag with Strata's own Spark build script. The release packages above are built from the same tag and, run exactly as shown, gave the same greedy output in our 4-prompt check, with and without DFlash2.

<details>
<summary><b>Variant: the built-in MTP head</b> (multilingual traffic, no separate drafter)</summary>

In the config set `"drafting": "mtp"`, delete `"user_args"`, and add these flags to `"args"` (write `./mtp/rt` as a full path, like the others):

```sh
  ... --spec 4 --spec-min-p 0.5 --mtp ./mtp/rt --mtp-q4 all --mtp-hnorm stream --batch-mtp
```

The MTP flags without `--batch-mtp` are also stored in `strata-aeon-small-gptq.json`.

| measured 25 November 2026 | one user | four at once (total) |
|---|---|---|
| Greedy, thinking off | 52.2 tok/s | 67.9 tok/s |
| Recommended sampling, thinking off | 47.1 | 64.3 |
| Recommended sampling, thinking on | 39.0 | 63.1 |

DFlash2 beat it in every cell, so DFlash2 is the default. MTP's one clear win is multilingual text: 46 against 40 tok/s greedy, 50 against 39 at the recommended sampling.

</details>

<details>
<summary><b>What happens if you raise Strata's session cap to 32</b> (an experiment, not a recipe)</summary>

A 131K, vision-on, `--parallel 32` launch does not keep 5 GiB of memory free on one Spark. We got a 32-session cap to start by giving up context, cache and vision:

- `--max-context` 32768 (and `--prefill auto:32768`)
- `--kv-resident` 8192
- `--conversation-cache-mib` 1024
- no `--vision` and no `vision` section
- `--parallel 32`, same DFlash2 flags as above

The Terminal says the engine decodes at most 8 requests together. More sessions are admitted and pass through that window in waves. On a fresh start, 8 concurrent generations stayed above 5 GiB free (5.77 GiB with thinking off, 5.69 GiB with thinking on). 12 dropped to 4.43 GiB, and 16 hit 3.74 GiB and were stopped.

Free memory did not come back between passes. On the next pass over the same serve it was already at 4.7 GiB with a single session, and 8 sessions were stopped at 3.7 GiB. So we do not recommend leaving this running. For several sessions on one Spark, stay with `--parallel 4`, or use Seat 2.

</details>

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
