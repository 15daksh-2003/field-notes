---
layout: post
title: "The login screen that drew to nowhere"
date: 2026-09-18
---

I switched my login manager to Plasma's own login screen, the thing that shows
the password box when you boot. Rebooted, and the login screen never came up.
Instead I landed at a pseudo-terminal, `pts/0`. The machine was up and running
fine. There was just no login screen on it.

I'd tried this same switch once before, given up, and gone back to SDDM without
working out why. This time I wanted the real reason.

## Reading the failed boot's logs

You can't see anything on a login screen that never drew, but the logs from that
boot are still on disk. `journalctl` keeps them per boot:

```
journalctl --list-boots
journalctl -b -1
```

`-b 0` is the current boot, `-b -1` the previous one, `-b -2` the one before
that. So I could boot back into a working setup and read what the broken boot
did.

Filtered to the login manager's compositor, the same two lines came up:

```
kwin_wayland: Failed to open /dev/dri/card2 device (Device or resource busy)
There are no outputs - creating placeholder screen
```

`kwin_wayland` is the compositor, the process that draws the Plasma login screen.
`/dev/dri/card2` is a GPU. It tried to open that GPU, was told it was busy, and
after a few seconds gave up and made a "placeholder screen": one that lives in
memory but isn't connected to any display. That's why nothing showed up.

## Which GPU, and why that one

Mine is a laptop with an AMD integrated GPU and an NVIDIA one. Only one of them
is wired to the screen. Two commands tell you which:

```
ls -l /dev/dri/by-path/
cat /sys/class/drm/card*-*/status
```

The first maps each `cardN` to a PCI address, so you know which chip it is. The
second shows which outputs are physically connected. On mine, `card2` is the AMD
GPU, and it's the one with the laptop panel (`eDP`) plugged in. The NVIDIA card
has nothing connected to it; it's there for offloading render work, not for
driving the display. So `card2` is the only GPU that can show anything. If the
compositor can't open it, there's nowhere to draw.

## Why did SDDM work?

This bugged me. SDDM, the manager I was replacing, used the same GPU and never
hit this.

The difference is X11 versus Wayland. SDDM's login screen runs on Xorg, and Xorg
takes the GPU by force (the kernel calls this becoming the "DRM master"). Plasma's
login screen runs on Wayland through `kwin_wayland`, which asks logind for the
GPU and backs off if something else already holds it. Same busy GPU: Xorg takes
it anyway, kwin doesn't.

That told me something was holding the GPU before the login screen tried to open
it.

## Looking it up first

Before digging further I searched the error. It turned out to be a known genre,
and three threads pointed me the right way:

- [Plasma 6.7.x Wayland fails to start on RX 580, AMDGPU DRM device busy](https://discuss.kde.org/t/plasma-6-7-x-wayland-is-failing-to-start-on-rx-580-amdgpu-drm-device-busy/48609) (KDE Discuss). Same error, an AMD card, Plasma 6.7, X11 working and Wayland not. Almost my exact setup, so I stopped blaming NVIDIA.
- [Systemd user service causes race condition](https://github.com/LizardByte/Sunshine/issues/3653) (Sunshine, GitHub). A background service opened the GPU a moment before the compositor and won the race. That gave me the shape of it: something grabs the GPU first.
- [KDE login randomly fails](https://discussion.fedoraproject.org/t/kde-login-randomly-fails/144377) (Fedora Discussion). Same error signature again, which told me this was a general "someone else has the GPU" problem, not anything specific to my machine.

So I needed to find what opens `card2` at boot before the login screen does.

## Finding the culprit

Two ways to look. `fuser` lists the processes using a file, and a GPU is a file
under `/dev/dri`:

```
fuser -v /dev/dri/card2
```

That tells you who has it right now. For the boot itself, I went back to the
journal and read the few seconds just before `kwin_wayland` gave up. One line sat
there, one second earlier:

```
Started KMS System Console on tty3.
kmscon ... Display [eDP] with backend [drm2d]
```

kmscon. It had opened `eDP` (the laptop panel, on `card2`) with its DRM backend,
one second before the login screen wanted the same GPU.

I'd installed kmscon a while back to get my external monitor working on the text
consoles, and had forgotten it was there.

Here's why it interferes: kmscon draws its console by opening the GPU directly and
holding it, the same `card2`, the same exclusive claim. It does that on `tty3` at
boot. Under SDDM it never mattered, because Xorg takes the GPU by force. Under a
Wayland login screen, both want `card2` at the same instant, and kmscon gets there
first.

There was one more piece. I *had* configured kmscon not to do this. My config said
`video=fbdev`, which tells it to use the plain framebuffer instead of grabbing the
GPU. But kmscon had since updated to a version that dropped that option, and the
log shows it ignoring the setting:

```
kmscon: ERROR: conf: unknown config option 'video'
```

So the config that used to keep it out of the way had quietly stopped working
after an update. It had been fine for months, and only broke once I changed the
login manager on the other side.

## The fix

The current kmscon can't be told to leave the GPU alone, and I don't need it
enough to fight it, so I removed it:

```
systemctl disable kmsconvt@tty3.service
pacman -Rns kmscon
```

`systemctl disable` stops it starting on that terminal. `pacman -Rns` removes the
package: `-R` removes it, `s` also removes dependencies nothing else uses, and `n`
deletes its config files instead of leaving `.pacsave` copies behind. With kmscon
gone, `card2` is free when the login screen starts.

Then the switch I wanted in the first place:

```
systemctl disable sddm.service
systemctl enable --force plasmalogin.service
```

`enable` also sets up the generic `display-manager.service` name that the system
boots at startup. `--force` lets it replace the one SDDM had claimed.

One last thing. The login screen used to come up on whatever virtual terminal was
free, which sometimes left me on a blank one. I wanted it on `tty1`. It wandered
because `getty@tty1`, the plain text login prompt, takes `tty1` at boot before the
login manager can:

```
systemctl mask getty@tty1.service
```

`mask` is a stronger `disable`. `disable` stops a service starting on its own but
still lets other units pull it in. `mask` points it at `/dev/null` so it can't
start at all. With `tty1` free, the login manager takes it.

Rebooted. Login screen on `tty1`, on both displays, first try.

## What I'd remember

- "Device or resource busy" on `/dev/dri/*` means something else already has the GPU. `fuser -v /dev/dri/cardN` tells you what.
- If X11 works and Wayland doesn't, that's a clue. They take the GPU differently, and that difference is often the whole bug.
- Debug the boot that failed, not the one you recovered into. `journalctl -b -1` is most of it.
