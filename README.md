# Fixing NVIDIA hibernate on Linux — the `-5` resume failure

**What this fixes:** your PC hibernates (saves everything to disk and powers fully
off) just fine — but when you turn it back on, instead of picking up where you left
off, it boots up fresh and your session is gone. If you check the logs you'll see the
NVIDIA driver failing with a `-5` error. This was solved on an RTX 50-series
("Blackwell") desktop with the open NVIDIA driver, but the cause is more general.

**Why it happens, in one sentence:** NVIDIA gets switched on too early in the boot
process, so when your PC tries to wake from hibernation, the little temporary system
that loads your saved session already has the graphics card running — and it can't
hand it over cleanly, so waking up fails. The fix is to load NVIDIA a moment *later*
instead. Your graphics still work perfectly; NVIDIA just starts after the tricky
moment has passed.

---

## How to use this repo — pick one

**Option A — let an AI do it for you (easiest).** Download **`agent.md`** and give it
to an AI assistant that can run terminal commands (such as Claude Code). That file is
a complete set of instructions written *for the AI*: it inspects your specific
machine, works out the right settings, applies the fix, tests it, and makes it
permanent — checking with you before it changes anything. You mostly just say "go" and
run the occasional command it hands you.

**Option B — follow this guide yourself.** Everything below is copy-paste-able into a
terminal. You don't need to understand the internals to do it — you just need to be
comfortable running commands. If you get curious about *why* a step works, the deeper
explanation lives in `agent.md`.

