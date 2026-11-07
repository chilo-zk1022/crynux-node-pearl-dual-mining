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

Requirements: Docker with the NVIDIA Container Toolkit, a Crynux node wallet, and a separate Pearl payout address (`prl1...`). Do **not** reuse the Crynux EVM private key as a Pearl address.

1. Clone this repository.

2. Choose a network and copy the matching example:

```bash
# Robinhood (default image tag in docker-compose.yml)
cp config/config.yml.rh.example config/config.yml

# Or Crynux-on-Base (also change the compose image to *-base-pearl)
cp config/config.yml.base.example config/config.yml
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

5. If you use Base, edit `docker-compose.yml` and set the image tag to the Base pearl image, for example:

```yaml
image: ghcr.io/chilo-zk1022/crynux-node:3.7.1-base-pearl
```

The default compose file uses:

```yaml
image: ghcr.io/chilo-zk1022/crynux-node:3.7.1-rh-pearl
```

6. Start:

```bash
docker compose up -d
```

7. Follow logs:

```bash
docker compose logs -f
```

Controller lines are prefixed with `[dual_controller]`.

### Layout

| Path | Purpose |
|---|---|
| `docker-compose.yml` | Single service; mounts `config/` and `data/` |
| `config/config.yml.rh.example` | Robinhood network template |
| `config/config.yml.base.example` | Base network template |
| `config/config.yml` | Your real config (gitignored) |
| `config/private_key.txt` | Crynux wallet key (gitignored) |
| `data/tmp`, `data/db`, `data/logs` | Persistent node data (contents gitignored) |

### Image tags

- `ghcr.io/chilo-zk1022/crynux-node:<version>-rh-pearl`
- `ghcr.io/chilo-zk1022/crynux-node:<version>-base-pearl`

Replace `<version>` with the release you want (for example `3.7.1`).
