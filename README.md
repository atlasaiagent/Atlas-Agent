<p align="center">
  <img src="banner.png" alt="AtlasAgent Banner" width="100%" />
</p>

<p align="center">
  <b>High-performance Solana transaction infrastructure — built for agents that never miss a slot.</b>
</p>

<p align="center">
  <a href="#overview">Overview</a> ·
  <a href="#quickstart">Quickstart</a> ·
  <a href="#configuration">Configuration</a> ·
  <a href="#deployment">Deployment</a> ·
  <a href="#repository-structure">Repository Structure</a> ·
  <a href="#contributing">Contributing</a> ·
  <a href="#license">License</a>
</p>

---

## Overview

**AtlasAgent** is a lean, production-grade transaction sender for the Solana network, purpose-built for AI agents and high-frequency pipelines that demand reliability at the edge of the block.

It bypasses standard preflight validation and blockhash roundtrips, forwarding transactions directly to the current slot leaders via TPU connections — keeping latency as low as the network allows.

### Core Capabilities

- **Direct TPU delivery** — sends transactions straight to upcoming slot leaders, skipping the standard RPC relay
- **Yellowstone gRPC streaming** — subscribes to live slot and block events via a Geyser plugin for real-time leader tracking
- **Staked connection support** — optionally uses a validator identity keypair to establish staked TPU connections with higher priority
- **Configurable leader fanout** — tune the number of leaders and connection pool size to trade off reliability vs. network load
- **Confirmation tracking** — uses streamed block data to verify whether submitted transactions landed on-chain
- **Ansible-automated deployment** — ship to production servers with a single playbook; haproxy and Datadog included

### Architecture

```
atlas-txn-sender/
├── src/
│   └── txn_sender.rs        # Core TPU sender logic
├── ansible/
│   ├── inventory/           # Target host configuration
│   ├── roles/
│   │   └── datadog-setup/   # Metrics agent setup
│   └── deploy_atlas_txn_sender.yml
├── Cargo.toml
└── README.md
```

---

## Quickstart

### Prerequisites

Install system dependencies (Ubuntu / Debian):

```bash
sudo apt-get install \
  libssl-dev libudev-dev pkg-config \
  zlib1g-dev llvm clang cmake make \
  libprotobuf-dev protobuf-compiler
```

### Build & Run

```bash
# Clone the repository
git clone https://github.com/your-org/AtlasAgent.git
cd AtlasAgent/atlas-txn-sender

# Set required environment variables
export RPC_URL="https://api.mainnet-beta.solana.com"
export GRPC_URL="your-yellowstone-grpc-endpoint"
export X_TOKEN="your-auth-token"
export PORT=4040

# Build and run in release mode
cargo run --release
```

---

## Configuration

All configuration is handled via environment variables:

| Variable | Required | Default | Description |
|---|---|---|---|
| `RPC_URL` | ✅ | — | RPC endpoint used to fetch upcoming slot leaders via `getSlotLeaders` |
| `GRPC_URL` | ✅ | — | Yellowstone gRPC Geyser URL for streaming live slot and block data |
| `X_TOKEN` | ✅ | — | Authentication token for the gRPC endpoint |
| `TPU_CONNECTION_POOL_SIZE` | ❌ | `4` | Number of leader TPU connections to maintain in the pool |
| `NUM_LEADERS` | ❌ | `4` | Number of upcoming leaders to broadcast each transaction to |
| `LEADER_OFFSET` | ❌ | `0` | Slot offset into the leader schedule to begin targeting |
| `IDENTITY_KEYPAIR_FILE` | ❌ | — | Path to a keypair file; if a validator key is provided, staked connections are used |
| `PORT` | ❌ | `4040` | Port the HTTP service listens on |

> **Note:** AtlasAgent does **not** perform preflight checks or validate blockhashes before forwarding. Ensure your upstream agent handles this if needed.

---

## Deployment

AtlasAgent ships with an Ansible playbook for one-command production deployment. The playbook configures a `systemd` service, wraps it with `haproxy` for port-80 access, and installs the Datadog agent for metrics.

### Step 1 — Install the Datadog Ansible role

```bash
ansible-galaxy install datadog.datadog
```

### Step 2 — Configure your inventory

Edit `ansible/inventory/hosts.yml` with your server's hostname, IP address, and SSH user.

### Step 3 — Set deployment variables

Open `ansible/deploy_atlas_txn_sender.yml` and fill in:

```yaml
rpc_url: "https://..."
grpc_url: "https://..."
x_token: "your-token"
datadog_api_key: "your-dd-api-key"
datadog_site: "datadoghq.com"
```

### Step 4 — Deploy

```bash
ansible-playbook -i ansible/inventory/hosts.yml ansible/deploy_atlas_txn_sender.yml
```

---

## Repository Structure

```
AtlasAgent/
└── atlas-txn-sender/
    ├── src/
    │   └── txn_sender.rs          # TPU connection management & transaction dispatch
    ├── ansible/
    │   ├── inventory/
    │   │   └── hosts.yml          # Server inventory
    │   ├── roles/
    │   │   └── datadog-setup/     # Datadog monitoring role
    │   └── deploy_atlas_txn_sender.yml
    ├── banner.png                 # Project banner
    ├── Cargo.toml
    └── README.md
```

---

## Contributing

Contributions are welcome. Please keep the following in mind:

- Open an issue before submitting large architectural changes
- Submit focused PRs with a clear description of the problem being solved
- Maintain minimal dependencies — AtlasAgent is intentionally lean
- Include performance notes when modifying the TPU connection or leader scheduling logic

Recommended local workflow:

```bash
cargo fmt
cargo clippy -- -D warnings
cargo test
```

---

## Security and Bug Reports

- Report security-sensitive issues privately before public disclosure when possible
- Open public issues for non-sensitive bugs with clear reproduction steps
- Include the affected module, environment variables, expected vs. actual behavior, and any relevant logs

---

## License

This project is licensed under the **MIT License**.

---

## Legal

AtlasAgent is designed for lawful use on networks you are authorized to interact with.

- Respect Solana network rate limits and validator policies
- Review all transaction logic before production deployment
- The authors are not responsible for losses arising from transaction failures, missed blocks, or misconfiguration