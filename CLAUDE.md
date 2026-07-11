# CLAUDE.md — NVIDIA Blackwell hibernate (`-5` on resume) fix playbook

You are an AI assistant helping a user get **hibernate (S4, suspend-to-disk)**
working on **their own Linux machine**. Suspend (S3) usually already works;
hibernate fails on resume. This file gives you the mechanism, the method, and the
exact fix — but **every machine-specific value must be read off the user's system,
never assumed from this document.** The fix described here was derived on one real
machine; the *values* (kernel versions, UUIDs, offsets, paths) were specific to it
and are deliberately not repeated here.

---

## The result you are driving toward

The user can run `systemctl hibernate`, the machine writes its state to disk and
powers fully off, and on the next boot the **exact session is restored** — same
windows, same unsaved text — with zero power draw in between. Then it's hardened so
a package update can't silently break it again.

---

## Operating rules (follow these — they are what makes this reliable)

1. **Read the fact before you theorize.** Run the inspection command, read the real
   output, *then* decide. Do not invent kernel/driver versions, UUIDs, offsets, or
   paths — get them from the machine.
2. **One variable per change.** Change one thing, test, observe, and revert it if it
   didn't help. Batching changes destroys the signal.
3. **Keep a log.** Record each step as: hypothesis → command → output → conclusion →
   next step. If the repo has a `debug-log.md`, append to it.
4. **Confirm before system changes.** Creating swap, editing the kernel command line,
   and rebuilding the initramfs are real, semi-persistent changes. Explain what you're
   about to do and why, then proceed. Anything hard to reverse: confirm first.
5. **You will need root.** If you can't run `sudo` non-interactively, hand the user
   the exact commands to run in their terminal and ask them to paste the output back.
6. **Verify against the *actually booted* image**, not a guessed path. A stale or
   wrong-path check is the #1 way this fix gets mistaken for done when it isn't.

---

## Why hibernate fails here (the mechanism — understand this first)

The failure signature, seen in `journalctl -b -1 -k` from the *failed resume*:

```
PM: Image loading progress: 100%
PM: Image loading done
NVRM: GPU <pci-id>: PreserveVideoMemoryAllocations module parameter is set.
      System Power Management attempted without driver procfs suspend interface.
nvidia <pci-id>: PM: pci_pm_freeze(): nv_pmops_freeze [nvidia] returns -5
nvidia <pci-id>: PM: dpm_run_callback(): pci_pm_freeze returns -5
nvidia <pci-id>: PM: failed to quiesce async: error -5
PM: hibernation: resume failed (-5)
```

Read it carefully: the hibernation **image loads to 100% first**, *then* the NVIDIA
freeze aborts with `-5` (`-EIO`). So the swap/image path is fine — the failure is
the **GPU device on the resume side**.

Root cause: the NVIDIA modules are loaded in **early KMS** — baked into the
initramfs and loaded in the first moments of boot. During resume, a small
"bootstrap" kernel loads your hibernation image and must quiesce its *own* devices
to hand off to the restored kernel. Because nvidia was loaded in early KMS, that
bootstrap kernel has a live GPU device — and its freeze callback fails (`-5`),
aborting the whole resume.

