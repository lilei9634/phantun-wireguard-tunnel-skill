---
name: phantun-wireguard-tunnel
description: Deploy a Phantun (UDP→fake-TCP obfuscator) + WireGuard tunnel between an OpenWrt router (acting as proxy gateway with daed/OpenClash/sing-box) and a Debian/Ubuntu VPS. Use when the user asks to set up such a tunnel, when their ISP throttles or blocks UDP, when WireGuard handshake is unstable due to QoS, or when they want to disguise a self-hosted WG egress as TCP. Bridges the WG endpoint into common OpenWrt-side proxies.
---

# Phantun + WireGuard Tunnel: OpenWrt Router ⇄ Linux VPS

A battle-tested deployment guide covering the **non-obvious gotchas** that consume hours when followed naively from upstream READMEs. Reference plan file: `~/.claude/plans/openwrt-daed-vps-debian13-phantun-wireg-wise-bird.md`.

## When to use this skill

The user wants to:
- Build a WG tunnel where the WG UDP is wrapped as fake TCP (Phantun) to bypass ISP UDP QoS / blocking
- Self-host a WG egress on a Linux VPS reachable from a Chinese-network OpenWrt router
- Bridge that WG endpoint into a proxy framework on OpenWrt: **daed** (via sing-box SOCKS5 because daed lacks WG outbound), **OpenClash / Mihomo** (native WG support), or **sing-box** alone (transparent proxy)
- Diagnose an existing phantun+WG setup that "should work but doesn't"

## Architecture

```
LAN client ──► OpenWrt proxy framework (daed/OpenClash/sing-box)
                 ↓ as WG client
              UDP 127.0.0.1:4567  ──► phantun-client (TUN + raw socket)
                                        ↓ fake TCP packet
                                      pppoe-wan / eth-wan
                                        ↓ encrypted-looking TCP
                                      VPS public IP : 4567
                                        ↓
                                      phantun-server (TUN + raw socket)
                                        ↓ decapsulated UDP
                                      127.0.0.1 : 51820
                                        ↓
                                      WireGuard kernel wg0  (10.200.0.1)
                                        ↓ NAT MASQUERADE
                                      Internet
```

**Layering rationale**: Phantun does only UDP↔TCP shape-shifting (no encryption). WG provides crypto/auth. Two WG keypairs are needed (server + client). One Phantun keypair is implicit (Phantun has no auth).

## Critical gotchas — read this section first, every time

These five issues caused the longest debugging cycles:

### G1. VPS phantun-server REQUIRES a DNAT rule

`phantun-server` reads packets from its TUN interface, **not** from the eth0 raw socket directly. Without DNAT, incoming TCP SYNs on eth0:4567 never reach phantun-server, no SYN-ACK is ever generated, and connections silently time out. **You will see SYNs in `tcpdump -i eth0` but no replies, and journalctl will be silent — no error.**

Required rule (substitute your phantun TCP port and tun-peer IP):
```bash
iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 4567 -j DNAT --to-destination 192.168.200.2
```

`192.168.200.2` is `phantun-server --tun-peer` (default). Confirm via `phantun-server --help`.

**Misleading observation**: Testing the VPS port from a Windows machine via `bash /dev/tcp/<vps>/4567` may show "connection succeeded" even when phantun is broken — middlebox TCP optimization on the *client* side fakes a SYN-ACK locally. Always verify the SYN-ACK is actually emitted from VPS via `tcpdump -i eth0`.

### G2. OpenWrt 24.10+ uses APK and fw4 (nftables) — old guides break

- `opkg` is gone, use `apk add <pkg>`
- Default firewall is **fw4** (nftables `inet fw4` table), policy is `forward DROP`
- iptables-nft compat layer works but **fw4 reload wipes nft rules in the `inet fw4` table** (e.g. our `nft insert rule inet fw4 forward iifname "tun-ph" accept` was lost on every firewall reload)
- iptables rules in legacy `ip filter` / `ip nat` tables (e.g. `OUTPUT` chain RST drop, MASQUERADE in `POSTROUTING`) DO survive fw4 reload (different table)

**Fix**: register the phantun TUN as a UCI firewall zone so fw4 includes it natively:
```sh
uci set firewall.phantun=zone
uci set firewall.phantun.name='phantun'
uci set firewall.phantun.input='ACCEPT'
uci set firewall.phantun.output='ACCEPT'
uci set firewall.phantun.forward='ACCEPT'
uci set firewall.phantun.mtu_fix='1'
uci add_list firewall.phantun.device='tun-ph'

uci set firewall.phantun_to_wan=forwarding
uci set firewall.phantun_to_wan.src='phantun'
uci set firewall.phantun_to_wan.dest='wan'
uci commit firewall && /etc/init.d/firewall reload
```

