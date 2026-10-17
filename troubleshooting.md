# Troubleshooting

Problems met during the build: symptom, cause, fix, and how to check it.

## 1. External names fail to resolve (DNSSEC and clock skew)

- **Symptom:** the private zone resolves, but recursive queries for external names fail.
- **Cause:** the VM clocks drift from real time, so DNSSEC signatures look expired or not yet valid and validation fails.
- **Fix:** `dnssec-validation no;` in `named.conf.options`. Acceptable for this project, not for production.
- **Check:** `dig @127.0.0.1 example.com` returns an answer.

## 2. BIND 9.20 rejects the `tls` block

- **Symptom:** `named-checkconf` fails, or nothing listens on port 853.
- **Cause:** older guides put `tls { }` inside `options { }`. BIND 9.20 requires it outside.
- **Fix:** define `tls local-tls { key-file ...; cert-file ...; };` at the top level, then `listen-on port 853 tls local-tls { any; };` inside `options`.
- **Check:** `named-checkconf`, then `ss -tlnp | grep 853`.

## 3. AppArmor blocks the TLS key

- **Symptom:** DoT does not start although the key is `640 root:bind`.
- **Cause:** the `named` AppArmor profile does not allow reading `/etc/ssl/dns-server/`.
- **Fix:** add `/etc/ssl/dns-server/** r,` to `/etc/apparmor.d/usr.sbin.named`, then `sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.named` and restart `named`.
- **Check:** `sudo journalctl -k | grep -i 'apparmor.*denied'` shows no new denial for `named`.

## 4. Certificate rejected although the CA is trusted

- **Symptom:** the client refuses the server certificate with a name error.
- **Cause:** modern clients check the Subject Alternative Name and ignore the common name.
- **Fix:** put both the name and the address in the SAN (`DNS:dns1.firstzone.akram.com, IP:192.168.30.2`) and re-issue.
- **Check:** `openssl x509 -in server.crt -noout -ext subjectAltName`.

## 5. Endpoint routing loop

- **Symptom:** with a broad `AllowedIPs`, the handshake never completes after the tunnel comes up.
- **Cause:** the route to the gateway's own address was also directed into `wg0`, so encrypted packets were sent back into the interface that produces them.
- **Fix:** keep the route to the endpoint on the physical interface by limiting `AllowedIPs` to the networks that must use the tunnel (`10.0.0.2/32, 192.168.30.0/24`).
- **Check:** `ip route get 192.168.11.2` shows the physical interface, not `wg0`.

## 6. Handshake works but pings get no reply

- **Symptom:** `wg show` reports a recent handshake, yet pings are lost.
- **How to read it:** the counters tell where packets stop. The client had sent 532 B (a 148 B handshake plus three 128 B pings) and the gateway had received 532 B, but had sent only 124 B (a 92 B handshake reply plus a 32 B keepalive). The pings arrived and no reply was ever produced, so the cause was inside the gateway.
- **Cause:** the gateway firewall, which allowed only the WireGuard port and DNS-over-TLS, dropped the ICMP packets arriving on `wg0`.
- **Fix:** accept `icmp type echo-request` from `wg0` in the `input` chain.
- **Check:** `sudo tcpdump -ni wg0 icmp` while pinging, and `sudo nft list ruleset` with counters.

## 7. The gateway shows Router 1 as the peer's endpoint

- **Symptom:** `wg show` on the gateway lists `192.168.11.1`, not the client's `192.168.10.12`.
- **Cause:** not an error. Router 1 translates the client's source address, and WireGuard adopts the address a packet arrives from.
- **Fix:** none. The gateway needs no `Endpoint` line for a client behind NAT.

## 8. Which firewall hook applies

- **Symptom:** rules for `wg0` in `input` and `output` do not affect DNS or SSH through the tunnel.
- **Cause:** traffic that passes through the gateway uses only the `forward` hook. `input` and `output` apply to traffic addressed to or from the gateway itself, such as the outer WireGuard packets on `eth0` and a ping to `10.0.0.2`.
- **Fix:** put policy for tunnel traffic in `forward`, and keep `input` and `output` for the gateway's own traffic.