**The fix: keep nvidia out of early KMS.** Then the bootstrap kernel has no GPU
device to freeze, and resume completes. NVIDIA still loads a moment later in
userspace, so the desktop and GPU work normally (expect a brief blank/low-res
console before the login screen — that's correct now).

---

## Step 0 — Confirm this is actually the user's problem

Gather these facts from **their** machine and confirm the picture matches before
changing anything:

```bash
# GPU + driver
lspci -k | grep -iA3 'vga\|3d\|display'
# (Confirm NVIDIA; the tested case was Blackwell / RTX 50-series with the OPEN
#  kernel modules, but the mechanism applies more broadly.)

# The failure signature from the last failed resume (the key evidence)
sudo journalctl -b -1 -k | grep -iE 'pm:|nvidia|freeze|hibernat|preserve|resume' | tail -40

# Is nvidia in early KMS? Find the initramfs generator first:
#   mkinitcpio -> Arch/CachyOS/etc ;  dracut -> Fedora/openSUSE/etc
command -v mkinitcpio dracut
# For mkinitcpio, look for a MODULES+=(nvidia ...) injection:
grep -rn 'MODULES' /etc/mkinitcpio.conf /etc/mkinitcpio.conf.d/ 2>/dev/null
```

Proceed only if: NVIDIA GPU, hibernate fails on resume (not on the way down), the
`-5` freeze signature is present, and nvidia is loaded via early KMS. If the failure
is something else (e.g. it never powers off), diagnose that instead.

---

## Step 1 — Make sure hibernate can even run (swap + resume params)

Test first:

```bash
systemctl hibernate
```

If it powers off and only *resume* fails, swap is already fine — **skip to Step 2.**
If instead you get `Not enough suitable swap space for hibernation`, the machine has
no usable disk-backed swap (commonly only zram, which is RAM-backed and invalid for
hibernation). Provision real swap sized ≥ RAM.

Detect the root filesystem type — the method differs:

```bash
findmnt -no FSTYPE,SOURCE,UUID /
```

**btrfs example** (adjust the size to ≥ RAM):

```bash
sudo btrfs subvolume create /swap                              # keep it out of snapshots
sudo btrfs filesystem mkswapfile --size <RAM_GiB>g /swap/swapfile
sudo swapon /swap/swapfile
echo '/swap/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab
findmnt -no UUID /                                             # -> <FS-UUID>
sudo btrfs inspect-internal map-swapfile -r /swap/swapfile    # -> <RESUME-OFFSET> (btrfs: use THIS, not filefrag)
```

**ext4/xfs swapfile:** create with `fallocate`/`dd` + `mkswap`, then get the offset
with `filefrag -v <swapfile>` (first physical block). **Dedicated swap partition:**
no offset needed — just its UUID.

Add resume parameters to the kernel command line (values from above):

```
resume=UUID=<FS-UUID> resume_offset=<RESUME-OFFSET>
```

Put them where *this* system keeps the cmdline and regenerate — detect the
bootloader, don't assume:

- **Limine (CachyOS):** edit `/etc/default/limine` (`KERNEL_CMDLINE[default]+=`),
  then `sudo limine-update` (rebuilds initramfs **and** the bootloader config).
- **GRUB:** edit `/etc/default/grub` (`GRUB_CMDLINE_LINUX_DEFAULT`), then
  `sudo grub-mkconfig -o /boot/grub/grub.cfg`.
- **systemd-boot:** add to the entry / `kernelcmdline`, per the distro's tooling.

Reboot, then verify and re-test:

```bash
cat /sys/power/resume /sys/power/resume_offset   # both non-zero; offset matches
swapon --show                                    # disk swapfile present
systemctl hibernate                              # should now reach the nvidia stage and fail with -5
```

Reaching the `-5` at resume confirms swap is solved and you're now at the real
problem.

---

## Step 2 — The fix: keep nvidia out of early KMS (durably)

Do this in the **update-proof** way from the start: don't delete the file that adds
nvidia (a package can regenerate it) — add an override that removes nvidia again,
ordered to run last.

### mkinitcpio systems (Arch / CachyOS / etc.)

Create `/etc/mkinitcpio.conf.d/99-no-nvidia-early-kms.conf`:

```bash
printf '%s\n' \
  '# Keep NVIDIA out of early KMS so hibernate can resume.' \
  '# Anything earlier (e.g. a generated 10-chwd.conf) may re-add them via MODULES+=();' \
  '# this file sorts last and strips them back out on every rebuild.' \
  'MODULES=(${MODULES[@]/nvidia*/})' \
  | sudo tee /etc/mkinitcpio.conf.d/99-no-nvidia-early-kms.conf
```

Drop-ins in `mkinitcpio.conf.d/` are sourced in filename order, so a `99-` file runs
*after* whatever added nvidia to `MODULES` and removes it. (This is the same pattern
generators like CachyOS's `chwd` already use to drop the generic `kms` hook.) Verify
the strip logic on any bash before trusting it:

```bash
bash -c 'MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm foo); MODULES=(${MODULES[@]/nvidia*/}); echo "[${MODULES[*]}]"'
# expected: [foo]
```

If a generator wrote a file that *disables* the generic `kms` hook plus one that
*injects* nvidia, leave both as the package manages them — the `99-` override
neutralizes the injection without fighting the package.

### dracut systems (Fedora / openSUSE / etc.)

The equivalent is to stop forcing nvidia to load early: remove nvidia from any
`force_drivers`/`add_drivers` line in `/etc/dracut.conf.d/*.conf` (or the nvidia
dracut drop-in), then regenerate. Verify the actual setting on the machine before
editing — don't assume the filename.

---

## Step 3 — Rebuild the initramfs

Use the tool the distro actually uses (you detected it in Step 0):

```bash
sudo limine-update        # CachyOS/Limine: rebuilds all images + limine.conf
# or
sudo mkinitcpio -P        # plain Arch
# or
sudo dracut -f --regenerate-all
```

If `mkinitcpio -P` errors with "No presets found," the distro drives initramfs
through its own wrapper (e.g. CachyOS uses `limine-mkinitcpio`) — use that instead.

---

## Step 4 — Verify nvidia is gone from the *booted* image

Do not trust a guessed path. Find every initramfs and check it:

```bash
sudo find /boot -name 'initramfs*' -exec sh -c \
  'echo "== $1 =="; lsinitcpio "$1" 2>/dev/null | grep -i "nvidia.*\.ko" || echo "  clean: no nvidia .ko"' _ {} \;
```

The **default-boot images** must report `clean: no nvidia .ko`. Some setups keep a
`*_history/`, `*-fallback`, or rollback directory of frozen older images that still
contain nvidia — that's expected and harmless; those are only used if the user
manually selects an old rollback entry (which would reintroduce the bug for that boot
only). What matters is that the images the default menu entries boot are clean.

If a default image still lists `nvidia*.ko`, **do not reboot into it** — the override
didn't take. Re-check filename ordering, that the rebuild ran against the right
config, and that the strip pattern matched, then rebuild and re-verify.

---

## Step 5 — Test it (the real proof is a power-off)

```bash
# Warm test: open an identifiable app + a terminal with unsaved scratch text, then:
systemctl hibernate
# Power back on. If the exact session returns (not the DE reopening apps fresh) -> good.
```

Then the **decisive test**: hibernate, and **physically cut power** (PSU switch or
pull the cord) for a few minutes. DRAM loses its contents within seconds, so a
successful restore after that proves the state came off the disk image with genuine
zero draw — true S4. Confirm the resume log is clean afterward:

```bash
sudo journalctl -b -1 -k | grep -iE 'pm:|nvidia|freeze|hibernat|preserve' | tail -40
# The 'pci_pm_freeze ... returns -5' line should be GONE.
```

---

## Step 6 — Harden (mostly already done)

The `99-` override *is* the hardening: it survives regeneration of the generator's
file, so a future nvidia/generator update that re-injects nvidia into early KMS gets
stripped again on the next rebuild. Explain to the user:

- After any large update, it's worth re-running the **Step 4** verification once.
- Manually booting an old rollback/history entry will behave like the old bug — that
  image predates the fix. Normal boots are unaffected.
- Bootloader updates (e.g. Limine point releases) are safe: the fix lives in the
  initramfs config and the `resume=` cmdline, not in the bootloader. The
  `resume_offset` only changes if the swapfile is recreated or moved.

---

## Reverting

```bash
sudo rm /etc/mkinitcpio.conf.d/99-no-nvidia-early-kms.conf   # (or restore the dracut setting)
sudo limine-update                                           # or the distro's rebuild command
```

NVIDIA returns to early KMS on next boot (and hibernate-resume breaks again).

---

## If it still fails after Step 4 shows a clean image

Early KMS was the confirmed root cause on the reference machine, but if a clean
initramfs still gives `-5`, work these **one at a time**, logging each:

1. **`PreserveVideoMemoryAllocations=0`** — disables VRAM-preservation across
   hibernate (you lose GPU state preservation, but the freeze may stop failing). Set
   via a modprobe.d drop-in and rebuild.
2. **NVIDIA PM systemd services** — try *enabling*
   `nvidia-suspend.service`, `nvidia-resume.service`, `nvidia-hibernate.service`
   (and `nvidia-suspend-then-hibernate.service` on newer drivers). Some driver
   versions need the classic service path rather than the kernel-notifier path
   (`UseKernelSuspendNotifiers=1`).
3. **Kernel ↔ driver pairing** — the `-5` regression tracks specific
   kernel/nvidia-open combinations. Boot an alternate installed kernel (e.g. LTS) to
   test a different pairing.
4. **A userspace process still holding `/dev/nvidia*`** — check with
   `sudo fuser -v /dev/nvidia*` if something blocks the freeze.

Do not apply these together. Read the versions off the machine; treat every forum
"fix" as a hypothesis to test and log on *this* hardware, not as ground truth.
