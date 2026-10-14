# RustFS on AWS: architecture reference

Design rationale for a RustFS deployment on AWS: a two-node erasure-coded
cluster spread across availability zones, fronted by nginx terminating TLS on
443, with a single-node deployment in a second AWS region acting as a
cross-region disaster-recovery target over VPC peering.


## Topology

```
                          clients
                             │ HTTPS :443
              ┌──────────────▼───────────────┐
              │ NLB  :443  (TCP passthrough) │
              │ HTTPS health check /health   │
              └──────────────┬───────────────┘
   region: us-east-1  ┌──────┴───────┐  vpc 10.40.0.0/16
      ┌───────────────▼──┐        ┌──▼───────────────┐
      │ az-a   node1     │        │ az-b   node2     │
      │ nginx :443 ──┐   │        │ nginx :443 ──┐   │
      │ rustfs :9000 ◄┘  │◄──────►│ rustfs :9000 ◄┘  │
      │ console 127.0.0.1│  :9000 │ console 127.0.0.1│
      │ 4 × 20 GiB gp3   │   RPC  │ 4 × 20 GiB gp3   │
      └──────────────────┘        └─────────┬────────┘
            one pool · one erasure set · 8 drives
                                            │
                      ╔═════════════════════▼═══════════╗
                      ║  inter-region VPC peering       ║
                      ║  async site replication :9000   ║
                      ╚═════════════════════╤═══════════╝
   region: us-west-2                        │  vpc 10.50.0.0/16
                              ┌─────────────▼────────────┐
                              │ az-a   dr1               │
                              │ nginx :443               │
                              │ 4 × 20 GiB gp3           │
                              │ separate deployment      │
                              └──────────────────────────┘
```

| | Primary cluster | DR deployment |
|---|---|---|
| Region | us-east-1 | us-west-2 |
| VPC CIDR | 10.40.0.0/16 | 10.50.0.0/16 |
| Instances | 2, one per AZ | 1 |
| Volumes | 4 × 20 GiB per node | 4 × 20 GiB |
| Erasure sets | 1 | 1 |
| Erasure set width | 8 | 4 |
| Parity | `EC:1` | `EC:1` |
| Raw / usable | 160 / 140 GiB | 80 / 60 GiB |
| Shards per node | 4 | 4 |
| Client access | nginx 443 behind NLB | nginx 443 |

## Decision summary

| Decision | Rationale |
|---|---|
| Four separate EBS volumes per node | Erasure coding assumes independent drive failure. RustFS rejects data paths that share a physical device. |
| `EC:1` parity | Maximises usable capacity at 87.5%. Removes node-level fault tolerance within the cluster. |
| nginx terminating TLS on 443 | One client-facing port, real certificates, no plaintext S3 traffic. Removes 9001 from the network entirely. |
| Console bound to `127.0.0.1` | Reachable only through nginx. The admin interface is never directly exposed. |
| RustFS S3 port left on `:9000` | Internode RPC shares the S3 port, so it cannot be bound to loopback. |
| NLB passthrough on 443 | Layer 4 to nginx, which terminates. Keeps certificate and routing logic on the nodes. |
| DR node in a second region | Cross-region latency is incompatible with synchronous erasure coding but acceptable for asynchronous replication. |
| Inter-region VPC peering | Private, backbone-only path between deployments. No public exposure of the replication channel. |

## TLS termination and the reverse proxy

Port 9001 is not exposed. nginx runs on each node, listens on 443, terminates
TLS, and proxies to RustFS on loopback.

### What stays on the private network

| Port | Binding | Reachable from |
|---|---|---|
| 443 | All interfaces | NLB, admin CIDR |
| 9000 | All interfaces | Peer nodes, DR VPC CIDR |
| 9001 | `127.0.0.1` | nginx only |

`RUSTFS_CONSOLE_ADDRESS` can and should be `127.0.0.1:9001`.

The NLB health check uses HTTPS against `/health` on 443, which RustFS exposes
unauthenticated and nginx passes through. This distinguishes a node that is
listening from one that is ready.

## Erasure configuration

One erasure set of 8 drives, `EC:1`:

- 7 data shards, 1 parity shard
- 140 GiB usable from 160 GiB raw (87.5%)
- `RUSTFS_ERASURE_SET_DRIVE_COUNT=8` pins the width so one set spans both nodes
- `RUSTFS_STORAGE_CLASS_STANDARD=EC:1`

## Cross-region disaster recovery

The DR node is an independent deployment in us-west-2 with its own drives,
format and deployment ID, receiving asynchronous site replication.


### VPC peering requirements

| Requirement | Detail |
|---|---|
| Non-overlapping CIDRs | 10.40.0.0/16 and 10.50.0.0/16 |
| Routes in both VPCs | Each route table needs an entry for the peer CIDR via the peering connection |
| Security groups by CIDR | **Cross-region peering does not support security group references.** Rules must use the peer VPC CIDR. |
| DNS resolution | Enable on the peering connection if resolving peer hostnames |
| Connectivity | Port 9000 bidirectionally — replication initiates from both deployments |

The security group constraint is the one that catches people. Within a region
you can permit a peer security group by ID; across regions you cannot, and the
rule must name the peer CIDR instead.

Inter-region peering traffic stays on the AWS backbone and is encrypted in
transit. Cross-region data transfer is billed.

### What replication does and does not give

It copies buckets, object versions, delete markers, and IAM state. It is
asynchronous, so a successful write at the primary does not confirm the DR site
holds that version.

It provides no DNS failover, no traffic routing, and no recovery
orchestration. Promoting the DR site is a separate, manual concern.

Bucket versioning is required at every participating site.

### Capacity

The DR deployment holds 60 GiB usable against the primary's 140 GiB. It cannot
hold a full replica of a filled primary. Either restrict replication to
selected buckets or size the DR volumes for the data they are intended to
receive — a target that runs out of space is not visible from the primary's
status output until replication begins to lag.

## Scope and status

RustFS marks distributed mode, lifecycle management and the built-in KMS as
under testing. The topology described here is conventional — the same design
used by MinIO and other erasure-coded object stores. The risk carried is
implementation maturity in the coordination and recovery paths, not the
architecture.

Versions referenced: RustFS 1.0.1, `rc` 0.1.30.
