# phantun-wireguard-tunnel (Claude Code Skill)

A [Claude Code](https://claude.com/claude-code) skill that walks Claude through deploying a **Phantun + WireGuard** tunnel between an OpenWrt router (acting as proxy gateway with daed / OpenClash / sing-box) and a Debian/Ubuntu VPS.

## What this is for

ISP throttles or blocks UDP? Self-hosting a WireGuard egress that needs to look like TCP? This skill encodes a battle-tested deployment flow plus the **non-obvious gotchas** that consume hours when you follow upstream READMEs naively:

- VPS-side `iptables -t nat ... DNAT` is **mandatory** (phantun-server reads from TUN, not eth0 — silent failure otherwise)
- OpenWrt 24.10+ uses APK + fw4(nftables); the forward chain DROP policy needs a UCI zone, not a runtime `nft insert`
- Phantun OpenWrt binary must be **musl** (gnu fails with `not found`)
- `dial_mode` semantics: only `ip` mode passes IP to upstream proxy — `domain+` still passes hostname, contrary to common misreading
- DNS poisoning paths and where to insert DoH detours

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
- **Phase C**: Bridge layer choices
  - **C-A** daed (no native WG) → sing-box SOCKS5 ⇄ WG bridge with DoH-via-tunnel
  - **C-B** OpenClash / Mihomo native WG outbound
  - **C-C** sing-box transparent proxy alone (referenced)
- **Phase D**: end-to-end verification commands
- **Diagnostic flow**: 5 common symptoms mapped to root cause

## Compatibility notes

Tested against:
- Phantun **v0.8.1**
- WireGuard kernel module (Debian 13, OpenWrt 25.x)
- OpenWrt **25.12.x** (APK + fw4)
- Debian **13 (trixie)**
- sing-box **1.12.x** (new endpoint format for WireGuard)
- daed **1.24.x** + dae core (no native WG outbound at this version)
- OpenClash with Mihomo Meta core (native WG)

## License

MIT — see [LICENSE](./LICENSE).

## Origin

Distilled from a real deployment session in April 2026. The conversation that produced this skill walked through every failure mode end-to-end on live infrastructure (VPS + OpenWrt). Each gotcha in the SKILL.md corresponds to an actual silent-failure incident, not theoretical advice.
