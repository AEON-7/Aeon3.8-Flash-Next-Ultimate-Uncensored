# Before you start

[Quick-start index](../README.md) · [Before you start](before-you-start.md) · [Sampling](sampling.md) · [Use it](use-it.md) · [AGENTS.md](../AGENTS.md) · [Model card and all measurements](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS)

**What you are setting up.** Three pieces: the **weights** (this repo), an **engine** that runs them (Strata or AEON vLLM Ultimate), and a small **drafter** model that roughly doubles the speed. Once it is running, the engine answers at an OpenAI-compatible URL, so any app or script that can talk to OpenAI can talk to it.

Every grey block on this page is meant to be pasted into a terminal on the Spark, in order, exactly as it is. The only values you ever fill in yourself are written `<like-this>`, and only the multi-Spark seats have any.

**1. Check the machine.** The first line prints `NVIDIA GB10` on a DGX Spark. The second shows the free disk space in the folder you are in.

```sh
nvidia-smi --query-gpu=name --format=csv,noheader
df -h .
```

| Seat | Free disk you need |
|---|---|
| 1 | about 110 GB |
| 2 | about 140 GB: the weights, the drafter, and 29 GB for the n-gram table |
| 3, 4, 5 | about 110 GB, on storage both nodes of a pair can read |

**2. Install the downloader.** `hf` is Hugging Face's command-line tool. The weights are public, so you need no account and no token.

```sh
python3 -m venv ~/hf-cli
~/hf-cli/bin/pip install -U huggingface_hub
export PATH="$HOME/hf-cli/bin:$PATH"
hf --version
```

If the first line fails, run `sudo apt-get install -y python3-venv` and start again. The `export` line lasts until you close the terminal. Add it to `~/.bashrc` to keep it. Older guides use `huggingface-cli download`; that command has been removed from current releases, and `hf download` is the same thing.

**3. Seats 2 to 5 only: check Docker, and make the folder the recipes use.**

```sh
docker ps
sudo mkdir -p /models && sudo chown $USER /models
```

`docker ps` should print a header line, plus one row for each container already running. If it answers "permission denied", run `sudo usermod -aG docker $USER`, log out and back in, and try again.

Now go to your seat.

---

Measurements, methods, charts and limits are on the [model card](https://huggingface.co/AEON-7/Aeon3.8-Flash-Next-ULTIMATE-UNCENSORED-NVFP4-XS). Back to the [quick-start index](../README.md).
