# Zero Trust Remote Administration

**WireGuard tunneling, DNS-over-TLS, and a self-hosted PKI: secure remote access to a private network with end-to-end DNS privacy, verified by 22 packet-level experiments.**

---

## What this project demonstrates

- **Reduced attack surface.** The WireGuard gateway is the only device reachable from the untrusted network, and it answers only to cryptographically authenticated peers.
- **Gated management access.** SSH to the management target is reachable only through the WireGuard tunnel and is never exposed directly.
- **Self-hosted PKI.** A Root CA, isolated in a Docker container, issues the X.509 certificate used by the DNS server. No public CA is involved.
- **End-to-end DNS privacy.** DNS queries are encrypted with DNS-over-TLS between the client and the DNS server. The WireGuard gateway forwards them but cannot read them.
- **Layered access control.** nftables policies with default-deny rules on the gateway, the DNS server, and both routers produce a defined access control matrix.
- **Evidence over configuration.** Each security claim is tested by an experiment that produces a captured artifact (pcap, screenshot, or command output).

---

## Architecture overview

The lab runs on VirtualBox and consists of six nodes across five networks.

| Network | Subnet | Members |
|---|---|---|
| Client LAN | `192.168.10.0/24` | Client `.12`, Router 1 (LAN) `.1` |
| DMZ | `192.168.11.0/24` | Router 1 (WAN) `.1`, Gateway (public) `.2` |
| Internal | `192.168.20.0/24` | Router 2 (WAN) `.1`, Gateway (internal) `.2` |
| DNS network | `192.168.30.0/24` | Router 2 (LAN) `.1`, DNS server `.2` |
| WireGuard overlay | `10.0.0.0/29` | Gateway `10.0.0.1`, Client `10.0.0.5` |

Router 1 simulates the untrusted internet and provides NAT. The WireGuard gateway is the only device exposed to that side. The DNS server runs BIND9 with DNS-over-TLS on port 853 and also hosts the management target. The Certificate Authority runs as a Docker container on the DNS server.

![Network topology](diagrams/png/01-network-topology.png)

*Figure 1: Physical topology (`diagrams/png/01-network-topology.png`). The logical roles and trust boundaries are shown in `diagrams/png/02-logical-architecture.png`.*

---

## The Zero Trust property

No node trusts another because of its network position. Every connection is authenticated cryptographically, and DNS queries are protected by a TLS session between the client and the DNS server. Because that session terminates at the DNS server, the WireGuard gateway has no key material with which to read it, even after it decrypts the tunnel.

![DNS query lifecycle](diagrams/png/04-data-flow-lifecycle.png)

*Figure 2: End-to-end DNS query lifecycle through the tunnel and DoT (`diagrams/png/04-data-flow-lifecycle.png`).*

Two adversaries are considered (`diagrams/png/03-threat-model.png`):

| Adversary | Can observe | Cannot observe |
|---|---|---|
| External observer on `192.168.11.0/24` | All WireGuard packets (outer headers, sizes, timing) | Tunnel contents, DNS query contents |
| Compromised WireGuard gateway | Decrypted tunnel payload, connection metadata | DNS query contents (TLS terminates at the DNS server) |

Endpoint compromise, private key theft, and operation at scale are out of scope.

---

## Repository structure

```
zero-trust-remote-administration/
├── README.md
├── LICENSE
├── docs/            Report, 10 chapters (Markdown)
├── diagrams/
│   ├── src/         6 draw.io source files
│   └── png/         6 exported diagrams
├── configs/
│   ├── wireguard/   Client and gateway wg0.conf
│   ├── bind9/       named.conf.options, named.conf.local, zone files
│   ├── nftables/    Rulesets for gateway, DNS server, Router 1, Router 2
│   ├── openssl/     ca.cnf, server-san.cnf
│   ├── apparmor/    usr.sbin.named profile
│   └── systemd/     Custom unit files, where used
├── scripts/         Helper scripts (certificate signing, rotation)
└── experiments/     22 subfolders: pcaps, screenshots, notes
```

---

## Verification evidence

The `experiments/` directory contains one subfolder per experiment, each with the raw artifacts that support its conclusion.

| # | Group | Experiment |
|---|---|---|
| 1–10 | Network and transport | Plaintext baseline, tunnel opacity, SSH trace, DNS visibility, port scan, unauthorized peer, wrong server key, AllowedIPs, routing table diff, packet structure |
| 11–16 | PKI | CA and server certificate inspection, chain verification, rejection of untrusted, mismatched-SAN, and expired certificates |
| 17–22 | DNS-over-TLS | Plain DNS vs DoT capture, gateway opacity, DoT through the tunnel, DoT without the tunnel, DoT without CA trust |

Each experiment is documented with its purpose, method, raw output, interpretation, and conclusion in [`docs/09-verification-experiments.md`](docs/09-verification-experiments.md).

---

## Report

The full technical report is in [`docs/`](docs/):

1. [Introduction](docs/01-introduction.md)
2. [Theoretical Foundations](docs/02-theory.md)
3. [Lab Architecture](docs/03-lab-architecture.md)
4. [Network and DNS Layer](docs/04-network-and-dns.md)
5. [Certificate Authority](docs/05-certificate-authority.md)
6. [WireGuard Implementation](docs/06-wireguard-implementation.md)
7. [DNS-over-TLS](docs/07-dns-over-tls.md)
8. [Firewall Integration](docs/08-firewall-integration.md)
9. [Verification Experiments](docs/09-verification-experiments.md)
10. [Analysis](docs/10-analysis.md)

---

## Requirements

- VirtualBox, with enough resources to run six Ubuntu Server VMs
- Ubuntu Server for all nodes
- WireGuard, BIND9 (9.20), nftables, AppArmor, systemd
- OpenSSL and Docker (Certificate Authority)
- tcpdump, Wireshark, kdig, nmap (verification)

To reproduce the lab, follow the chapters in order: network and DNS (4), Certificate Authority (5), WireGuard (6), DNS-over-TLS (7), firewall (8). Reference configurations for each are in `configs/`.

---

## License

See [LICENSE](LICENSE).