Both options make the same changes, and every change is reversible — see
[Reverting](#reverting) at the end if you ever want to undo it.

> A quick note on the jargon you'll meet below: **hibernate** = save everything to
> disk and power off; **initramfs** = a tiny startup bundle the kernel loads before
> anything else; **early KMS** = the graphics driver being put *into* that bundle so
> it loads first thing; **swap** = disk space the system can borrow to stash memory
> (hibernation needs a real chunk of it). You don't have to memorize these — they're
> just here so the steps make sense.

---

## Does this apply to you?

You're likely in the right place if **all** of these are true:

- NVIDIA GPU (this was tested on **Blackwell / RTX 50-series** with the **open**
  kernel modules, but the mechanism applies more broadly).
- **Suspend (S3) works, but hibernate (S4) does not** — on resume the machine
  cold-boots into a fresh session instead of restoring the one you left.
- Your `journalctl -b -1 -k` from the failed resume contains something like:

  ```
  PM: Image loading progress: 100%
  PM: Image loading done
  NVRM: GPU 0000:01:00.0: PreserveVideoMemoryAllocations module parameter is set.
        System Power Management attempted without driver procfs suspend interface.
  nvidia 0000:01:00.0: PM: pci_pm_freeze(): nv_pmops_freeze [nvidia] returns -5
  nvidia 0000:01:00.0: PM: dpm_run_callback(): pci_pm_freeze returns -5
  nvidia 0000:01:00.0: PM: failed to quiesce async: error -5
  PM: hibernation: resume failed (-5)
  ```

The tell-tale detail: the image **loads to 100% first**, *then* the NVIDIA freeze
fails. The image write and swap are fine — the failure is the GPU device on the
resume side.

### Is NVIDIA being loaded too early?

This command looks inside your startup bundles and reports whether the NVIDIA driver
is baked in. If it prints any `nvidia*.ko` lines, that's the thing we're going to fix:

```bash
sudo find /boot -name 'initramfs*' -exec sh -c \
  'echo "== $1 =="; lsinitcpio "$1" 2>/dev/null | grep -i "nvidia.*\.ko" || echo "  none found here"' _ {} \;
```

On Arch-based systems (including CachyOS) the culprit is usually a small config file
under `/etc/mkinitcpio.conf.d/` that contains a line like
`MODULES+=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)` — that's what forces NVIDIA
into the startup bundle. (On CachyOS that file is written automatically by a tool
called **chwd**, which matters later for making the fix stick.)

---

## First: can your PC hibernate at all?

Before touching NVIDIA, make sure hibernation can even start. Try:

```bash
systemctl hibernate
```

If your PC powers off and only *waking up* fails, your swap is fine — **skip to
[The fix](#the-fix-load-nvidia-a-little-later).** But if you instead get:

```
Call to Hibernate failed: Not enough suitable swap space for hibernation
available on compatible block devices and file systems
```

…then your PC has no usable **swap** to write the saved session into. (A common cause:
the only swap is **zram**, which lives in RAM — useless for hibernation, since the
whole point is to power RAM off.) You need to set aside a real chunk of disk for it
first. If this isn't you, skip ahead to the fix.

Example for a **btrfs** root (adjust size to ≥ your RAM; skip the subvolume step
on other filesystems):

```bash
# 1. Dedicated subvolume keeps the swapfile out of snapshots
sudo btrfs subvolume create /swap
sudo btrfs filesystem mkswapfile --size 64g /swap/swapfile   # handles NOCOW + mkswap
sudo swapon /swap/swapfile

# 2. Persist it
echo '/swap/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab

# 3. Get the values the kernel needs to find the image on resume
findmnt -no UUID /                                            # -> <FS-UUID>
sudo btrfs inspect-internal map-swapfile -r /swap/swapfile   # -> <RESUME-OFFSET>
#    (use map-swapfile, NOT filefrag, on a compressed btrfs)
```

Then add to your kernel command line (replace the placeholders with the values
you just read):

```
resume=UUID=<FS-UUID> resume_offset=<RESUME-OFFSET>
```

Rebuild your initramfs/bootloader config and reboot, then verify:

```bash
cat /sys/power/resume /sys/power/resume_offset   # both non-zero; offset matches
swapon --show                                    # your disk swapfile is listed
```

> **Bootloader note:** where you put the cmdline and how you regenerate differ by
> distro. On CachyOS with Limine, the cmdline lives in `/etc/default/limine`
> (`KERNEL_CMDLINE[default]+=`) and you regenerate with **`limine-update`** (which
> rebuilds the initramfs *and* the bootloader config). GRUB users edit
> `/etc/default/grub` and run `grub-mkconfig`. Use whatever your distro uses.

---

## The fix: load NVIDIA a little later

The goal is simple — **stop NVIDIA from being baked into the startup bundle** so it
loads a moment later instead. The smart way to do this survives future updates: rather
than deleting the file that puts NVIDIA there (an update can just recreate it), we add
a *second* file that runs afterward and takes NVIDIA back out.

### 1. Add the override file

This creates one small config file. Copy-paste the whole block as-is:

```bash
printf '%s\n' \
  '# Keep NVIDIA out of early startup so hibernate can wake back up.' \
  '# Something earlier (e.g. chwd) may add it; this file runs last and removes it.' \
  'MODULES=(${MODULES[@]/nvidia*/})' \
  | sudo tee /etc/mkinitcpio.conf.d/99-no-nvidia-early-kms.conf
```

Why this works: these config files are read in filename order, and a name starting
with `99-` sorts last — so it runs *after* whatever added NVIDIA and strips it out
again on every rebuild. (It's the same trick the system already uses elsewhere.)

### 2. Rebuild the initramfs

Use your distro's tool:

```bash
# CachyOS / Limine:
sudo limine-update           # or: sudo limine-mkinitcpio
# Plain Arch:
sudo mkinitcpio -P
```

### 3. Check that NVIDIA is really gone

Run the same check from earlier — it should now come back clean:

```bash
sudo find /boot -name 'initramfs*' -exec sh -c \
  'echo "== $1 =="; lsinitcpio "$1" 2>/dev/null | grep -i "nvidia.*\.ko" || echo "  clean: no nvidia"' _ {} \;
```

Your main startup bundles should say `clean: no nvidia`. **Don't skip this step** — it's
how you know the fix actually took. One thing that's normal: if some entries live in a
folder like `limine_history` (or are labelled "fallback" / "rollback"), those are
frozen backups of older setups and may still list NVIDIA. That's fine — they're only
used if you deliberately pick an old entry from the boot menu. What matters is that the
ones you normally boot are clean.

### 4. Reboot

Expect a **brief blank or low-resolution console** before your login screen —
that's correct now. NVIDIA loads later, in userspace, instead of during early
boot. Your desktop and GPU work exactly as before once you're logged in.

---

## Test it

1. **Warm test first.** Log in, open a couple of identifiable things (a terminal
   with some unsaved scratch text is perfect), then:

   ```bash
   systemctl hibernate
   ```

   Power it back on. If your session comes back *exactly* as you left it — same
   windows, same scratch text — that's a real restore, not your DE reopening apps
   fresh.

2. **The real proof — full power-off.** Hibernate, then **physically cut power**
   (PSU switch or pull the cord) for a few minutes. DRAM loses its contents within
   seconds, so if the session still restores on cold boot, you've proven it came
   off the disk image with genuine zero draw in between. That's true S4.

Confirm the resume log is clean:

```bash
sudo journalctl -b -1 -k | grep -iE 'pm:|nvidia|freeze|hibernat|preserve' | tail -40
```

The `pci_pm_freeze ... returns -5` line should be **gone**.

---

## Make it durable (so an update can't silently break it)

The override drop-in above is already the durable approach — but understand *why*
it matters:

On CachyOS the file that injects NVIDIA into early KMS (`10-chwd.conf`) is
**auto-generated by chwd**. If you "fix" this by deleting or renaming that file,
a future NVIDIA/chwd package update can regenerate it, re-inject NVIDIA into the
initramfs, and silently bring the `-5` failure back the next time the initramfs
is rebuilt — with no obvious cause.

The `99-no-nvidia-early-kms.conf` drop-in avoids that trap: it doesn't fight the
generated file, it just runs afterward and removes the modules again. chwd can
regenerate `10-chwd.conf` all it likes; the `99-` override strips NVIDIA out
every rebuild. After any big update, it's still worth re-running the **step 3**
verification once, just to be sure.

---

## Reverting

Remove the override and rebuild:

```bash
sudo rm /etc/mkinitcpio.conf.d/99-no-nvidia-early-kms.conf
sudo limine-update      # or your distro's initramfs rebuild command
```

NVIDIA returns to early KMS on the next boot (and hibernate resume breaks again).

---

## Notes & caveats

- **Your GPU is not disabled.** NVIDIA still loads — just later, in userspace.
  Confirm with `lsmod | grep nvidia` after logging in.
- **Cosmetic quirk:** with NVIDIA out of early KMS the login greeter may draw a
  beat before the password field grabs focus, so you might have to click the
  screen once before typing. Harmless.
- **This is the fix that worked on one specific Blackwell/open-driver setup.**
  Other reported approaches (setting `PreserveVideoMemoryAllocations=0`, or
  enabling the `nvidia-suspend`/`nvidia-resume`/`nvidia-hibernate` systemd
  services) address the same underlying conflict from different angles. If early
  KMS isn't the source of *your* early GPU load, those are the next things to try.
