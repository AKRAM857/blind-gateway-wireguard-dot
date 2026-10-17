# Verification experiments

Each folder holds `notes.md` (purpose, method, raw output, interpretation, conclusion) and the artifacts: `.pcap` files, screenshots, command output.

Mark the status column when an experiment's artifacts are committed.

## Firewall state needed
- Experiments **1**, **4** (plain DNS part), and **17** need plain traffic to pass: run them before the firewall is tightened, or allow the traffic temporarily.
- Experiments **5** and **21** need the final, tightened rules.

## Network and transport

| # | Experiment | Vantage point and action | Commands | Status |
|---|---|---|---|---|
| 01 | Baseline plaintext | Tunnel down. Capture on the gateway while the client queries and connects | Gateway: `tcpdump -i eth0 -w exp01.pcap not port 51820`<br>Client: `dig @192.168.30.2 web.firstzone.akram.com` | ☐ |
| 02 | Tunnel opacity | Tunnel up. Capture the outer traffic on the gateway | Gateway: `tcpdump -i eth0 -w exp02.pcap udp port 51820`<br>Client: `ping -c 10 10.0.0.1`<br>Analyze: `tshark -r exp02.pcap -Y wg` | ☐ |
| 03 | SSH through the tunnel | Capture inside and outside the tunnel | Client: `ssh user@192.168.30.2`<br>Gateway: `tcpdump -i wg0 -w exp03-wg0.pcap tcp port 22` and `tcpdump -i eth0 -w exp03-eth0.pcap udp port 51820` | ☐ |
| 04 | DNS visibility at the gateway | Capture on `wg0`. Compare plain DNS (allow port 53 temporarily) with DoT | Gateway: `tcpdump -i wg0 -w exp04.pcap "port 53 or port 853"`<br>Client: `kdig +tls @192.168.30.2 ...` | ☐ |
| 05 | Port scan | Tunnel down, tightened rules | From client: `nmap -p 22,53,853 192.168.30.2` | ☐ |
| 06 | Unauthorized peer | A key pair unknown to the gateway tries to connect | Create new keys, use them in a client config. `sudo wg-quick up wg0`, `ping -c 3 10.0.0.1`<br>Gateway: `tcpdump -i eth0 udp port 51820` (packets in, none out) | ☐ |
| 07 | Wrong server key | Client config with an incorrect gateway public key | Edit `PublicKey` in the client config, `sudo wg-quick up wg0`, `ping -c 3 10.0.0.1`, `sudo wg show` | ☐ |
| 08 | AllowedIPs routing | Which destinations use the tunnel | Client: `ip route get 192.168.30.2` (should show wg0) vs `ip route get 8.8.8.8` (should show eth0) | ☐ |
| 09 | Routing table diff | Client before and after the tunnel | Client: `ip route > route-before.txt`, `wg-quick up wg0`, `ip route > route-after.txt`, `diff route-before.txt route-after.txt` | ☐ |
| 10 | Packet structure | Dissect the WireGuard packets from experiment 2 | `tshark -r exp02.pcap -Y wg -V -c 3`<br>`tshark -r exp02.pcap -Y wg -T fields -e frame.len -e wg.type` | ☐ |

## PKI

| # | Experiment | Commands | Status |
|---|---|---|---|
| 11 | CA certificate inspection | `openssl x509 -in /etc/ssl/project/ca.crt -noout -text` | ☐ |
| 12 | Server certificate inspection | `openssl x509 -in /etc/ssl/dns-server/server.crt -noout -text` | ☐ |
| 13 | Chain verification | `openssl verify -CAfile ca.crt server.crt` | ☐ |
| 14 | Untrusted certificate | Create an unrelated CA, then query with it as the only trust anchor:<br>`openssl req -x509 -newkey rsa:2048 -nodes -keyout /tmp/other.key -out other-ca.crt -subj "/CN=Other CA" -days 30`<br>`kdig @192.168.30.2 +tls +tls-ca=other-ca.crt +tls-hostname=dns1.firstzone.akram.com web.firstzone.akram.com` (fails) | ☐ |
| 15 | Mismatched name | `kdig @192.168.30.2 +tls +tls-ca=ca.crt +tls-hostname=wrong.example.com web.firstzone.akram.com` (fails) | ☐ |
| 16 | Expired certificate | `openssl verify -attime $(date -d '+400 days' +%s) -CAfile ca.crt server.crt` (expired) | ☐ |

## DNS-over-TLS

| # | Experiment | Commands | Expected | Status |
|---|---|---|---|---|
| 17 | Plain DNS capture | `tcpdump -i any -w exp17.pcap port 53` while the client runs `dig`<br>Analyze: `tshark -r exp17.pcap -Y dns` | Query name decoded and found in the capture | ☐ |
| 18 | DoT capture | `tcpdump -i any -w exp18.pcap port 853` while the client runs `kdig +tls`<br>Analyze: `tshark -r exp18.pcap -Y 'tls.handshake.type == 2' -V \| grep -i version` | 0 DNS packets decoded; TLS handshake visible; negotiated TLS version recorded | ☐ |
| 19 | Gateway cannot read DoT | Tunnel up. Capture on the gateway during a DoT query<br>`tcpdump -i wg0 -w exp19-wg0.pcap tcp port 853`<br>Analyze: `tshark -r exp19-wg0.pcap -Y dns` | 0 DNS packets, query name not found. Only TLS application data visible. Use exp 17 as the control. | ☐ |
| 20 | DoT through the tunnel | `ip route get 192.168.30.2` (via `wg0`), `kdig @192.168.30.2 +tls +tls-ca=ca.crt +tls-hostname=dns1.firstzone.akram.com web.firstzone.akram.com`<br>Gateway: `tcpdump -i eth0 -w exp20.pcap "port 853 or udp port 51820"` | Answer returned; only UDP/51820 visible outside the tunnel | ☐ |
| 21 | DoT without the tunnel | `sudo wg-quick down wg0`, then the same `kdig` (needs the tightened rules) | Query times out | ☐ |
| 22 | DoT without CA trust | `sudo mv /usr/local/share/ca-certificates/zerotrust-ca.crt /tmp/ && sudo update-ca-certificates --fresh`, then `kdig @192.168.30.2 +tls +tls-ca=ca.crt +tls-hostname=dns1.firstzone.akram.com web.firstzone.akram.com`<br>Restore: `sudo mv /tmp/zerotrust-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates` | Certificate verification error | ☐ |
