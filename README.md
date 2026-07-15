# jump

`jump` is a single, dependency-light Bash script for *jumpstarting* systems —
that is, preparing a target machine's disks and filesystems and seeding an
operating system onto them in a repeatable, declarative way. It is designed to
run from a minimal boot environment (a rescue/initramfs shell, a PXE boot, a
live image, etc.) and turn a handful of disks into a ready-to-boot installed
system without any manual `fdisk`/`zpool`/`mount` typing.

Instead of an imperative install script, `jump` is driven by a small
**INI-style configuration file**. The configuration describes:

- the **inputs** to collect from the operator (drives, passwords, network
  interfaces, IP addresses, …),
- the **actions** to perform on the target system (wipe, partition, create ZFS
  pools/datasets, create ext4/vfat/swap filesystems, extract a tarball,
  write/modify files, set passwords, generate SSH host keys, add authorized
  keys, create EFI boot entries, rename network interfaces, assemble MD RAID,
  refresh the initramfs, …), and
- in unattended mode, the **answers** to those inputs and what to do when
  finished.

`jump` then validates the configuration, collects the required inputs (either
interactively or from an answers file), and runs the actions in the order they
are declared.

## Key features

- **Declarative configuration.** Everything is described in one INI file:
  user inputs, actions, variables, and includes.
- **Two operating modes.**
  - *Interactive* — prompts the operator (via `whiptail` dialogs) for each
    declared input, then runs the actions with a progress display.
  - *Unattended* — reads answers from a separate answers file and runs to
    completion without any prompts, then reboots, powers off, or waits.
- **Validation mode.** `--validate` checks a configuration (and optionally an
  answers file) without touching the system, so recipes can be linted in CI.
- **Config from anywhere.** The configuration and answers files may be local
  paths or `http://`/`https://` URLs fetched with `curl`. Includes can pull in
  other files or URLs, so common recipes can be shared.
- **Kernel command-line integration.** When booting a dedicated jumpstart
  image, the configuration URL and unattended answers can be supplied with
  `jumpconfig=<url>` and `jumpunattended=<url>` on the kernel command line.
- **Rich, self-describing references.** Disk, network, and memory references
  such as `%active%`, `%mac:..%`, `%diskid:..%`, `%disksize:..%`, and
  `%memory:..%` let a single recipe target machines by their hardware
  identity rather than hard-coded device names.

## What jump is *not*

`jump` does not ship or install an OS image itself. It prepares the storage
and writes files (for example by extracting a prebuilt root filesystem tarball
via an `extract` action). The OS image, tarball, and any bootloader files are
provided by you and referenced from the configuration.

## Documentation

- **[CLI.md](CLI.md)** — command-line usage: every option, the positional
  configuration argument, kernel command-line parameters, and per-option
  examples.
- **[CONFIG.md](CONFIG.md)** — the configuration file format: the cross-cutting
  `%...%` reference syntax, and a per-section reference for every section type
  (`[global]`, `[var.*]`, `[include ...]`, `[userinput.*]`, `[action.*]`,
  `[answers]`, `[done]`), each ending with a worked example, plus two complete
  end-to-end example configurations.

## Quick start

1. Write a configuration file, e.g. `myhost.cfg` (see [CONFIG.md](CONFIG.md)
   for the full syntax, or copy one of the complete examples at the end).

2. Validate it before running anything destructive:

   ```sh
   jump --validate myhost.cfg
   ```

3. Run it interactively as root:

   ```sh
   jump myhost.cfg
   ```

   Or, to run fully unattended with an answers file:

   ```sh
   jump --unattended myhost.answers.cfg myhost.cfg
   ```

## Requirements

- Root privileges (except in `--validate` mode).
- `curl`, `lsblk`, and `tar` are always required.
- `whiptail` is required for interactive mode.
- Action-specific tools are required only when the corresponding action is
  used: `sgdisk`/`sfdisk` (partitioning), `cryptsetup` (LUKS),
  `zpool`/`zfs` (ZFS), `mkfs.ext4`/`mkfs.vfat`/`mkswap` (filesystems),
  `efibootmgr` (EFI entries), `openssl` (passwords), `ssh-keygen` (SSH host
  keys), `mdadm` (MD RAID), `blkid` (crypttab UUIDs),
  `chroot` + `update-initramfs` (initramfs refresh). `jump` checks for these
  at runtime and reports what is missing.