The wan zone's existing `masq=1` will NAT outgoing traffic — no need for a separate iptables MASQUERADE.

OpenWrt may also need `apk add kmod-ipt-nat` so iptables-nft `MASQUERADE` target works (only relevant if you don't use the UCI zone approach above).

### G3. Phantun OpenWrt binary must be musl, not gnu

OpenWrt uses musl libc. The glibc binary fails with `not found` (missing dynamic loader). Always use `phantun_x86_64-unknown-linux-musl.zip` (or the `aarch64-...-musl` variant for ARM routers). VPS Debian/Ubuntu can use either gnu or musl.

Verify: after install, `/usr/bin/phantun-client --help | head -2` should print Usage. If it prints `not found`, you grabbed the wrong build.

### G4. Phantun release zip filenames are NOT `client` / `server`

The zip extracts to `phantun_client` and `phantun_server` (with underscore prefix). Old guides that say `mv client /usr/local/bin/...` will fail. Use the actual filenames.

Also: **busybox does not ship `install`**. Use `cp + chmod 755` on OpenWrt:
```sh
cp phantun_client /usr/bin/phantun-client && chmod 755 /usr/bin/phantun-client
```

### G5. `dial_mode` semantics in dae/daed

The dial_mode setting controls whether dae passes IP or hostname to upstream proxies. It affects whether the upstream proxy needs its own DNS resolver:

| `dial_mode` | What dae sends to proxy | Sniffing | Domain rules work? | Upstream needs DNS? |
|---|---|---|---|---|
| `ip` | IP only | OFF | ❌ | ❌ |
| `domain` | hostname | ON | ✅ | ✅ |
| `domain+` | hostname (more aggressive sniff) | ON | ✅ | ✅ |
| `domain++` | hostname (force) | ON | ✅ | ✅ |

**Common confusion**: `domain+` does NOT pass IP — only `ip` does. If you want to remove sing-box's DNS module, you must switch dae to `dial_mode: ip` AND convert all `domain(...)` routing rules to `dip(...)` IP-based rules. Usually not worth it; keep upstream DNS.

## Phase A — VPS (Debian 13 / Ubuntu)

### A1. Probe environment first

```bash
ssh root@<vps>
cat /etc/os-release | head -2
ip route | head -1                   # note the WAN interface (typically eth0)
ss -ltnp | grep -E ':(4567|51820)'   # ensure ports free
sysctl net.ipv4.ip_forward
```

### A2. Install + enable forwarding

```bash
apt update && apt install -y wireguard-tools unzip iptables-persistent
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-wg-phantun.conf
sysctl --system
```

### A3. Generate WG keypair, fetch Phantun

```bash
umask 077; mkdir -p /etc/wireguard && cd /etc/wireguard
wg genkey | tee server.key | wg pubkey > server.pub
# Decide: client keypair generated on VPS (convenient) or on OpenWrt (more secure)
wg genkey | tee client.key | wg pubkey > client.pub

cd /tmp
wget https://github.com/dndx/phantun/releases/download/v0.8.1/phantun_x86_64-unknown-linux-gnu.zip -O p.zip
unzip p.zip
install -m 0755 phantun_server /usr/local/bin/phantun-server
install -m 0755 phantun_client /usr/local/bin/phantun-client
rm -f p.zip phantun_server phantun_client
phantun-server --version  # confirms 0.8.1
```

### A4. WireGuard config `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.200.0.1/24
ListenPort = 51820
PrivateKey = <contents of /etc/wireguard/server.key>
MTU = 1300
PostUp   = iptables -A INPUT -p udp --dport 51820 ! -s 127.0.0.1 -j DROP
PostDown = iptables -D INPUT -p udp --dport 51820 ! -s 127.0.0.1 -j DROP
PostUp   = iptables -t nat -A POSTROUTING -s 10.200.0.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.200.0.0/24 -o eth0 -j MASQUERADE
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp   = iptables -A FORWARD -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT

[Peer]
PublicKey = <contents of /etc/wireguard/client.pub>
AllowedIPs = 10.200.0.2/32
PersistentKeepalive = 25
```

The `INPUT DROP` for non-localhost UDP 51820 is critical: without it, attackers could bypass phantun and hit WG directly.

### A5. systemd unit `/etc/systemd/system/phantun-server.service`

```ini
[Unit]
Description=Phantun Server (TCP->UDP obfuscator for WireGuard)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=RUST_LOG=info
# G1: DNAT incoming TCP to TUN peer — REQUIRED, server is silent without this
ExecStartPre=-/usr/sbin/iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 4567 -j DNAT --to-destination 192.168.200.2
# Drop kernel RST that would fight phantun's fake-TCP
ExecStartPre=-/usr/sbin/iptables -A OUTPUT -p tcp --sport 4567 --tcp-flags RST RST -j DROP
# Allow FORWARD on phantun TUN (Debian default FORWARD policy may be DROP)
ExecStartPre=-/usr/sbin/iptables -A FORWARD -i tun0 -j ACCEPT
ExecStartPre=-/usr/sbin/iptables -A FORWARD -o tun0 -j ACCEPT
# MASQUERADE phantun TUN return path
ExecStartPre=-/usr/sbin/iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE
ExecStart=/usr/local/bin/phantun-server --local 4567 --remote 127.0.0.1:51820 --tun tun0 --tun-local 192.168.200.1 --tun-peer 192.168.200.2
ExecStopPost=-/usr/sbin/iptables -t nat -D PREROUTING -p tcp -i eth0 --dport 4567 -j DNAT --to-destination 192.168.200.2
ExecStopPost=-/usr/sbin/iptables -D OUTPUT -p tcp --sport 4567 --tcp-flags RST RST -j DROP
ExecStopPost=-/usr/sbin/iptables -D FORWARD -i tun0 -j ACCEPT
ExecStopPost=-/usr/sbin/iptables -D FORWARD -o tun0 -j ACCEPT
ExecStopPost=-/usr/sbin/iptables -t nat -D POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE
Restart=always
RestartSec=3
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW

[Install]
WantedBy=multi-user.target
```

### A6. Enable, start, persist, verify

```bash
systemctl daemon-reload
systemctl enable --now phantun-server wg-quick@wg0
netfilter-persistent save                    # persist iptables across reboots
wg show                                      # interface up, listening :51820
journalctl -u phantun-server -n 5 --no-pager # should say "Listening on 4567"
iptables -t nat -nvL PREROUTING | grep 4567  # confirm DNAT rule present
```

## Phase B — OpenWrt (modern fork: 24.10+, APK + fw4)

### B1. Probe environment

```sh
ssh root@<openwrt>
cat /etc/openwrt_release | grep DISTRIB_RELEASE   # >= 24.10 means APK + fw4
uname -m                                          # confirm arch (x86_64 / aarch64)
which apk opkg                                    # 24.10+: apk only
nft list table inet fw4 2>/dev/null | head -3     # fw4 active?
df -h /                                           # need ~10 MB
```

### B2. Install dependencies

For OpenWrt 24.10+ with APK:
```sh
apk add unzip wget tcpdump kmod-tun kmod-ipt-nat iptables-nft
```

For older opkg systems: `opkg install ...` with same package names (mostly).

### B3. Install Phantun musl binary

```sh
cd /tmp
wget https://github.com/dndx/phantun/releases/download/v0.8.1/phantun_x86_64-unknown-linux-musl.zip -O p.zip
# (use phantun_aarch64-unknown-linux-musl.zip on ARM)
unzip p.zip
cp phantun_client /usr/bin/phantun-client && chmod 755 /usr/bin/phantun-client
cp phantun_server /usr/bin/phantun-server && chmod 755 /usr/bin/phantun-server
rm -f p.zip phantun_client phantun_server
/usr/bin/phantun-client --version       # 0.8.1
```

### B4. procd init `/etc/init.d/phantun-client`

Substitute `<VPS_IP>` and `<WAN_IF>` (typically `pppoe-wan` for Chinese ISP, or `eth1`/`wan`):

```sh
#!/bin/sh /etc/rc.common
# phantun client: WG UDP -> fake TCP to VPS

USE_PROCD=1
START=85
STOP=15

VPS_IP="<VPS_IP>"
VPS_PORT="4567"
TUN_IF="tun-ph"
TUN_LOCAL="192.168.201.2"
TUN_PEER="192.168.201.1"

start_service() {
    iptables -C OUTPUT -p tcp -d ${VPS_IP} --dport ${VPS_PORT} --tcp-flags RST RST -j DROP 2>/dev/null || \
        iptables -A OUTPUT -p tcp -d ${VPS_IP} --dport ${VPS_PORT} --tcp-flags RST RST -j DROP

    procd_open_instance
    procd_set_param command /usr/bin/phantun-client \
        --local 127.0.0.1:4567 \
        --remote ${VPS_IP}:${VPS_PORT} \
        --tun ${TUN_IF} \
        --tun-local ${TUN_LOCAL} \
        --tun-peer ${TUN_PEER}
    procd_set_param env RUST_LOG=info
    procd_set_param respawn 3600 5 0
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}

stop_service() {
    iptables -D OUTPUT -p tcp -d ${VPS_IP} --dport ${VPS_PORT} --tcp-flags RST RST -j DROP 2>/dev/null
}
```

```sh
chmod +x /etc/init.d/phantun-client
/etc/init.d/phantun-client enable
/etc/init.d/phantun-client start
ip addr show tun-ph                  # should be UP with 192.168.201.2
logread -e phantun | tail -5         # "Created TUN device tun-ph"
```

### B5. UCI firewall zone (G2 fix)

This is **mandatory** on OpenWrt 24.10+ because fw4's `forward` policy is DROP. Without this, all phantun traffic is dropped silently.

```sh
uci set firewall.phantun=zone
uci set firewall.phantun.name='phantun'
uci set firewall.phantun.input='ACCEPT'
uci set firewall.phantun.output='ACCEPT'
uci set firewall.phantun.forward='ACCEPT'
uci set firewall.phantun.mtu_fix='1'
uci add_list firewall.phantun.device='tun-ph'

uci set firewall.phantun_to_wan=forwarding
uci set firewall.phantun_to_wan.src='phantun'
uci set firewall.phantun_to_wan.dest='wan'
uci commit firewall
/etc/init.d/firewall reload

# verify fw4 picked it up
nft list chain inet fw4 forward | grep tun-ph
```

## Phase C — Bridge into your proxy framework

Pick one based on what's installed. The output of all paths is the same: a transparent proxy that forwards selected traffic via WG.

### C-A: daed (no native WG) → sing-box bridge → WG

daed v1.x lacks WG outbound. Use sing-box as a SOCKS5 ⇄ WG converter that daed treats as a normal SOCKS5 node.

Install sing-box (1.11+ for new endpoint format):
```sh
apk add sing-box
```

Write `/etc/sing-box/config.json`:
```json
{
  "log": { "level": "info", "timestamp": true },
  "dns": {
    "servers": [
      {
        "tag": "remote-doh",
        "type": "https",
        "server": "8.8.8.8",
        "server_port": 443,
        "path": "/dns-query",
        "detour": "wg-ep"
      }
    ],
    "final": "remote-doh",
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "socks",
      "tag": "socks-in",
      "listen": "127.0.0.1",
      "listen_port": 1080
    }
  ],
  "endpoints": [
    {
      "type": "wireguard",
      "tag": "wg-ep",
      "address": ["10.200.0.2/32"],
      "private_key": "<client.key from VPS>",
      "mtu": 1300,
      "peers": [
        {
          "address": "127.0.0.1",
          "port": 4567,
          "public_key": "<server.pub from VPS>",
          "allowed_ips": ["0.0.0.0/0", "::/0"],
          "persistent_keepalive_interval": 25
        }
      ]
    }
  ],
  "route": { "final": "wg-ep" }
}
```

```sh
sing-box check -c /etc/sing-box/config.json
uci set sing-box.main.enabled=1 && uci commit sing-box
/etc/init.d/sing-box enable && /etc/init.d/sing-box start
```

The DNS block is **required** if daed runs in `dial_mode: domain*` (default). Without it, sing-box falls back to system resolver → ISP DNS → poisoned for sensitive domains. The DoH query is itself routed through `wg-ep` (`detour: wg-ep`), so it's encrypted end-to-end.

In daed Web UI: add a **SOCKS5 node** pointing to `socks5://127.0.0.1:1080`, place it in a group, route via routing rules.

