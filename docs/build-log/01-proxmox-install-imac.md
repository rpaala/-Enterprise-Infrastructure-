# Build Log 01: Proxmox VE on a 2011 iMac (foundation, done before Phase 1)

**Status:** Complete (host is built and running; see the [phase plan](../phase-plan.md))
**Hardware:** iMac 12,1 (2011, 21.5" or 27"), previously running Fedora 43
**Goal:** Replace Fedora with Proxmox VE, running fully headless, managed remotely from Windows laptop

---

## Context / Decisions

- Fedora removed entirely — Proxmox VE installed as bare-metal hypervisor
- Known hardware issue: AMD Radeon GPU pixelation on sleep/wake (solder degradation, common in this model/year)
- Mitigation: disable sleep, never rely on the internal display after install
- All post-install management via Proxmox web UI + SSH from Windows laptop
- Ethernet required (not wifi) — both for install reliability and future Wake-on-LAN

---

## Networking Note — Powerline Adapters

iMac is on a different level of the house from the Ubiquiti router, with no existing wall ethernet. Solution: **powerline adapters** — carry a wired network signal over the home's existing electrical wiring.

- Kit chosen: **TRENDnet TPL-422E2K** (Powerline 1300 AV2, single Gigabit port per adapter, 128-bit AES encryption, US company) — under $90, avoids TP-Link (see note below)
- TP-Link avoided due to March 2026 FCC restrictions on foreign-produced networking gear tied to national security concerns
- 128-bit AES is standard across all mainstream powerline adapters (TRENDnet, Netgear, TP-Link) and is not a meaningful security gap for this use case — VLAN segmentation (Phase 2) does far more for actual security posture than chasing 256-bit AES on this link
- Stick to HomePlug AV2 standard (not Zyxel's G.hn) if the powerline network is ever expanded later — HomePlug AV2 devices are cross-compatible across brands, G.hn generally is not

**Topology:**
```
Gateway LAN port → powerline adapter (router end) → [house electrical wiring] →
powerline adapter (iMac end) → small unmanaged gigabit switch → iMac
                                                                → NAS (or other future device)
```

- Router end only needs a single port — the powerline adapter there just injects the signal; any other wired devices near the router plug directly into the gateway's remaining LAN ports
- iMac end needs a small **unmanaged gigabit switch** (~$12-15, e.g. Netgear GS305 or TRENDnet TEG-S350) since the powerline adapter itself is single-port but multiple devices (iMac, NAS, future Pi) may live at that end
- Plug both powerline adapters directly into wall outlets — **not** into a power strip or surge protector, which kills the signal
- Powerline only works reliably within a single electrical circuit/breaker panel — if adapters won't sync at all, this is the most likely cause

- [ ] Powerline kit purchased (TRENDnet TPL-422E2K)
- [ ] Small unmanaged switch purchased for iMac end
- [ ] Adapter A connected at router end, into Gateway LAN port, synced
- [ ] Adapter B connected at iMac end, synced
- [ ] Switch connected to Adapter B, iMac (and NAS if applicable) connected to switch
- [ ] Link light active on all connections before proceeding to Step 1

---

## Pre-Install Checklist

- [ ] Confirm iMac model exactly (About This Mac / dmidecode) — 2011 = iMac12,1 or iMac12,2
- [ ] Back up anything needed off the Fedora install (this wipes the drive)
- [ ] USB drive, 8GB+, for the Proxmox installer
- [ ] Powerline adapter kit (see Networking Note below) — iMac has no wired path to router, no existing wall ethernet
- [ ] Small unmanaged gigabit switch for iMac end (see Networking Note)
- [ ] Windows laptop on the same network/VLAN for initial access
- [ ] Keyboard temporarily connected to iMac (needed for installer + first boot troubleshooting only)
- [ ] Verify iMac RAM (8GB minimum workable, 16GB+ preferred for multiple VMs)

---

## Step 1 — Download Proxmox VE ISO

1. On Windows laptop, go to: https://www.proxmox.com/en/downloads
2. Download the latest **Proxmox VE ISO Installer**
3. Note the version number for this log: `___________`

---

## Step 2 — Create Bootable USB Installer

Using **Rufus** (Windows):

1. Insert USB drive (will be erased)
2. Open Rufus → select the Proxmox ISO
3. Partition scheme: **GPT** (2011 iMac uses EFI boot)
4. File system: default (FAT32)
5. Click Start, wait for completion

> Note: 2011 iMacs use Apple's EFI implementation, not standard PC EFI/legacy BIOS. Boot behavior can be inconsistent — see Troubleshooting section if the USB doesn't appear at boot.

---

## Step 3 — Boot the Installer on the iMac

1. Confirm powerline link is up (link lights active on both adapters — see Networking Note above)
2. Shut down the iMac fully
3. Insert the USB installer
4. Power on and immediately hold **Option/Alt (⌥)** key
5. Boot picker should show an "EFI Boot" option — select it
6. Proxmox installer boot menu should appear

**If the USB doesn't show up in the boot picker:**
- Try re-flashing with `dd` from a Linux/WSL machine instead of Rufus (more reliable EFI boot record for Mac firmware)
- Try holding **C** instead of Option at boot
- Confirm USB is formatted GPT, not MBR

---

## Step 4 — Run the Installer

> This is the one point in the whole build where you'll be looking at the iMac's own screen. If pixelation appears here, it's cosmetic to the install process — the installer should still be usable. Note in this log if it was legible or not.

1. Select **Install Proxmox VE**
2. Accept EULA
3. **Target disk:** select the internal drive — confirm it's wiping the right one
4. **Country/Timezone/Keyboard:** set accordingly
5. **Root password + email:** set (record password in your password manager, not in this doc)
6. **Network configuration:**
   - Hostname: `<PROXMOX_HOSTNAME>` (e.g. `proxmox-01.lab.example`)
   - IP address: set a **static IP** (do this now, not later — pick something outside your router's DHCP range, e.g. `<PROXMOX_IP>` from your management subnet)
   - Gateway: your Ubiquiti router's IP
   - DNS: router IP or `1.1.1.1`
7. Confirm and install
8. Reboot when prompted, **remove the USB drive** when it ejects/prompts

Record actual values used:

| Setting | Value |
|---|---|
| Hostname | |
| Static IP | |
| Gateway | |
| VLAN (if applicable) | |
| Proxmox version installed | |

---

## Step 5 — First Boot Verification

1. On reboot, the iMac screen should show a console with a URL like:
   `https://<PROXMOX_IP>:8006`
2. **This is the last time you need to look at the physical screen for normal operation.**
3. From the Windows laptop, open a browser to that URL
4. Accept the self-signed cert warning (expected — replace later if you want proper TLS)
5. Log in: username `<PROXMOX_USER>`, password set in Step 4

- [ ] Web UI reachable from laptop
- [ ] Node shows green/online status

---

## Step 6 — Disable Sleep (Critical — GPU Mitigation)

Sleep is what's believed to trigger the pixelation fault. Disable it at the OS level.

Via SSH (from Windows laptop — use PowerShell, or PuTTY):

```bash
ssh <PROXMOX_USER>@<PROXMOX_IP>
```

Check/set power management so the system never suspends:

```bash
# Proxmox is Debian-based; confirm no sleep target is active
systemctl status sleep.target
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

- [ ] Sleep/suspend/hibernate targets masked
- [ ] Confirmed no scheduled sleep in firmware settings (Mac EFI power settings, if accessible)

---

## Step 7 — Close the Lid / Stop Using the Screen

From this point forward:
- Do not wake or interact with the physical iMac screen for routine use
- All management happens via:
  - Web UI: `https://<static-ip>:8006`
  - SSH: `ssh <PROXMOX_USER>@<static-ip>`

- [ ] Confirmed can manage node fully without touching the iMac directly

---

## Step 8 — Power / Access Planning (decide and document)

- [ ] Option A: Leave running 24/7 (~$7–8/month estimated power cost)
- [ ] Option B: Wake-on-LAN configured, machine sleeps/shuts down on schedule
- [ ] Option C: Smart plug scheduling (fallback if WoL unreliable on this hardware)

WoL test (if pursuing B) — from Proxmox shell:

```bash
ethtool eth0 | grep Wake-on
```

Note result here: `___________` (look for `g` in supported/current Wake-on modes)

---

## Troubleshooting Log

| Date | Issue | Resolution |
|---|---|---|
| | | |

---

## Next Phase

→ **Phase 2:** Network segmentation on Ubiquiti (VLANs) before creating first VMs
→ **Phase 3:** UGreen NAS as Proxmox storage backend (NFS mount)
