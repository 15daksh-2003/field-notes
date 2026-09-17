---
layout: post
title: "Route just one thing through a VPN"
date: 2026-09-17
---

There's one git server I use that only answers when I'm on the VPN. Fine, I'll
connect the VPN. The catch is that the moment I do, everything else gets worse.
Calls start breaking up, downloads crawl, my editor's AI stuff hangs. All of my
traffic is suddenly detouring through some server far away, just so that one
host is reachable.

That's a bad trade. I don't need my whole connection in the tunnel. I need one
address in it, and I want everything else left alone.

WireGuard can do exactly that, without any extra tooling. It's one line in the
config.

## What am I actually connecting to?

```bash
getent hosts git.example.com
# 203.0.113.42  git.example.com
```

(That's just a DNS lookup — it prints the IP behind a name.)

One address, and it doesn't move around. That changes the whole shape of the
problem. I'm not routing a network, I'm routing a single IP, which is a much
smaller thing to ask for.

## The line that does it

Every WireGuard peer has an `AllowedIPs` field. You've almost certainly seen it
as:

```
AllowedIPs = 0.0.0.0/0
```

which just means "put everything through the tunnel." That's the setting that
slows the rest of your connection down.

But it's only a list of what belongs in the tunnel, so you can shrink it right
down:

```
AllowedIPs = 203.0.113.42/32
```

The `/32` is "this one address, nothing else." WireGuard adds a route for that
single IP and doesn't touch your default route. One host goes through the
tunnel. The rest of your traffic never notices.

## The config

You're not writing this by hand. If you're on something like Proton VPN, their
WireGuard generator (it's in the account dashboard) gives you a finished `.conf`
with the keys and `Endpoint` already filled in. It just ships with
`AllowedIPs = 0.0.0.0/0`, so you change that one line:

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

Then bring it up. `wg-quick` reads the file, creates the interface, and adds
routes for whatever's in `AllowedIPs`:

```bash
sudo wg-quick up ./tunnel.conf
```

## Did it take?

Ask the routing table where two different addresses go:

```bash
ip route get 203.0.113.42   # dev tunnel  — the one host
ip route get 1.1.1.1        # dev wlan0   — everything else
```

The first one's in the tunnel, the second isn't. That's all I wanted.

## A couple of things worth knowing

I deleted the `DNS =` line that came with the config. The name already resolves
fine without the VPN, so there's no reason to send my lookups through the
tunnel — only the connection to that one host needs it.

If you're nervous the address might change on you someday, give it some room:
`203.0.113.0/24` covers the surrounding block and is still nowhere near your
default route.

And you don't need a kill-switch here. A kill-switch is a rule that cuts off all
traffic when the tunnel drops, so nothing leaks out unprotected. With a single
host in the tunnel there's nothing to leak — if it drops, that one server just
goes quiet, and everything else was never in there to begin with.

So: one line, and turning the VPN on stops meaning "make everything else
worse."
