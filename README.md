# Crynux + Pearl Dual Mining

A customized [Crynux node](https://github.com/crynux-network/crynux-node) that dual-mines Pearl on the same GPU — stay on Crynux and mine Pearl at the same time.

Built on Crynux Node **3.7.1**, with **WildRig Multi 0.51.9** mining `pearlhash` against the [Pearlhash](https://pearlhash.xyz/) pool (`pool.pearlhash.xyz:9000`).

## Overview / how dual mining works

Crynux tasks are rare. Even with heartbeat tasks, a node usually runs only a few short jobs per day, so Pearl keeps almost all of the GPU time.

Those few, irregular tasks still decide your share of Crynux **weekly CNX emission** — and that emission is large. Dual mining stays joined to Crynux to keep earning that emission weight, while the GPU mines Pearl the rest of the time.

In the published image, one container runs:

1. The Crynux node
2. A dual-mining controller
3. WildRig (`pearlhash`) as a child process when Crynux is idle

When a Crynux task arrives, Pearl mining pauses so Crynux can use the GPU. When the task finishes, Pearl mining resumes.

> Next step: replace the GEMMs inside Crynux model inference with Pearl's PoUW matmul path (NoisyGEMM / `pearl-gemm`), so running a Crynux task would also mine Pearl at the same time.

## Impact on Crynux

- Pearl may force the node to drop its in-VRAM model cache, which can reduce Crynux locality / matching weight until the next task reloads a model.
- Switching from Pearl to a Crynux task adds a few seconds of delay.

## Impact on Pearl

- Pearl does not mine while a Crynux task is running.
- Each Crynux task interrupts Pearl for a few extra seconds around the switch.

## Usage

Requirements: Docker with the NVIDIA Container Toolkit, a Crynux node wallet, and a separate Pearl payout address (`prl1...`).

> Startup is the same as the [official Crynux Node Docker guide](https://docs.crynux.io/node-hosting/start-a-node/start-a-node-docker), with two differences:
>
> 1. Use the dual-mining images from [ghcr.io/chilo-zk1022/crynux-node](https://github.com/chilo-zk1022/crynux-node/pkgs/container/crynux-node) instead of the official Crynux image. Builds are published for both official L2 networks: Crynux-on-Robinhood (`*-rh-pearl`) and Crynux-on-Base (`*-base-pearl`).
> 2. Set your Pearl payout address in `config/config.yml` under `dual_mining.pearl_address`.

1. Clone this repository.

2. Choose a Crynux L2 network. Copy the matching config example, and set the matching image tag in `docker-compose.yml`:

```bash
# Crynux-on-Robinhood
cp config/config.yml.rh.example config/config.yml
# image: ghcr.io/chilo-zk1022/crynux-node:3.7.1-rh-pearl

# Or Crynux-on-Base
cp config/config.yml.base.example config/config.yml
# image: ghcr.io/chilo-zk1022/crynux-node:3.7.1-base-pearl
```

3. Put your Crynux private key in `config/private_key.txt` (same format as a normal Crynux Docker node).

4. Edit `config/config.yml` and set your Pearl address under `dual_mining`:

```yaml
dual_mining:
  enabled: true
  pearl_address: "prl1..."
  pearl_worker: "crynux-dual"
  pearl_pool_url: "stratum+tcp://pool.pearlhash.xyz:9000"
  pearlhash_kernel: "2"
  free_vram_threshold_mib: 1024
  idle_grace_sec: 2.0
```

5. Start:

```bash
docker compose up -d
```

6. Follow logs:

```bash
docker compose logs -f
```

Controller lines are prefixed with `[dual_controller]`.

### Layout

| Path | Purpose |
|---|---|
| `docker-compose.yml` | Single service; mounts `config/` and `data/` |
| `config/config.yml.rh.example` | Crynux-on-Robinhood network template |
| `config/config.yml.base.example` | Crynux-on-Base network template |
| `config/config.yml` | Your real config (gitignored) |
| `config/private_key.txt` | Crynux wallet key (gitignored) |
| `data/tmp`, `data/db`, `data/logs` | Persistent node data (contents gitignored) |

### Image tags

- `ghcr.io/chilo-zk1022/crynux-node:<version>-rh-pearl` — Crynux-on-Robinhood
- `ghcr.io/chilo-zk1022/crynux-node:<version>-base-pearl` — Crynux-on-Base

Replace `<version>` with the release you want (for example `3.7.1`).
