# K3s Homelab

A GitOps-managed K3s deployment running on a Proxmox VM. Infrastructure and apps are organized into two root Flux kustomizations (`infrastructure` and `apps`), which compose everything else.

```
homelab/
├── k3s/                    # Flux bootstrap & root kustomizations
├── apps/                   # Application deployments (Immich, Jellyfin, Arr suite, etc.)
└── infrastructure/
    ├── controllers/        # NFS provisioner
    └── operators/          # Cert-manager, CNPG, MetalLB, NVIDIA, Redis, etc.
```

---

## Hardware

**Host:** Proxmox, running K3s as a VM alongside other VMs.

**GPU:** NVIDIA RTX 4070 Super via PCIe passthrough to the K3s VM. The `nvidia-device-plugin` operator manages GPU access with time-slicing configured for 4 concurrent workloads — used by Immich (ML inference) and Jellyfin (hardware transcoding).

**Storage:** An HBA card connects SATA HDDs to the host via PCIe, with the HBA passed through to the K3s VM for direct disk access. NFS shares from a TrueNAS host are attached for large shared volumes (media, downloads, Immich library).

---

## Infrastructure

### GitOps — Flux CD
All state is managed declaratively via Flux. Changes committed to this repo are automatically reconciled to the cluster. Flux bootstraps from `k3s/`, which pulls in the two root kustomizations.

### Storage
Two storage classes:
- **`local-path`** (K3s built-in) — used for smaller DBs and config PVCs that need quick storage via NVMe drive
- **`nfs`** (default, via `nfs-subdir-external-provisioner`) — dynamic provisioning backed by TrueNAS (`/mnt/tank/k3s`)

Large volumes (media, downloads) use static PVs pointing directly to NFS paths.

### Networking
- **MetalLB** — bare-metal load balancer, allocates IPs from a VLAN 10 pool
- **Traefik** — built-in K3s ingress controller, using `IngressRoute` CRDs; configured for cross-namespace routing
- **Cert-Manager** — wildcard TLS certs for `*.dozydelta.com` via Let's Encrypt + Cloudflare DNS challenge
- **Tailscale Operator** — external access for personal and family use without opening ports

### Secrets — Sealed Secrets
Using [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets). Secrets are encrypted with the cluster's public key and safe to commit to git. The controller decrypts them at runtime. Encrypted secrets live as `SealedSecret` manifests alongside the resources that consume them.

### Databases — CloudNative PG
The `cloudnative-pg` operator manages PostgreSQL clusters. Beats bundling Postgres into individual Helm charts — gives proper lifecycle management, backups, and extensions without manual DB ops. Spin up a new `Cluster` resource when an app needs a DB.

### Redis
A single Redis master (via Redis operator) available cluster-wide at `redis-master.redis.svc.cluster.local:6379`. Currently only Authentik uses it. No persistence configured used for session/cache only.

### Other Operators
- **Reflector** — mirrors secrets/configmaps across namespaces (useful for wildcard certs)
- **NVIDIA Device Plugin** — exposes GPU resources with time-slicing

---

## Apps

### [Immich](https://immich.app) — Photo Management
Self-hosted Google Photos replacement. Backed by a CloudNative PG cluster with `pgvecto.rs` for vector search (AI-powered search and face recognition). GPU-accelerated ML. Library stored on NFS (`10Ti` PVC).

### [Jellyfin](https://jellyfin.org) — Media Server
Local media streaming. GPU passthrough for hardware-accelerated transcoding. Media library on NFS (`15Ti` PVC).

### Arr Suite — Media Automation
The usual stack for acquiring Linux ISOs:
- **Prowlarr** — indexer manager/proxy
- **Sonarr** — TV series automation
- **Radarr** — movie automation
- **FlareSolverr** — Cloudflare scrape bypass for indexers

All protected behind an Authentik forward-auth middleware.

### [qBittorrent](https://www.qbittorrent.org) — Torrent Client
Runs with a [Gluetun](https://github.com/qdm12/gluetun) sidecar for ProtonVPN (Wireguard). All torrent traffic exits through the VPN. Firewall rules in Gluetun allow cluster subnets through for local access. No public ingress, internal only.

### [Authentik](https://goauthentik.io) — SSO / Identity Provider
Self-hosted OIDC provider. Two usage patterns:
1. **Native OIDC** — apps that support it get proper SSO (e.g., Immich)
2. **Forward Auth Outpost** — apps that don't get Authentik dropped in front of them via Traefik middleware (e.g., Arr suite)

### [Dozynet](https://github.com/mikegio27/dozynet) — Dashboard
Custom homepage for service links. Lightweight (`ghcr.io/mikegio27/dozynet`).

---

## Access

| Layer | Tool | Purpose |
|---|---|---|
| VPN | Tailscale | External access for self and family |
| SSO | Authentik | OIDC and forward-auth for apps |
| TLS | Cert-Manager + Cloudflare | Wildcard certs for `*.dozydelta.com` |
| Ingress | Traefik | Routing and middleware |