If daed's main routing has any `dip(8.8.8.8, 8.8.4.4) -> proxy`, the daed-side DNS upstream queries also stay clean.

### C-B: OpenClash / Mihomo — native WG outbound

Mihomo (OpenClash's Meta core) supports WireGuard natively since 2022. Skip sing-box entirely.

Edit OpenClash's active config.yaml (LuCI → OpenClash → Configurations → click your config → Edit), or use the **Overwrite Settings → Custom Override** YAML editor:

```yaml
proxies:
  - name: "wg-vps"
    type: wireguard
    server: 127.0.0.1
    port: 4567
    private-key: "<client.key from VPS>"
    public-key: "<server.pub from VPS>"
    ip: 10.200.0.2
    mtu: 1300
    udp: true
    remote-dns-resolve: true
    dns: ['8.8.8.8', '1.1.1.1']

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "wg-vps"
      - DIRECT
```

`remote-dns-resolve: true` makes Mihomo resolve through the WG tunnel (DNS sealed inside tunnel — no leak).

Restart OpenClash. Pick the PROXY group's wg-vps in dashboard.

### C-C: sing-box alone (no daed/OpenClash)

Use sing-box's `tun` inbound for transparent proxy. Out of scope for this skill — see sing-box upstream docs.

## Phase D — verification

```sh
# VPS side
ssh root@<vps> 'wg show && journalctl -u phantun-server -n 10 --no-pager'

# OpenWrt side
ssh root@<openwrt> 'pgrep -a phantun-client && ip addr show tun-ph && \
  iptables -nvL OUTPUT | grep 4567 && \
  nft list chain inet fw4 forward | grep tun-ph'

# End-to-end via SOCKS5 (if sing-box bridge in use)
ssh root@<openwrt> 'curl -s -x socks5h://127.0.0.1:1080 https://ipv4.icanhazip.com'
# Expected: returns the VPS public IP, NOT the ISP IP

# Dual-side packet capture during traffic (for diagnostics)
# On VPS: tcpdump -i eth0 -n "tcp port 4567" -c 20
# On OpenWrt: tcpdump -i pppoe-wan -n "host <vps_ip> and tcp port 4567" -c 20
```

WG handshake confirms by `wg show`'s `latest handshake: <seconds> ago` updating within the 25-second keepalive interval.

## Common diagnostic flow when something is broken

1. **VPS phantun-server has no incoming connections (silent log)**: 
   - Run `iptables -t nat -nvL PREROUTING | grep 4567` → if missing, fix G1 (DNAT)
   - `tcpdump -i eth0 -n 'tcp port 4567'` → if SYNs arrive but no SYN-ACK, definitely G1
2. **OpenWrt phantun-client logs "Sent SYN, Connection closed"**:
   - SYN reaches WAN but no SYN-ACK comes back → see step 1 on VPS, OR ISP/middlebox dropping
   - Verify with parallel tcpdump on both sides
3. **WG handshake never completes (`wg show` stays at "(none)" for handshake)**:
   - Phantun link broken OR WG keys mismatched OR sing-box/Mihomo config typo
   - Check sing-box log: `logread -e sing-box | tail -20`, look for `endpoint/wireguard[wg-ep]` errors
4. **DNS resolves to wrong IPs (Facebook range for YouTube CDN)**:
   - Upstream DNS is being intercepted/poisoned. Add DoH detour through tunnel (see C-A DNS block)
   - Or set `remote-dns-resolve: true` in Mihomo proxy config
5. **fw4 reload kills connectivity**:
   - You added `nft insert rule inet fw4 forward ... accept` at runtime instead of UCI zone (G2). Use the UCI zone approach.

## File locations to remember

| What | Where |
|---|---|
| VPS WG config | `/etc/wireguard/wg0.conf`, keys in same dir |
| VPS phantun service | `/etc/systemd/system/phantun-server.service` |
| VPS persisted iptables | `/etc/iptables/rules.v4` (saved by netfilter-persistent) |
| OpenWrt phantun service | `/etc/init.d/phantun-client` |
| OpenWrt firewall UCI | `/etc/config/firewall` (phantun zone) |
| OpenWrt sing-box config | `/etc/sing-box/config.json` |
| OpenClash config | `/etc/openclash/config/<name>.yaml` |
| daed config | SQLite `/etc/daed/wing.db` (edit via Web UI 2023 port, not text file) |

## What this skill does NOT cover

- Multi-peer WG setups (we assume 1 client : 1 server)
- IPv6 inside the tunnel (sing-box's `wg-ep` `address` field can be extended; needs VPS-side v6 NAT)
- Phantun handshake-packet feature (`--handshake-packet`) for stronger TCP fingerprint disguise
- Switching from sing-box bridge to native daed WG (daed roadmap may add this; check current version)
- Building from source (use upstream prebuilt binaries — check signature)

## Quick reference: parameters used in the canonical example

| Param | Value | Notes |
|---|---|---|
| Phantun TCP port | 4567 | Change to 443/8443 for stronger HTTPS disguise |
| WG UDP port (VPS local only) | 51820 | Bound to `127.0.0.1`, never exposed |
| WG subnet | 10.200.0.0/24 | server `.1`, client `.2` |
| MTU | 1300 | Conservative; prevents PMTU black holes on PPPoE |
| VPS phantun TUN | tun0, peer 192.168.200.2 | `--tun-peer` is the DNAT target (G1) |
| OpenWrt phantun TUN | tun-ph, peer 192.168.201.1 | Distinct subnet from VPS side |
| sing-box SOCKS5 | 127.0.0.1:1080 | Listen local only |
