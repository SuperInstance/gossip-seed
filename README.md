# Gossip Seed

A **bootstrapping library** for gossip protocols that solves the cold-start problem — how does a new node discover existing cluster members when it has no prior contact information? This library provides seed node lists, DNS-based discovery, and persistent contact caches.

## Why It Matters

Every gossip protocol faces the bootstrap problem: a new node needs to know at least one existing member to join the cluster. Hardcoding seed addresses works for small fixed clusters but fails for dynamic environments (auto-scaling, Kubernetes, ephemeral containers). This library implements three complementary discovery mechanisms: (1) static seed lists for known infrastructure, (2) DNS-based discovery for dynamic environments, and (3) persistent local caches that remember peers from previous sessions. The combination ensures that a node can almost always find the cluster — if any one mechanism succeeds, the gossip protocol takes over and learns the full membership.

## How It Works

**Seed list** (static): A configured list of `host:port` addresses for known stable nodes. The new node attempts to contact each in random order until one responds. O(K) attempts where K is the seed list size.

**DNS discovery**: For environments like Kubernetes where pods come and go, the library queries DNS SRV records:
```
_srvc._proto.name TTL class SRV priority weight port target
```
This returns all pods in a headless service. The library polls DNS every T_discover seconds (typically 30s) to refresh the seed list.

**Persistent cache**: On graceful shutdown, the node writes its current member list to disk (`~/.cache/gossip-peers.json`). On restart, it reads this file first — peers from the last session are likely still alive. This reduces cold-start latency from seconds (DNS) to milliseconds (file read).

**Join protocol**:
```
1. Read peer cache (O(1) file read)
2. If cache empty, query DNS SRV records (O(DNS_latency))
3. If DNS empty, fall back to static seed list (O(K))
4. Contact first reachable peer
5. Exchange full membership tables (anti-entropy)
6. Node is now a cluster member
```

**Failure handling**: If all seeds are unreachable, the node starts a singleton cluster and periodically retries discovery. This handles the "first node" case (datacenter bootstrap) and the "total partition" case gracefully.

**Complexity**:
- Peer cache read: O(N) where N is cached peer count (typically <100)
- DNS query: O(1) cached, O(RTT) uncached
- Seed list probe: O(K × RTT) worst case, O(1 × RTT) if first seed responds

## Quick Start

```rust
use gossip_seed::{SeedManager, SeedConfig};

let config = SeedConfig::new()
    .static_seeds(&["10.0.0.1:7946", "10.0.0.2:7946"])
    .dns_name("gossip.default.svc.cluster.local")
    .cache_path("~/.cache/gossip-peers.json");

let mut seeds = SeedManager::new(config);
let peers = seeds.discover().await;

if peers.is_empty() {
    println!("No seeds found — starting singleton cluster");
} else {
    println!("Discovered {} peers", peers.len());
}
```

## API

| Type | Description |
|------|-------------|
| `SeedManager::new(config)` | Create a seed discovery manager |
| `SeedConfig` | Configuration (seeds, DNS name, cache path) |
| `.discover()` | Execute discovery cascade → `Vec<PeerAddr>` |
| `.save_cache(peers)` | Persist peer list to disk |
| `.refresh_dns()` | Force DNS re-query and update seed list |

## Architecture Notes

Gossip Seed is the bootstrap layer of the SuperInstance gossip stack. It solves the initial coordination problem — finding the cluster. In **γ + η = C**, the seed discovery is a one-time γ cost at join time; after that, gossip protocol membership is pure η. The persistent cache minimizes even that γ cost. See [Architecture](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## References

- Ripeanu, M. & Foster, I. "Mapping the Gnutella Network," IEEE Internet Computing (2002). — Peer discovery in P2P.
- Kubernetes Headless Services Documentation. https://kubernetes.io/docs/concepts/services-networking/service/#headless-services
- Hashicorp memberlist: Known Hosts and Join. https://github.com/hashicorp/memberlist

## License

MIT
