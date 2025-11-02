<div align="center">
  <img src="banner.jpg" alt="AtlasAgent Banner" width="100%" />
</div>

# AtlasAgent

> High-performance, low-latency transaction infrastructure for Solana — built for validators, MEV searchers, and protocol teams.

```
Contract Address : 3MRMqTpaH1warLB91obvZhrsERQbfMsDgGydNPx8pump
```

<p align="center">
  <b>High-performance Solana transaction infrastructure — built for agents that never miss a slot.</b>
</p>

<p align="center">
  <a href="#overview">Overview</a> ·
  <a href="#core-capabilities">Core Capabilities</a> ·
  <a href="#quickstart">Quickstart</a> ·
  <a href="#configuration">Configuration</a> ·
  <a href="#deployment">Deployment</a> ·
  <a href="#repository-structure">Repository Structure</a> ·
  <a href="#contributing">Contributing</a> ·
  <a href="#contact">Contact</a> ·
  <a href="#license">License</a>
</p>

---

## Overview

**AtlasAgent** is a lean, production-grade transaction sender for the Solana network, purpose-built for AI agents and high-frequency pipelines that demand reliability at the edge of the block.

It bypasses standard preflight validation and blockhash roundtrips, forwarding transactions directly to the current slot leaders via TPU connections — keeping latency as low as the network allows.

AtlasAgent is built with the following principles:

- **Speed first** — direct TPU routing with zero preflight overhead
- **Reliability** — staked identity connections for validators, pooled leader connections for everyone else
- **Observability** — first-class Datadog metrics and Ansible-managed deployments

---

## Core Capabilities

### Transaction Pipeline
- Direct TPU packet dispatch to upcoming slot leaders
- **Direct TPU delivery** — sends transactions straight to upcoming slot leaders, skipping the standard RPC relay
- Configurable leader look-ahead (`NUM_LEADERS`, `LEADER_OFFSET`)
- Connection pooling per leader (`TPU_CONNECTION_POOL_SIZE`)
- Staked identity support via validator keypair

### Geyser Streaming
- **Yellowstone gRPC streaming** — subscribes to live slot and block events via a Geyser plugin for real-time leader tracking
- Confirmed-block tracking to verify landing success
- Automatic reconnection and stream recovery

### Infrastructure
- HTTP server for transaction submission (`/send_transaction`)
- HAProxy front-end with Ansible-managed systemd deployment
- Datadog integration for latency, throughput, and error telemetry
- **Ansible-automated deployment** — ship to production servers with a single playbook; haproxy and Datadog included

### Architecture

```
atlas-txn-sender/
├── src/
│   ├── main.rs               # Service entry point & HTTP server
│   ├── txn_sender.rs         # Core TPU dispatch logic
│   ├── leader_tracker.rs     # Geyser-backed leader schedule tracker
│   └── ...
├── ansible/
│   ├── inventory/            # Target host configuration
│   ├── roles/
│   │   └── datadog-setup/    # Metrics agent setup
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
git clone https://github.com/atlasaiagent/Atlas-Agent.git
cd Atlas-Agent

# Set required environment variables
export RPC_URL="https://api.mainnet-beta.solana.com"
export GRPC_URL="your-yellowstone-grpc-endpoint"
export X_TOKEN="your-auth-token"
export PORT=4040

# Build and run in release mode
cargo run --release
```

The service will be available at `http://localhost:4040`.

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

Edit `ansible/inventory/hosts.yml` with your server details:

```yaml
all:
  hosts:
    atlas-node-1:
      ansible_host: YOUR_SERVER_IP
      ansible_user: ubuntu
```

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

The playbook will:
- Install all system dependencies
- Build the Rust binary
- Register a `systemd` service
- Configure HAProxy to expose the service on port 80
- Set up Datadog agent with custom metrics

---

## Repository Structure

```
AtlasAgent/
└── atlas-txn-sender/
    ├── src/
    │   ├── main.rs               # Service entry point & HTTP server
    │   ├── txn_sender.rs         # TPU connection management & transaction dispatch
    │   └── ...
    ├── ansible/
    │   ├── inventory/
    │   │   └── hosts.yml         # Server inventory
    │   ├── roles/
    │   │   └── datadog-setup/    # Datadog monitoring role
    │   └── deploy_atlas_txn_sender.yml
    ├── banner.png                # Project banner
    ├── Cargo.toml
    └── README.md
```

---

## Contributing

Contributions are welcome across the core engine, Ansible roles, and observability tooling.

- Open an issue before submitting large architectural changes
- Submit focused PRs with a clear description of the problem being solved
- Maintain minimal dependencies — AtlasAgent is intentionally lean
- Include performance notes when modifying the TPU connection or leader scheduling logic
- Include benchmark deltas for throughput or latency changes

Recommended local workflow:

```bash
cargo fmt
cargo clippy -- -D warnings
cargo test
```

---

## Security and Bug Reports

Security is critical for infrastructure that routes live on-chain transactions.

- Report security-sensitive issues privately before public disclosure when possible
- Open public issues for non-sensitive bugs with clear reproduction steps
- Include the affected module, environment variables, expected vs. actual behavior, and any relevant logs

**Security focus areas:**
- TPU connection authentication
- Keypair file handling and isolation
- gRPC token exposure
- HAProxy surface hardening

---

## Contact

- **GitHub Issues:** [Report bugs or request features](https://github.com/atlasaiagent/Atlas-Agent/issues)

---

## License

This project is licensed under the **MIT License**.

---

## Legal

AtlasAgent is designed for lawful use on networks you are authorized to interact with.

- Respect Solana network rate limits and validator policies
- Review all transaction logic before production deployment
- Add human approval gates for high-risk or high-value transaction flows
- The authors are not responsible for losses arising from transaction failures, missed blocks, or misconfiguration