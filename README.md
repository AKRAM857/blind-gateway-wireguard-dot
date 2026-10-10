# Zero Trust Remote Administration: WireGuard + DNS-over-TLS with a Private CA

A multi-VM network security lab implementing Zero Trust remote administration. A
remote client reaches a private management network through a WireGuard gateway,
and all DNS resolution is encrypted end-to-end with DNS-over-TLS (DoT) using a
self-hosted Certificate Authority. The gateway routes encrypted queries without
ever seeing their contents.

The project implements the complete path from a remote client to a private
management server, then verifies the security properties of the architecture
through packet captures, port scans, routing analysis, certificate validation,
and controlled attack scenarios.


## What this project actually does

Rather than "configure a VPN and call it done," this project treats security
claims as things to be demonstrated under controlled observation:

- **Architecture**: a WireGuard gateway sits between a remote administrator and
  a private management network, fronted by private DNS resolution with
  end-to-end encryption via DNS-over-TLS, and nftables-enforced access control
  on every hop.
- **Threat model**: two distinct adversaries are considered. First, an observer
  positioned on the untrusted path between client and gateway, representing a
  network-level attacker with no cryptographic material and no path into the
  management network. Second, a compromised gateway — the architecture is
  designed so that even if the gateway is fully compromised, DNS queries remain
  confidential because they are encrypted end-to-end between client and DNS
  server.
- **Verification, not assertion**: controlled experiments use packet capture,
  port scanning, certificate inspection, and negative-case testing (wrong keys,
  unauthorized peers, untrusted certificates) to demonstrate — not just claim —
  confidentiality, peer authentication, PKI integrity, end-to-end DNS privacy,
  and reduced attack surface.


## Why this project

Administrators often manage infrastructure from networks they do not control.
Exposing SSH or other management services directly to those networks increases
the attack surface.

Traditional VPNs protect transport between the client and the VPN gateway, but
require trusting the gateway with all plaintext traffic — including DNS queries.
This project eliminates that trust assumption. By combining WireGuard for
transport with DNS-over-TLS for end-to-end DNS encryption and a self-hosted
Certificate Authority for trust, the architecture guarantees that even a
compromised gateway cannot reveal what the client is resolving.

The goal is not only to build the architecture, but to verify what an observer
on the untrusted network can actually see, what an unauthorized client can and
cannot access, and what a compromised gateway can and cannot learn.


## Lab topology

Multiple VMs on VirtualBox simulating a small enterprise network with an
untrusted "internet" segment, a DMZ, and a segmented internal network:

- **Client**: remote administrator, positioned behind a simulated internet.
- **Router 1**: simulates the untrusted internet; provides NAT for the client.
- **WireGuard Gateway**: VPN endpoint, routing, and firewall (the only device
  exposed to the untrusted path).
- **Router 2**: internal routing; separates the DNS network from the
  management network.
- **DNS Server**: authoritative BIND9 server with DNS-over-TLS (port 853) and a
  certificate signed by the lab's own CA.
- **Certificate Authority**: self-hosted CA (isolated in a Docker container on
  the DNS server), issuing and signing all internal certificates.
- **Management Server**: SSH target on a segmented internal network, reachable
  only through the gateway.

Full topology, addressing plan, and threat model:
[`report/03-lab-architecture`](./report/03-lab-architecture).


## Report structure

| Chapter | Covers |
|---|---|
| 1 — Introduction | Motivation, problem statement, objectives, Zero Trust framing |
| 2 — Theory | L2/L3 separation, underlay/overlay, tunneling, WireGuard internals, TLS, PKI, X.509, DNS-over-TLS |
| 3 — Lab Architecture | VM roles, threat model (external observer and compromised gateway), addressing plan |
| 4 — Network & DNS | VirtualBox networking, routing, NAT, private DNS zone, BIND9 zones |
| 5 — Certificate Authority | Root CA design, key protection, CSR lifecycle, certificate signing, chain of trust |
| 6 — WireGuard Implementation | Keys, peers, tunnel configuration, gateway routing, AllowedIPs semantics |
| 7 — DNS-over-TLS | BIND9 DoT configuration, certificate installation, permissions, AppArmor integration, firewall rules |
| 8 — Firewall Integration | nftables rules on gateway, DNS server, and management server; policy design |
| 9 — Verification Experiments | Full set of experiments (see below), with evidence |
| 10 — Analysis | What was proven vs. explicitly out of scope, limitations, comparison to alternative VPN and Zero Trust architectures |


## Experiments

### Network and transport

- Baseline plaintext capture (what an observer sees without a tunnel)
- WireGuard tunnel opacity (encrypted payload analysis)
- SSH end-to-end trace through the tunnel
- DNS-through-tunnel visibility (what the gateway sees)
- Port scan: exposed services vs. gated services
- Unauthorized peer rejection (peer without valid key)
- Incorrect server key rejection (client with wrong server public key)
- AllowedIPs routing behavior (route selection on the client)
- Routing table diffing (before/after tunnel establishment)
- Packet structure and encapsulation analysis (WireGuard headers)

### PKI and certificate validation

- CA certificate inspection (subject, issuer, extensions, self-signature)
- Server certificate inspection (subject, SAN, key usage, signature chain)
- Certificate chain verification against the CA
- Rejection of untrusted certificates (client without CA trust)
- Rejection of certificates with mismatched SAN (wrong hostname)
- Rejection of expired or invalid certificates

### DNS-over-TLS (Zero Trust proof)

- Plain DNS capture (query visible in plaintext on the wire)
- DoT capture (query encrypted end-to-end)
- Gateway cannot read DoT traffic (proof: DNS query content is not recoverable
  from any packet captured at the gateway)
- DoT works end-to-end through the WireGuard tunnel
- DoT fails without the tunnel (negative case)
- DoT fails without the correct CA trust (negative case)

Evidence (pcaps, screenshots, notes) for each:
[`experiments/`](./experiments).


## Status
🚧 Work in progress — currently in the front-matter / Chapter 1 stage.


## License
TBD
