---
layout: post
title: "Is it me, my ISP, or them?"
date: 2026-09-17
---

One site wouldn't load. Everything else was completely fine, which is the part
that throws you — if the whole connection were down I'd know where to start.
Instead it's a single host timing out while the rest of the internet works.

So before guessing, I spent a few minutes working out where it actually breaks.
"It doesn't work" covers at least three different problems, and you fix them in
completely different places.

## Does the name even resolve?

```bash
getent hosts git.example.com
# 203.0.113.42  git.example.com
```

An IP came back, so DNS is fine. If this had turned up empty I'd be off chasing
a name-resolution problem, which is a different thing entirely. It didn't, so I
kept going.

## What does the failure look like?

This is the bit worth slowing down on, because *how* it fails narrows things
down fast. So watch one connection go:

```bash
curl -v --connect-timeout 10 https://git.example.com
```

Roughly three ways it can go:

- It just hangs. `Trying 203.0.113.42:443…` and then silence until it gives up.
  That's a firewall dropping your packets on the floor — nothing comes back at
  all.
- Connection refused. Something's home and it's slamming the door (you get a
  `RST` back).
- It connects, the TLS handshake works, and then you get a `403`. You actually
  reached them; they just won't serve you. That's a geo rule or a WAF.

Mine hung and timed out with nothing coming back. So: a firewall quietly
dropping my traffic, not a refusal and not an app rule.

## Is this about me, or the site?

Cheap way to check — change where I'm coming from. Same request over a VPN, or
off a phone hotspot on a different network. It connected straight away.

That settles it. Same laptop, same DNS answer, same site. The only thing that
changed was the address I showed up from, and that was enough to fix it. So
whatever's blocking me is looking at my source IP, not my account and not the
site being down.

## Fine — what does my IP look like to them?

If I'm getting blocked by address, I want to see my address the way they see
it. Three lookups:

```bash
curl -s https://ipinfo.io/json      # public IP, ISP, ASN
getent hosts 203.0.113.42           # reverse DNS of my own IP
curl -s "https://stat.ripe.net/data/network-info/data.json?resource=203.0.113.42"
```

The reverse DNS had the word `dynamic` sitting right in the hostname, and the
RIPEstat lookup put me in a shared residential block owned by a big consumer
ISP.

Which explains it. A dynamic IP isn't really *mine* — the ISP loans one out of
a pool and I get whatever's free that day. If the person who had it before me,
or someone a few addresses down the street, did something bad enough to get the
range blocked, I get caught in it without having done anything. And tomorrow
I'll be on a different address anyway.

## What the five minutes got me

Instead of "the site's broken" I know it's a firewall dropping traffic from my
ISP's range, that I'm collateral rather than the actual target, and that the
fix is to come from somewhere else. The reason for going in order — resolve,
watch the failure, swap the source, then look myself up — is that each step
rules out a layer. Skip them and you can lose an hour debugging DNS when the
real problem is a firewall three hops away.
