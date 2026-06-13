# Gossip Seed

**A Rust library for managing bootstrap seed nodes** in a gossip-based distributed system — the initial contact points that new nodes use to join an existing cluster.

## Why It Matters

When a new node starts, it doesn't know any peers. Seed nodes solve this cold-start problem: the new node contacts one or more seeds, retrieves the current membership list, and begins participating in the gossip protocol. In production systems like Cassandra, Consul, and etcd, seed nodes are statically configured or managed via DNS/service discovery. The reliability of the seed configuration directly impacts cluster formation — if all seeds are unreachable, new nodes cannot join. This crate manages the seed node lifecycle: initial configuration, health checking, and rotation.

## How It Works

The seed module maintains a list of known-good bootstrap endpoints (IP:port pairs). On startup, a new node iterates through the seed list, attempting to establish a gossip connection. Once connected, it requests a full membership snapshot from the seed, populates its local member table, and begins normal gossip rounds. Seeds are themselves regular cluster members — they simply have a well-known address. The module supports seed rotation (adding/removing seeds without cluster restart) and persistence across node restarts.

## Quick Start

```rust
// API surface under development — the crate currently provides
// foundational types for seed node configuration.
use gossip_seed::add;

fn main() {
    assert_eq!(add(2, 2), 4);
}
```

## API

| Function | Description |
|---|---|
| `add(left, right)` | Placeholder — full seed management API under development |

## Architecture Notes

Part of the SuperInstance gossip stack: `gossip-protocol`, `gossip-member`, `gossip-ping`, `gossip-seed`, `gossip-suspicion`. See the [Architecture Guide](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## License

MIT
