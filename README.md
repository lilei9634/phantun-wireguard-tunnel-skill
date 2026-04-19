# phantun-wireguard-tunnel (Claude Code Skill)

A [Claude Code](https://claude.com/claude-code) skill that walks Claude through deploying a **Phantun + WireGuard** tunnel between a home router (**OpenWrt** or **MikroTik RouterOS 7.4+ with container**) and a Debian/Ubuntu VPS.

## What this is for

ISP throttles or blocks UDP? Self-hosting a WireGuard egress that needs to look like TCP? Want phantun-client running inside a RouterOS container instead of buying a dedicated OpenWrt device? Want multiple parallel tunnels to different VPSs? This skill encodes a battle-tested deployment flow plus the **non-obvious gotchas** that consume hours when you follow upstream READMEs naively:

- VPS-side `iptables -t nat ... DNAT` is **mandatory** (phantun-server reads from TUN, not eth0 — silent failure otherwise)
- OpenWrt 24.10+ uses APK + fw4(nftables); the forward chain DROP policy needs a UCI zone, not a runtime `nft insert`
- Phantun binary must be **musl** on both OpenWrt and RouterOS (gnu fails with `not found`)
- `dial_mode` semantics: only `ip` mode passes IP to upstream proxy — `domain+` still passes hostname, contrary to common misreading
- DNS poisoning paths and where to insert DoH detours
- **RouterOS-specific**: must add `/ip/route/add dst-address=<TUN_PEER-subnet>/24 gateway=<container-LAN-IP>` or phantun silently stuck in `syn-recv` forever
- **RouterOS-specific**: OUTPUT DROP RST iptables rule lives in the container's netns, not the host — your image **must** include iptables
- **Mihomo + WG**: `remote-dns-resolve: true` + `dns:` is required on the WG proxy block (unlike HY2/VMess/Trojan), because WG is L3 and can't pass domains to the server

## Architecture

```
LAN client → OpenWrt (daed / OpenClash / sing-box transparent proxy)
                ↓
            127.0.0.1:4567 UDP
                ↓
            phantun-client  ──[fake TCP]──►  VPS:4567
                                                 ↓
                                            phantun-server
                                                 ↓
                                            127.0.0.1:51820 UDP
                                                 ↓
                                            wg0  →  Internet egress
```

## Install

Drop this directory into your Claude Code skills folder:

```bash
git clone https://github.com/lilei9634/phantun-wireguard-tunnel-skill ~/.claude/skills/phantun-wireguard-tunnel
```

Claude Code will auto-discover it on next session and activate when the user describes a matching task (per the SKILL.md frontmatter `description`).

To verify:
```bash
ls ~/.claude/skills/phantun-wireguard-tunnel/SKILL.md
```

Then in Claude Code, ask something like *"help me set up a phantun + wireguard tunnel from my OpenWrt to a VPS"* and Claude should pick up the skill.

## Coverage

- **Phase A**: VPS (Debian 13 / Ubuntu) — wireguard-tools, phantun-server systemd unit, the critical DNAT rule
- **Phase B**: OpenWrt 24.10+ (APK / fw4 / nftables) — phantun-client procd init, UCI firewall zone for the TUN device
- **Phase B-R**: RouterOS 7.4+ with container — hand-built Docker v1.2 image (alpine + iptables, no Docker daemon required), veth/bridge/envs setup, the mandatory TUN_PEER return route, multi-VPS pattern
- **Phase C**: Bridge layer choices
  - **C-A** daed (no native WG) → sing-box SOCKS5 ⇄ WG bridge with DoH-via-tunnel
  - **C-B** OpenClash / Mihomo native WG outbound (including WG-specific `remote-dns-resolve`)
  - **C-C** sing-box transparent proxy alone (referenced)
- **Phase D**: end-to-end verification commands
- **Diagnostic flow**: 9 common symptoms mapped to root cause (OpenWrt + RouterOS)
- **Gotchas G1–G8**: covering OpenWrt kernel/firewall, RouterOS multi-netns routing/firewall, and Mihomo WG DNS

## Compatibility notes

Tested against:
- Phantun **v0.8.1**
- WireGuard kernel module (Debian 13, OpenWrt 25.x, RouterOS 7.x kernel)
- OpenWrt **25.12.x** (APK + fw4)
- MikroTik **RouterOS 7.4+** (container-capable)
- Debian **13 (trixie)**
- sing-box **1.12.x** (new endpoint format for WireGuard)
- daed **1.24.x** + dae core (no native WG outbound at this version)
- OpenClash / Mihomo Meta (native WG)
- Alpine **3.19** minirootfs as container base (~10 MB with iptables)

## License

MIT — see [LICENSE](./LICENSE).

## Origin

Distilled from real deployment sessions in April 2026. The initial OpenWrt version was from a daed/sing-box build; the RouterOS container variant came from a second session working through RouterOS's multi-netns quirks on live infrastructure (VPS1 + VPS2 + MikroTik + Mihomo). Each gotcha in the SKILL.md corresponds to an actual silent-failure incident, not theoretical advice.
