# zs-node

The operator node for the [ZeroSignal](https://zerosignal.ai) private inference
network.

This repository hosts the **prebuilt release binaries** for `zs-node`. The
binaries on the [Releases](https://github.com/txnlab/zs-node/releases) page are
the canonical downloads for Homebrew, Scoop, direct install, and the public
Docker image.

## How it works

`zs-node` is the program an **operator** runs to earn by serving inference. It
sits in front of your own inference backend (vLLM, llama.cpp, LM Studio, an
OpenAI-compatible gateway, or Vertex AI), advertises your model catalog and
pricing on-chain, and admits paid requests:

- A caller's request arrives **sealed** (age-encrypted) with an on-chain payment
  reservation. The node verifies the reservation, decrypts the prompt, forwards
  it to your backend, and seals the response on the way back. Prompts and
  responses are never logged or retained.
- Admission is via the **on-chain seal**, not an API key. The node signs a usage
  receipt and settles the caller's escrow to your payout account in USDC.

You keep two Algorand accounts: an **owner** key (cold — receives funds, stays
off the node) and a **signing** key (hot — on the node, signs tickets and
receipts).

Full concepts — the payment flow, encryption and keys, registration, health, and
economics — live in the operator guide at
**[docs.zerosignal.ai/operators](https://docs.zerosignal.ai/operators)**.

## Quick start

1. **Register** as an operator with your owner wallet via the ZeroSignal
   dashboard — this returns the `operator_id` you put in the config. See
   [Registering on-chain](https://docs.zerosignal.ai/operators/registration).
2. **Install** `zs-node` (below).
3. **Configure** a `config.yaml` (backend, models, pricing, `operator_id`) and
   supply your signing mnemonic via the environment — see
   [Configuration](#configuration).
4. **Run** it:
   ```sh
   zs-node --config config.yaml
   ```
5. **Run it as a service** so it survives reboots — `zs-node install-service`
   (below).

## Install

### macOS (Homebrew)

```sh
brew install txnlab/tap/zs-node
```

The macOS build is a code-signed, notarized universal binary.

### Windows (Scoop)

Install [Scoop](https://scoop.sh) first, then:

```powershell
scoop bucket add txnlab https://github.com/txnlab/scoop-bucket
scoop install zs-node
```

### Docker

A public, multi-arch (`linux/amd64` + `linux/arm64`) image:

```sh
docker pull ghcr.io/txnlab/zs-node:latest
```

Minimal run — `NODE_SERVER_LISTEN=0.0.0.0:9090` is **required** in a container
(the default `127.0.0.1:9090` is only reachable from the container's own
loopback):

```sh
docker run -d --name zs-node --restart unless-stopped \
  -p 9090:9090 -p 127.0.0.1:9091:9091 \
  -e NODE_SERVER_LISTEN=0.0.0.0:9090 \
  -e NODE_SERVER_PRIVATE_LISTEN=:9091 \
  -e OPERATOR_SIGNING_MNEMONIC="word1 word2 … word25" \
  -e NODE_LLM_OPENAI_API_KEY=sk-... \
  -e NODE_HAYAI_SETTLEMENT_DB_PATH=/app/data/settlement.db \
  -v /etc/zerosignal/config.yaml:/app/config.yaml:ro \
  -v zs-node-data:/app/data \
  ghcr.io/txnlab/zs-node:latest --config /app/config.yaml
```

See the
[Docker install guide](https://docs.zerosignal.ai/operators/installation) for the
full walkthrough (config keys, TLS, and listen overrides).

### Linux / direct download

Grab the archive for your platform from the
[latest release](https://github.com/txnlab/zs-node/releases/latest):

| Platform | Asset |
|---|---|
| Linux | `zs-node_<version>_linux_<arch>.tar.gz` |
| macOS (universal) | `zs-node_<version>_darwin_all.tar.gz` |
| Windows | `zs-node_<version>_windows_<arch>.zip` |

Verify against `checksums.txt`, extract, and put `zs-node` on your `PATH`.

## Configuration

Unlike the client-side proxy, a node **needs a `config.yaml`** — it declares your
inference backend, the models you serve and their pricing, your `operator_id`,
and your algod endpoint. The one secret it refuses to start without is the
**signing mnemonic**, supplied via the environment (never in the YAML):

```sh
# any *_MNEMONIC var is picked up automatically
OPERATOR_SIGNING_MNEMONIC="word1 word2 … word25"
# your upstream LLM key, if you use openai_passthrough
NODE_LLM_OPENAI_API_KEY=sk-...
```

Config keys, provider setup (vLLM / llama.cpp / LM Studio / Vertex AI), pricing,
TLS, and on-chain registration are all covered in the operator guide:
**[docs.zerosignal.ai/operators](https://docs.zerosignal.ai/operators)**.

Check the build with `zs-node version`.

## Run as a service

To start at boot and restart on failure, register an OS service:

```sh
sudo zs-node install-service      # Linux: systemd system unit (needs root)
zs-node install-service           # macOS: per-user launchd LaunchAgent
zs-node install-service --print   # print the generated unit instead of installing
zs-node uninstall-service         # stop + disable + remove
```

The binary writes and enables the unit with its own path and `--config` baked in;
the OS supervisor owns the process from there (`systemctl status zs-node` /
`journalctl -u zs-node -f`). On first Linux run it drops a `0600` secrets template
at `/etc/zerosignal/secrets.env` for your mnemonic, then stops so you can fill it
in before the service starts.

## License

[PolyForm Noncommercial 1.0.0](./LICENSE).

---

Releases are published by [goreleaser](https://goreleaser.com). The macOS binary
is a code-signed, notarized universal binary; the Docker image is multi-arch
(`linux/amd64` + `linux/arm64`).
