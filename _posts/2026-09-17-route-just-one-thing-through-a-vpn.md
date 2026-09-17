---
layout: post
title: "Route just one thing through a VPN"
date: 2026-09-17
---

I hit a small annoyance this week. One service I need — call it an internal
git server — is only reachable through a VPN. Fine, flip the VPN on. Except the
moment I do, *everything* slows down: video calls stutter, downloads crawl, my
editor's AI assistant starts lagging. All my traffic is suddenly taking a
detour through some far-away server just so one host works.

I didn't want the whole internet in the tunnel. I wanted exactly one
destination in it, and everything else to stay on my normal, fast connection.

Turns out WireGuard does this out of the box. No extra tools, no containers, no
proxy juggling. It comes down to one line.

## The realization

First I checked what I was actually connecting to:

```bash
getent hosts git.example.com
# 203.0.113.42  git.example.com
```

One IP. Stable. That reframes the whole problem — I'm not trying to route "a
network," I'm trying to route *one address*. Much smaller ask.

## How WireGuard decides what to tunnel

A WireGuard peer has a field called `AllowedIPs`. Most configs you'll see set
it to:

```
AllowedIPs = 0.0.0.0/0
```

which reads as "send *all* my traffic through here." That's the full-tunnel
default, and it's exactly why everything else slows down.

But `AllowedIPs` is really just a list of destinations that should go through
the tunnel. So you can make it as narrow as you like:

```
AllowedIPs = 203.0.113.42/32
```

The `/32` means "this one address, nothing else." Now WireGuard adds a route
for that single IP and leaves your default route alone. One host in the tunnel;
the rest of the internet untouched.

## The config

```ini
[Interface]
PrivateKey = <your key>
Address = 10.2.0.2/32

[Peer]
PublicKey = <server key>
Endpoint = <server ip>:51820
AllowedIPs = 203.0.113.42/32
PersistentKeepalive = 25
```

Bring it up:

```bash
sudo wg-quick up ./tunnel.conf
```

## Check it actually worked

Two lookups tell you everything:

```bash
ip route get 203.0.113.42   # -> dev tunnel   (the one host: tunneled)
ip route get 1.1.1.1        # -> dev wlan0    (everything else: direct)
```

First goes through the tunnel, second goes out your normal link. Done.

## A few notes

- I dropped the `DNS =` line. If the name already resolves fine without the
  VPN, you don't need the tunnel's DNS — only the *connection* needs routing.
  Fewer moving parts.
- Worried the address might change on you? Widen it a touch —
  `203.0.113.0/24` covers the whole block and is still nowhere near your
  default route.
- No kill-switch needed here. If the tunnel drops, that one host just stops
  working instead of leaking. Everything else was never in the tunnel to begin
  with.

That's it. One line of config, and "turn on the VPN" stops meaning "make
everything else worse."
