# ZeroTier NAT-flap fix (home LAN + relay fallback)

Runbook from a real incident (2026-09-28): SFTP over ZeroTier collapsed to
~60–100 kbps with 300–400 ms ping and 10%+ loss, while LAN and ISP were fine.
Two Ubuntu boxes (`strix-5090m-02` .207, `strix-5090m-01` .217) had the same
misconfiguration; an Android phone (`bb6de65b4d`, app 1.16.0) was the affected peer.

## Symptoms

- `ping <zt-ip>` over ZeroTier: 300–400 ms avg, 10%+ loss (same-WiFi peers should be <20 ms, 0% loss)
- `ss -ti`: `cwnd:1`, `ssthresh:2`, ~30% retransmits (`bytes_retrans` vs `bytes_sent`),
  `delivery_rate` in bps–kbps; `Send-Q` stuck at 70–120 kB on port 22
- `zerotier-cli listpeers`: affected peer shows `-` path and latency `-1`
  (100% root-relayed via PLANETs instead of direct P2P/LAN)
- `zerotier-cli info -j`: 4–5 different `surfaceAddresses` ports for one public IP
  (NAT mapping flapping), ~20+ `listeningOn` sockets including docker bridges

## Diagnose (on the slow server)

```bash
sudo zerotier-cli listpeers                      # look for '-' path / -1 latency
ping -c 8 -i 0.3 <peer-zt-ip>                    # expect <20ms on LAN, 0% loss
ss -ti dst <peer-zt-ip>                          # cwnd, retrans, delivery_rate
ping -c 8 -i 0.3 10.0.2.1                        # gateway (LAN health)
ping -c 8 -i 0.3 8.8.8.8                         # ISP uplink health
cat /var/lib/zerotier-one/local.conf            # missing = defaults (bad, see below)
sudo zerotier-cli info -j | python3 -m json.tool # listeningOn / surfaceAddresses
upnpc -l                                         # needs miniupnpc; "No IGD" = no UPnP
curl -s ifconfig.me                              # compare with router WAN: CGNAT check
```

Healthy baseline for contrast: gateway ~5 ms, 8.8.8.8 ~34 ms, PLANETs 150–350 ms
from Thailand. If LAN+ISP are fine but ZeroTier is bad, it's ZeroTier path
selection / NAT — not your WiFi.

## Root cause

1. No `/var/lib/zerotier-one/local.conf` → ZeroTier binds **every** interface,
   including docker bridges (`172.17–21.x`, `br-*`), creating ~20–24 UDP sockets
   across 3 ports. Stale mappings accumulate in `surfaceAddresses`.
2. Router has no UPnP IGD and no static port forward, so inbound depends purely
   on NAT hole-punch timing; mappings flap across ports and peers fall back to
   slow PLANET relay (`tcpFallbackActive` stays false — it's UDP relay, still slow).
3. Phone on same WiFi still relayed: no `10.0.2.x/9993` LAN path formed until the
   server side was cleaned up + service restarted (fresh discovery).

## Fix

`/var/lib/zerotier-one/local.conf` (see `zerotier-local.conf` in this repo):

```json
{
  "settings": {
    "primaryPort": 9993,
    "portMappingEnabled": true,
    "allowTcpFallbackRelay": true,
    "interfacePrefixBlacklist": ["br-", "docker", "veth", "virbr"]
  }
}
```

Apply:

```bash
sudo cp zerotier-local.conf /var/lib/zerotier-one/local.conf
sudo chown zerotier-one:zerotier-one /var/lib/zerotier-one/local.conf
sudo chmod 644 /var/lib/zerotier-one/local.conf
sudo systemctl restart zerotier-one
sleep 10
sudo zerotier-cli listpeers   # affected peer should gain 10.0.2.x/999x path, ~15 ms
ping -c 8 -i 0.3 <peer-zt-ip> # expect ~15 ms avg, 0% loss
```

Result here: sockets 24→12 (20→9 on box 1), phone `-1/no path` → direct
`10.0.2.209/9994 @ 16 ms`, ping 346 ms/12.5% loss → 16 ms/0%, SFTP ~62 Mbps.

## Router (durable fix for away-from-home / 5G)

Add a static forward on the home router (`http://10.0.2.1` here):
**UDP external 9993 → 10.0.2.207:9993** (repeat per box as needed).
Without UPnP this is the only thing that makes public-IP hole-punch
deterministic. Stale `surfaceAddresses` entries age out by themselves.

## Same-WiFi shortcut

If both ends are home, skip ZeroTier: SFTP to the LAN IP (`10.0.2.207`)
instead of the ZeroTier IP (`172.23.199.1`). Phone WiFi power-save causes one
initial ping spike (~200–800 ms) then stable 5–10 ms — that's normal.

## Tailscale cross-check

Tailscale was unaffected on the same NAT: endpoint-independent mapping,
DERP Singapore 64 ms, direct LAN peer at 6 ms. If a Tailscale device shows
`offline, last seen Nd ago`, its app just isn't running — open it to reconnect.
