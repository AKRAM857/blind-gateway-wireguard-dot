# Configurations

Private keys are replaced by `<private key>`. Files marked "target" are the intended rulesets; the live files are exported with `scripts/tools/collect-configs.sh`.

| File | Machine | Installed at |
|---|---|---|
| `wireguard/gateway-wg0.conf` | Gateway | `/etc/wireguard/wg0.conf` |
| `wireguard/client-wg0.conf` | Client | `/etc/wireguard/wg0.conf` |
| `bind9/named.conf.options` | DNS server | `/etc/bind/named.conf.options` |
| `bind9/named.conf.local` | DNS server | `/etc/bind/named.conf.local` |
| `bind9/zones/db.com.akram.firstzone` | DNS server | `/etc/bind/zones/` |
| `bind9/zones/db.192.168.30` | DNS server | `/etc/bind/zones/` |
| `nftables/gateway.nft` | Gateway | `/etc/nftables.conf` |
| `nftables/router1.nft` | Router 1 | `/etc/nftables.conf` |
| `nftables/router2.nft` | Router 2 | `/etc/nftables.conf` |
| `openssl/server-san.cnf` | DNS server | used when generating the CSR |
| `openssl/server-ext.cnf` | Certificate Authority | used when signing |
| `apparmor/usr.sbin.named.local` | DNS server | rule added to the `named` AppArmor profile |
