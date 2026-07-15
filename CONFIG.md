# Configuration file

`jump` is driven by an INI-style configuration file. This document describes
the file format in full: the cross-cutting `%...%` reference syntax, then every
section type, with each section type ending in a worked example. Two complete
end-to-end example configurations are given at the end.

## File format

The configuration is plain text with the following rules:

- **Section headers** are written `[name]`. Section names are hierarchical via
  dots, e.g. `[action.zpool.main]`, `[userinput.drives.target]`.
- **Key/value lines** are written `key = value`. Keys match
  `[a-zA-Z_][a-zA-Z0-9_.-]*`. Leading/trailing whitespace around both the key
  and value is trimmed.
- **Comments** start with `#` or `;`. Everything from the comment character to
  the end of the line is ignored. (To put a literal `#`/`;` in a value, use the
  multiline form below, where comment stripping is disabled.)
- **Quoting:** if a value is wrapped in exactly one pair of surrounding double
  quotes, those quotes are stripped. Embedded quotes are preserved as data.
- **Multiline values:** any line that *begins with a literal tab character*
  (`\t`) is treated as a continuation of the previous key. The leading tab is
  stripped and the rest is appended verbatim (no comment stripping, no quote
  processing) to the previous value, joined with a newline. This is the
  recommended way to write multi-line file `content`:
  ```
  content = #!/bin/sh
  \techo hello
  \texec /sbin/init
  ```
- **Duplicate keys** within a section are a validation error.

## Execution order

Actions execute in the order their `[action.*]` sections appear in the
configuration. `[userinput.*]` sections are also processed in file order when
collecting answers. Plan the order so that devices referenced by a later action
are created by an earlier one (e.g. `partition` before `ext4`, `zpool` before
`zfs`, `zfs`/`ext4`/`vfat` before any action that targets them).

---

## Reference syntax

Many config values contain `%...%` references that are expanded before use.
References come in two families: **user-input variables** (the answers to
`[userinput.*]` sections) and **built-in references** (resolved from the running
system). All references are expanded recursively.

| Reference                        | Resolves to                                                                 |
| -------------------------------- | --------------------------------------------------------------------------- |
| `%varname%`                      | The scalar answer stored under `varname` (password, generic, ipv4address, ipv4netmask). |
| `%varname[idx]part%`             | Drive device path with partition number. `varname` is a `userinput.drives` variable, `idx` is the 0-based index into the chosen drives, and `part` is the partition number, appended to the device path. E.g. `%drives[0]3%` → `…/drives[0]` + partition 3. |
| `%varname[idx]%`                 | Whole-drive device path (no partition).                                     |
| `%active%`                       | The network interface that currently holds the default IPv4 route.        |
| `%mac:XX:XX:XX:XX:XX:XX%`        | The network interface whose MAC address matches.                          |
| `%diskid:<glob>%`                | Disk(s) in `/dev/disk/by-id/` whose symlink name matches the glob (partition symlinks are skipped). E.g. `%diskid:ata-WDC*%`. |
| `%disksize:<spec>%`             | Disk(s) whose size matches `spec` within a tolerance of ±10% or ±10GiB, whichever is larger. `spec` is a size string, e.g. `%disksize:500GB%`, `%disksize:1TiB%`. |
| `%memory:<factor>[,<unit>]%`    | A portion of total system memory, computed as `factor × MemTotal`. `factor` is a positive decimal (e.g. `0.9`, `1.5`). `unit` is optional (`B`, `K`/`KB`/`KiB`, `M`/`MB`/`MiB`, `G`/`GB`/`GiB`, `T`/`TB`/`TiB`); if omitted the value is in bytes. E.g. `%memory:0.9,GiB%`. |
| `%%`                             | A literal `%` character. Use this when a value must contain a percent sign that is not a reference. |

Notes on units: single-letter suffixes (`K`, `M`, `G`, `T`) and the `KB`/`MB`/
`GB`/`TB` forms are SI (powers of 10); the `KiB`/`MiB`/`GiB`/`TiB` forms are
binary (powers of 1024).

### User-defined variables: `[var.name]`

You can define your own variables with `[var.name]` sections and reference them
as `%name%`. Variable values may themselves reference other variables (forward
references are supported). See the `[var.name]` section below for details.

### Auto-defined variables (URL configs)

When the configuration is fetched from a URL, two variables are defined
automatically and may be referenced like any `[var.*]` variable:

- `%starturl%` — the base URL of the config: everything up to and including the
  last `/`. For `http://example.com/js/jump` this is `http://example.com/js/`.
- `%startbaseurl%` — the scheme and host (with port if present):
  `http://example.com` (or `http://example.com:8080`).

These are useful for building relative URLs to archives and includes from a
single base location.

### Target references

Several actions operate on a filesystem that was created by an earlier action.
They take a `target` key whose value is a **target reference** of the form
`<type>.<instance>`, where:

- `<type>` is one of `zfs`, `ext4`, `vfat`, and
- `<instance>` is the instance name of a matching `[action.<type>.<instance>]`
  section.

For example, if you have `[action.zfs.root_debian]`, a later action may target
it with `target = zfs.root_debian`. The target action is mounted at a temporary
mount point, the operation is performed, and it is unmounted again.

### Reserved variable names

Because they are used by built-in references or action locals, the following
names may not be used as `[var.*]` names or `userinput` variable names:
`active`, `mac`, `diskid`, `disksize`, `memory`, `starturl`, `startbaseurl`
(the latter two only reserved when not auto-defined), and `drive` (used as an
action-local variable by `[action.efibootmgr]`).

---

## `[global]`

The `[global]` section holds top-level metadata. It is required.

| Key           | Required | Description                                                  |
| ------------- | -------- | ------------------------------------------------------------ |
| `description` | yes      | A human-readable description of this configuration. Shown to the operator in interactive mode and in the validation summary. |

### Example

```ini
[global]
description = Debian 12 web host, single SSD, UEFI boot
```

---

## `[var.name]`

A user-defined variable. Variables are expanded everywhere (in prompts, action
keys, include paths, and other variable values) before validation and
execution. Variable names must match `[a-zA-Z_][a-zA-Z0-9_]*` and must not
collide with reserved names (see *Reference syntax*) or any `userinput`
variable.

| Key    | Required | Description                       |
| ------ | -------- | --------------------------------- |
| `value` | yes      | The variable's value. May contain `%othervar%` references (forward references allowed) and built-in references. |

Action sections may also define variables locally with `var.<name> = value`
keys (or, for backward compatibility, the legacy `var = name:value` shorthand).
These are collected together with `[var.*]` sections and removed from the
config before validation, so they are available as `%name%` everywhere but are
not passed to the action as a normal key.

### Example

```ini
[var.arch]
value = amd64

[var.rootfs_url]
value = %starturl%rootfs-debian-12-%arch%.tar.zst
```

---

## `[include <path-or-url>]`

A directive (not a normal section) that pulls in another file inline. The line
must be exactly `[include <source>]`, where `<source>` is a local path or an
`http://`/`https://` URL.

- Relative paths are resolved against the directory of the file that contains
  the `[include ...]` line (URL-fetched files have no directory context, so
  relative includes from them must be absolute URLs or resolved via `%var%`).
- `%var%` references in the include source are expanded using the `[var.*]`
  values seen *before* that include, so you can build include paths from
  variables.
- Includes are resolved recursively, depth-first, up to a maximum depth of 5 to
  prevent infinite loops. Each included file's contents replace the
  `[include ...]` line in place.

### Example

```ini
[var.site]
value = http://boot.example.com/jumpstart

[include %site%/common/zfs-base.cfg]
[include %site%/common/debian-files.cfg]
```

---

## `[userinput.*]` — inputs

Each `[userinput.<type>.<name>]` section declares one input the operator must
provide. In interactive mode the input is collected with a `whiptail` dialog
of the appropriate kind; in unattended mode the value is read from the
`[answers]` section of the answers file.

All `userinput` sections share two required keys:

| Key       | Required | Description                                                                 |
| --------- | -------- | --------------------------------------------------------------------------- |
| `variable` | yes      | The variable name under which the answer is stored and referenced as `%variable%`. Must match `[a-zA-Z_][a-zA-Z0-9_]*` and be unique across all `userinput` sections and `[var.*]` definitions. |
| `prompt`   | yes      | The text shown to the operator. May contain `%...%` references, expanded before display. |

The remainder of the keys depend on the `<type>`, described below. The valid
types are: `password`, `drives`, `generic`, `ipv4address`, `ipv4netmask`, and
`netinterface`.

### `userinput.password`

Collects a password with a confirmation prompt. The answer is stored as a
scalar string.

| Key        | Required | Description |
| ---------- | -------- | ----------- |
| `variable` | yes      | (common)    |
| `prompt`   | yes      | (common)    |

#### Example

```ini
[userinput.password.rootpw]
variable = rootpw
prompt   = Enter the root password for the new system
```

### `userinput.drives`

Lets the operator select exactly `number` whole disks from a checklist of all
detected block devices (shown with size, vendor, model, and stable
`/dev/disk/by-id` path). The answer is stored as a space-separated list of
device paths and is indexed with `%varname[idx]%` / `%varname[idx]part%`.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `variable` | yes      | (common)    |
| `prompt`   | yes      | (common)    |
| `number`   | yes      | A positive integer: the exact number of drives the operator must select. |

#### Example

```ini
[userinput.drives.target]
variable = targetdisks
prompt   = Select the two disks for the mirror
number   = 2
```

### `userinput.generic`

Collects a free-form text string with optional length and character constraints.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `variable` | yes      | (common)    |
| `prompt`   | yes      | (common)    |
| `length`   | no       | A positive integer: maximum length in characters. |
| `type`     | no       | `any` (default) or `numeric`. When `numeric`, the input may contain only digits. |

#### Example

```ini
[userinput.generic.hostname]
variable = hostname
prompt   = Hostname for the new system
length   = 63
type     = any
```

### `userinput.ipv4address`

Collects and validates a dotted-decimal IPv4 address (each octet 0–255).

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `variable` | yes      | (common)    |
| `prompt`   | yes      | (common)    |

#### Example

```ini
[userinput.ipv4address.ip]
variable = ipaddr
prompt   = IPv4 address for the management interface
```

### `userinput.ipv4netmask`

Collects an IPv4 netmask. Accepts dotted-decimal (`255.255.255.0`), a bare CIDR
prefix length (`24`), or a CIDR prefix with a leading slash (`/24`). The value
is normalized to dotted-decimal before being stored.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `variable` | yes      | (common)    |
| `prompt`   | yes      | (common)    |

#### Example

```ini
[userinput.ipv4netmask.mask]
variable = netmask
prompt   = IPv4 netmask (e.g. 255.255.255.0 or /24)
```

### `userinput.netinterface`

Lets the operator select exactly `number` network interfaces from a checklist
of non-loopback interfaces (shown with state, MAC, link speed, and any IPv4
addresses). The answer is stored as a space-separated list and indexed with
`%varname[idx]%`.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `variable` | yes      | (common)    |
| `prompt`   | yes      | (common)    |
| `number`   | yes      | A positive integer: the exact number of interfaces the operator must select. |

#### Example

```ini
[userinput.netinterface.mgmt]
variable = mgmtif
prompt   = Select the management interface
number   = 1
```

---

## `[action.*]` — actions

Each `[action.<type>.<instance>]` section performs one step. The `<instance>`
suffix is required for every action type (it names this particular instance so
multiple actions of the same type and cross-references like `target` are
unambiguous). Actions run in the order they appear in the file.

Supported types, in their canonical order:

`wipe`, `partition`, `crypt`, `zpool`, `zfs`, `extract`, `file`, `filemodify`, `ext4`,
`vfat`, `swap`, `efibootmgr`, `password`, `sshhostkeys`, `mkdir`,
`netifrename`, `md`, `mdadm`, `crypttab`, `updateinitramfs`, `authorizedkeys`.

Common conventions:

- **Drive references** (`drive` keys) accept either a `%varname[idx]part%`
  reference (must include a partition number where required), a literal
  device path such as `/dev/sda1`, or a **crypt back-reference**
  `crypt.<instance>` naming an earlier `[action.crypt.<instance>]` section
  (see `action.crypt` below). The LUKS device is opened lazily the first time
  it is referenced and closed on exit (like MD arrays).
- **`target` keys** reference an earlier `zfs`/`ext4`/`vfat` action as
  `<type>.<instance>` (see *Reference syntax* › Target references). If that
  filesystem was created on a `crypt.<instance>` device, mounting it
  transparently opens the LUKS device (again, closed on exit).
- **`options` keys**, where present, are space-separated argument lists passed
  through to the underlying tool.

### `action.wipe`

Wipe one or more whole drives completely: partition tables, ZFS labels, and LVM
metadata are destroyed. ZFS pools using the drives are exported and MD arrays
are stopped first. This is destructive and is confirmed separately in
interactive mode.

| Key     | Required | Description |
| ------- | -------- | ----------- |
| `drives` | yes      | A single reference to a `userinput.drives` variable, e.g. `%targetdisks%`. All selected drives are wiped. |

#### Example

```ini
[action.wipe.everything]
drives = %targetdisks%
```

### `action.partition`

Create a partition table on each selected drive.

| Key        | Required | Description |
| ---------- | -------- | ----------- |
| `drives`   | yes      | A single `%drivesvar%` reference. All selected drives are partitioned identically. |
| `type`     | no       | `gpt` (default) or `mbr`. `gpt` uses `sgdisk`; `mbr` uses `sfdisk` with a `dos` label. |
| `part-<n>` | yes (≥1) | One key per partition, numbered sequentially from `1` (`part-1`, `part-2`, …). The value is `<size> <typecode>`. `<size>` is e.g. `512MB`, `1.5GB`, a `%variable%` reference, or the keyword `remain` (uses all remaining space; at most one partition, and it must be the highest-numbered). `<typecode>` is a 2–4 digit hex GPT type code, e.g. `8300` (Linux filesystem), `8200` (Linux swap), `EF00` (EFI System Partition). |

#### Example

```ini
[action.partition.main]
drives  = %targetdisks%
type    = gpt
part-1  = 512MB EF00
part-2  = remain 8300
```

### `action.crypt`

Create a LUKS-encrypted device with `cryptsetup luksFormat`. The instance
name is the **mapper name** the device will be opened as
(`/dev/mapper/<instance>`). The device is **not** opened by this action; it is
opened lazily the first time a later action references it via a
`crypt.<instance>` drive back-reference, and closed again on exit (like MD
arrays created by `action.md`).

| Key         | Required | Description |
| ----------- | -------- | ----------- |
| `drive`     | yes      | The underlying device to encrypt: a `%varname[idx]part%` reference (with partition) or a literal path such as `/dev/sda2`. A whole-drive reference (no partition) is also accepted. |
| `passphrase` | yes     | The LUKS passphrase. **Must contain at least one `%variable%` reference** (typically a `userinput.password` variable), like `action.password` — a literal passphrase is a validation error. It is fed to `cryptsetup` via a temporary key-file, so it never appears in `argv` or logs. |
| `options`   | no       | Space-separated extra arguments passed to `cryptsetup luksFormat` (after `--key-file`), e.g. `--type luks2 --cipher aes-xts-plain64`. |

Other actions can target the encrypted device by using `crypt.<instance>`
wherever a `drive` is accepted (`ext4`/`vfat`/`swap`, `zpool` `vdev`, `md`
member `drives`, and `efibootmgr` single-`drive`). At runtime `jump` runs
`cryptsetup luksOpen` on first use and substitutes `/dev/mapper/<instance>`;
any `ext4`/`vfat`/`zfs` action created on the mapper may then be used as a
normal `target` by later actions.

#### Example

```ini
[action.crypt.root]
drive      = %targetdisks[0]2%
passphrase = %luks_passphrase%
options    = --type luks2

[action.ext4.system]
drive = crypt.root
label = system
```

### `action.zpool`

Create a ZFS pool. The instance name is the **pool name**.

| Key          | Required | Description |
| ------------ | -------- | ----------- |
| `vdev`       | yes      | Space-separated list of vdev references. Each is a drive reference `%varname[idx]part%` (with partition) or `%varname[idx]%` (whole drive). |
| `fsoptions`  | no       | Space-separated pool options (`-o` to `zpool create`), e.g. `ashift=12`. |
| `options`    | no       | Space-separated root-dataset properties (`-O` to `zpool create`), e.g. `mountpoint=/`. |

#### Example

```ini
[action.zpool.rpool]
vdev      = %targetdisks[0]2%
fsoptions = ashift=12
options   = mountpoint=/ canmount=noauto atime=off
```

### `action.zfs`

Create a ZFS dataset inside an existing pool. The dataset is created unmounted
(`zfs create -u -p`).

| Key          | Required | Description |
| ------------ | -------- | ----------- |
| `pool`       | yes      | The pool name; must match the instance name of an `[action.zpool.<pool>]` section. |
| `dataset`    | yes      | The dataset name (appended as `<pool>/<dataset>`). |
| `properties` | no       | Space-separated dataset properties (`-o` to `zfs create`), e.g. `mountpoint=/root compression=lz4`. |

#### Example

```ini
[action.zfs.root_debian]
pool       = rpool
dataset    = ROOT/debian
properties = mountpoint=/ canmount=noauto
```

### `action.extract`

Download a tar archive (with `curl`) and stream-extract it onto a target
filesystem.

| Key      | Required | Description |
| -------- | -------- | ----------- |
| `source` | yes      | URL or path of the tar archive to fetch and extract. |
| `target` | yes      | Target reference (`zfs`/`ext4`/`vfat.<instance>`) where the archive is extracted. |

#### Example

```ini
[action.extract.rootfs]
source = %rootfs_url%
target = zfs.root_debian
```

### `action.file`

Write a new file to a target filesystem. Parent directories are created as
needed. The `content` value is `%...%`-expanded before writing.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `target`  | yes      | Target reference. |
| `file`    | yes      | Path of the file to write, relative to the target filesystem root (e.g. `/etc/hostname`). |
| `content` | yes      | The file's contents. Use tab-continuation lines for multi-line content (see *File format*). |

#### Example

```ini
[action.file.hostname]
target  = zfs.root_debian
file    = /etc/hostname
content = %hostname%
```

### `action.filemodify`

Modify an existing file on a target filesystem. File mode and ownership are
preserved. The `content` value is `%...%`-expanded before use.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `target`  | yes      | Target reference. |
| `file`    | yes      | Path of the existing file to modify. |
| `action`  | yes      | One of `prepend`, `append`, or `insert`. |
| `content` | yes      | The text to prepend/append/insert. |
| `anchor`  | yes iff `action = insert` | The file is scanned line by line; `content` is inserted immediately after the first line containing `anchor`. An error is raised if the anchor is not found. |

#### Example

```ini
[action.filemodify.add_module]
target  = zfs.root_debian
file    = /etc/modules
action  = append
content = zfs
```

### `action.ext4` / `action.vfat` / `action.swap`

Create an ext4, vfat, or swap filesystem on a partition. These three share the
same keys (only the underlying tool differs: `mkfs.ext4`, `mkfs.vfat`,
`mkswap`).

| Key      | Required | Description |
| -------- | -------- | ----------- |
| `drive`  | yes      | A drive reference with partition, e.g. `%targetdisks[0]1%`, or a literal path like `/dev/sda1`. |
| `label`  | no       | Filesystem/swap label. Passed as `-L` for ext4/swap, `-n` for vfat. |
| `options`| no       | Space-separated extra arguments passed to the `mkfs`/`mkswap` command. |

#### Example (ext4)

```ini
[action.ext4.system]
drive = %targetdisks[0]2%
label = system
```

#### Example (vfat)

```ini
[action.vfat.efi]
drive = %targetdisks[0]1%
label = EFI
```

#### Example (swap)

```ini
[action.swap.swap0]
drive = %targetdisks[1]1%
label = swap0
```

### `action.efibootmgr`

Create one or more EFI boot manager entries. There are two forms:

- **Single drive:** `drive` references one partition (the EFI System
  Partition). One boot entry is created.
- **Multiple drives:** `drives` references a `userinput.drives` variable and
  `partition` gives the partition number on each. One boot entry is created per
  drive. While iterating, the action-local variable `%drive%` expands to the
  current drive's path and may be used inside `label` and `path`.

`drive` and `drives` are mutually exclusive; exactly one must be given.

| Key         | Required | Description |
| ----------- | -------- | ----------- |
| `drive`     | one of   | Single-drive form: a drive reference with partition, e.g. `%targetdisks[0]1%` or `/dev/sda1`. |
| `drives`    | one of   | Multi-drive form: a single `%drivesvar%` reference. Requires `partition`. |
| `partition` | yes with `drives` | A positive integer: the partition number to use on each drive. Must not be combined with `drive`. |
| `label`     | yes      | EFI boot entry label. Supports `%...%` expansion (including `%drive%` in multi-drive form). |
| `path`      | yes      | EFI loader path, e.g. `\EFI\debian\grubx64.efi`. Supports `%...%` expansion (including `%drive%`). |

#### Example (single drive)

```ini
[action.efibootmgr.debian]
drive = %targetdisks[0]1%
label = Debian
path  = \EFI\debian\grubx64.efi
```

#### Example (multi-drive, mirrored EFI)

```ini
[action.efibootmgr.debian_mirror]
drives    = %targetdisks%
partition = 1
label     = Debian (%drive%)
path      = \EFI\debian\grubx64.efi
```

### `action.password`

Set a user's password on a target filesystem by replacing the hash in
`/etc/shadow` (the user must already exist in `/etc/passwd`). The password is
hashed with `openssl passwd -6` (SHA-512).

| Key        | Required | Description |
| ---------- | -------- | ----------- |
| `target`   | yes      | Target reference (the filesystem containing `/etc/passwd` and `/etc/shadow`). |
| `user`      | yes      | The username whose password is set. Must exist in `/etc/passwd` on the target. |
| `password`  | yes      | The password value. Must contain at least one `%variable%` reference (typically a `userinput.password` variable). |

#### Example

```ini
[action.password.root]
target   = zfs.root_debian
user     = root
password = %rootpw%
```

### `action.sshhostkeys`

Generate SSH host keys on a target filesystem in `/etc/ssh`. Existing keys of a
given type are left untouched.

| Key     | Required | Description |
| ------- | -------- | ----------- |
| `target` | yes      | Target reference. |
| `types`  | no       | Space-separated subset of `rsa`, `ecdsa`, `ed25519`. If omitted, all three types are generated. |

#### Example

```ini
[action.sshhostkeys.main]
target = zfs.root_debian
types  = ed25519 rsa
```

### `action.authorizedkeys`

Append one or more public keys to `/root/.ssh/authorized_keys` on a target
filesystem. Creates `/root/.ssh` (mode `0700`, owned by root) if needed and
enforces `0600` on `authorized_keys`. Entries already present are skipped
(idempotent). The `content` value is `%...%`-expanded; one key per line.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `target`  | yes      | Target reference. |
| `content` | yes      | One public key per line (use tab-continuation for multiple lines). |

#### Example

```ini
[action.authorizedkeys.admin]
target  = zfs.root_debian
content = ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... admin@jumpstart
```

### `action.mkdir`

Create a directory (and parents) on a target filesystem. The `directory` value
is `%...%`-expanded.

| Key         | Required | Description |
| ----------- | -------- | ----------- |
| `target`    | yes      | Target reference. |
| `directory` | yes      | Path to create, relative to the target filesystem root. |

#### Example

```ini
[action.mkdir.srv]
target    = zfs.root_debian
directory = /srv/app
```

### `action.netifrename`

Write a systemd `.link` file on a target filesystem that renames a network
interface to a stable name (matched by the interface's MAC address, read from
the running system).

| Key         | Required | Description |
| ----------- | -------- | ----------- |
| `target`    | yes      | Target reference. |
| `interface` | yes      | The interface to rename. May be a literal name, a `%netinterfacevar[idx]%` reference, or a built-in `%active%` / `%mac:XX:XX:XX:XX:XX:XX%`. |
| `name`      | yes      | The new interface name. Must match `[a-zA-Z][a-zA-Z0-9_]*`. |

#### Example

```ini
[action.netifrename.mgmt]
target    = zfs.root_debian
interface = %mgmtif[0]%
name      = mgmt0
```

### `action.md`

Assemble an MD RAID device with `mdadm --create`.

| Key          | Required | Description |
| ------------ | -------- | ----------- |
| `device`     | yes      | The MD device name (without `/dev/`), e.g. `md0`. The device is created as `/dev/md0`. |
| `level`      | yes      | RAID level, e.g. `1`, `5`, `10`. |
| `numdevices` | yes      | A positive integer: the number of active (`--raid-devices`) members. |
| `drives`     | yes      | Space-separated list of member devices. Each may be a drive reference (`%var[idx]part%`), a literal `/dev/...` path, or the keyword `missing` (a sparse slot). |
| `bitmap`     | no       | Passed as `--bitmap=` to `mdadm`. Defaults to `none`, which also suppresses mdadm's interactive prompt. |
| `options`    | no       | Space-separated extra `mdadm` arguments. |

The created device is tracked so it is stopped on exit.

#### Example

```ini
[action.md.md0]
device     = md0
level      = 1
numdevices = 2
drives     = %targetdisks[0]2% %targetdisks[1]2%
```

### `action.mdadm`

Write `/etc/mdadm/mdadm.conf` (containing `AUTO -all` and the output of
`mdadm --detail --scan`) onto a target filesystem so the installed system
assembles the same arrays on boot.

| Key      | Required | Description |
| -------- | -------- | ----------- |
| `target` | yes      | Target reference (the root filesystem). |

#### Example

```ini
[action.mdadm.root]
target = ext4.system
```

### `action.crypttab`

Write `/etc/crypttab` onto a target filesystem so the installed system opens
the same LUKS devices at boot. One line is written per referenced crypt
instance, in the form:

```
<instance>  UUID=<uuid>  none  <options>
```

The UUID is resolved with `blkid` from the `drive` of each referenced
`[action.crypt.<instance>]` section. The `keyfile` field is always `none`
(passphrase prompted at boot). The `options` field depends on `remember`:

- `remember = false` (default): `luks,discard`
- `remember = true`: `luks,discard,initramfs,keyscript=decrypt_keyctl` —
  used when several devices share the same passphrase so it only has to be
  entered once during early boot.

| Key       | Required | Description |
| --------- | -------- | ----------- |
| `target`  | yes      | Target reference (the root filesystem where `/etc/crypttab` is created). |
| `drives`  | yes      | Space-separated list of `crypt.<instance>` back-references; each must match an existing `[action.crypt.<instance>]` section. |
| `remember` | no      | `true` or `false` (default `false`). See the options field above. |

The file is written fresh (any pre-existing `/etc/crypttab` in the target is
overwritten). Place this action before any `action.updateinitramfs` so the
initramfs regenerated afterwards picks up the crypttab.

#### Example

```ini
[action.crypttab.root]
target   = ext4.system
drives   = crypt.root crypt.swap
remember = true
```

### `action.updateinitramfs`

Mount the target root filesystem, optionally mount additional filesystems
inside it, bind-mount `/proc`, `/sys`, `/sys/firmware/efi/efivars`, and `/dev`,
then `chroot` into it and run `update-initramfs -u -k all`.

| Key      | Required | Description |
| -------- | -------- | ----------- |
| `target` | yes      | Target reference for the root filesystem. |
| `mountN` | no       | Additional filesystem to mount inside the chroot. `<N>` is any non-negative integer; entries are sorted numerically. The value is `<source> <mountpoint>`, where `<mountpoint>` is an absolute path inside the chroot (must start with `/`). `<source>` may be a target reference (`zfs`/`ext4`/`vfat.<instance>`), a `/dev/...` literal, or a single `%variable%` reference. |

#### Example

```ini
[action.updateinitramfs.root]
target = zfs.root_debian
mount1 = vfat.efi /boot/efi
```

---

## `[answers]` — unattended answers

Used only in unattended mode, in the **answers file** (the file passed to
`--unattended`, or named by `jumpunattended=`). It supplies one value per
`[userinput.*]` variable. Every `userinput` variable must have an answer.

The form of each value depends on the corresponding `userinput` type:

- **Scalar types** (`password`, `generic`, `ipv4address`, `ipv4netmask`): a
  single string. `ipv4address` must be a valid dotted-decimal address;
  `ipv4netmask` accepts dotted-decimal or a CIDR prefix (`24`/`/24`), normalized
  to dotted-decimal; `generic` honors the `length` and `type` constraints of
  its `userinput.generic` section.
- **`drives`:** a space-separated list of device paths. Built-in references
  like `%diskid:...%` and `%disksize:...%` are expanded, then the list is
  split. If more paths than `number` are given, the extras are trimmed; fewer
  than `number` is an error.
- **`netinterface`:** a space-separated list of interface names. Built-in
  references like `%active%` and `%mac:...%` are expanded, then split, with the
  same `number` enforcement as `drives`.

### Example

```ini
[answers]
rootpw     = S3cret!
hostname   = webhost01
ipaddr     = 192.168.1.10
netmask    = 24
targetdisks = %diskid:ata-Samsung*% %diskid:ata-Samsung*%
mgmtif     = %mac:00:11:22:33:44:55%
```

---

## `[done]` — unattended completion

Used only in unattended mode (in the answers file). It tells `jump` what to do
when finished and how to report the result. It is parsed *before* the main
configuration, so its `failaction` is honored even if the main config fails to
fetch or validate.

| Key          | Required | Description |
| ------------ | -------- | ----------- |
| `action`     | yes      | What to do after a successful run: `reboot`, `poweroff`, or `wait` (drop to a shell). Before reboot/poweroff, any ZFS pools created by this script are exported and any MD devices created are stopped. |
| `failaction` | no       | What to do after a failed run. Same vocabulary as `action`. Defaults to `wait`. |
| `feedback`   | no       | Path to an executable script invoked with the run result. It is called as: `feedback <config-url-or-path> <answers-url-or-path> <status> <logfile-path>`, where `<status>` is `success` or `failure`. A non-zero exit from the feedback script is logged as a warning but does not change `jump`'s behavior. |

### Example

```ini
[done]
action     = reboot
failaction = wait
feedback   = /usr/local/bin/report-jump-result
```

---

## Complete end-to-end examples

The following two complete configurations demonstrate a typical interactive
single-disk ext4/UEFI install and a fully unattended ZFS mirror install. They
are self-contained and validate with `jump --validate`.

### Example 1 — interactive single-disk ext4 + UEFI

Save as `webhost.cfg` and run with `jump webhost.cfg`.

```ini
[global]
description = Debian 12 web host on a single SSD (interactive)

# --- Variables ---
[var.arch]
value = amd64

[var.rootfs_url]
value = http://boot.example.com/images/debian-12-rootfs-%arch%.tar.zst

# --- Inputs (collected interactively) ---
[userinput.drives.target]
variable = targetdisks
prompt   = Select the SSD to install onto
number   = 1

[userinput.generic.hostname]
variable = hostname
prompt   = Hostname for the new system
length   = 63

[userinput.password.rootpw]
variable = rootpw
prompt   = Root password for the new system

# --- Actions (executed in order) ---
[action.wipe.everything]
drives = %targetdisks%

[action.partition.main]
drives  = %targetdisks%
type    = gpt
part-1  = 512MB EF00
part-2  = remain 8300

[action.vfat.efi]
drive = %targetdisks[0]1%
label = EFI

[action.ext4.system]
drive = %targetdisks[0]2%
label = system

[action.extract.rootfs]
source = %rootfs_url%
target = ext4.system

[action.file.hostname]
target  = ext4.system
file    = /etc/hostname
content = %hostname%

[action.password.root]
target   = ext4.system
user     = root
password = %rootpw%

[action.sshhostkeys.main]
target = ext4.system
types  = ed25519

[action.authorizedkeys.admin]
target  = ext4.system
content = ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIexamplekey admin@jumpstart

[action.efibootmgr.debian]
drive = %targetdisks[0]1%
label = Debian
path  = \EFI\debian\grubx64.efi

[action.updateinitramfs.root]
target = ext4.system
mount1 = vfat.efi /boot/efi
```

### Example 2 — unattended ZFS mirror with mirrored EFI

Two files are used: a main config (`dbhost.cfg`) and an answers file
(`dbhost.answers.cfg`). Run with:

```sh
jump --unattended dbhost.answers.cfg http://boot.example.com/jumpstart/dbhost.cfg
```

`dbhost.cfg`:

```ini
[global]
description = Debian 12 DB host, ZFS mirror, unattended

# --- Variables (URL-relative, since the config is fetched over HTTP) ---
[var.rootfs_url]
value = %starturl%images/debian-12-rootfs-amd64.tar.zst

# Pull in shared snippets
[include %starturl%common/authorized-keys.cfg]

# --- Inputs (answers supplied in the answers file) ---
[userinput.drives.target]
variable = targetdisks
prompt   = Select the two disks for the mirror
number   = 2

[userinput.generic.hostname]
variable = hostname
prompt   = Hostname for the new system
length   = 63

[userinput.password.rootpw]
variable = rootpw
prompt   = Root password for the new system

[userinput.netinterface.mgmt]
variable = mgmtif
prompt   = Select the management interface
number   = 1

# --- Actions ---
[action.wipe.everything]
drives = %targetdisks%

[action.partition.main]
drives  = %targetdisks%
type    = gpt
part-1  = 512MB EF00
part-2  = remain 8300

[action.zpool.rpool]
vdev      = %targetdisks[0]2% %targetdisks[1]2%
fsoptions = ashift=12
options   = mountpoint=/ canmount=noauto atime=off

[action.zfs.root_debian]
pool       = rpool
dataset    = ROOT/debian
properties = mountpoint=/ canmount=noauto

[action.vfat.efi0]
drive = %targetdisks[0]1%
label = EFI0

[action.vfat.efi1]
drive = %targetdisks[1]1%
label = EFI1

[action.extract.rootfs]
source = %rootfs_url%
target = zfs.root_debian

[action.file.hostname]
target  = zfs.root_debian
file    = /etc/hostname
content = %hostname%

[action.password.root]
target   = zfs.root_debian
user     = root
password = %rootpw%

[action.netifrename.mgmt]
target    = zfs.root_debian
interface = %mgmtif[0]%
name      = mgmt0

# authorizedkeys.admin is pulled in by common/authorized-keys.cfg

[action.sshhostkeys.main]
target = zfs.root_debian
types  = ed25519 rsa

[action.efibootmgr.debian_mirror]
drives    = %targetdisks%
partition = 1
label     = Debian (%drive%)
path      = \EFI\debian\grubx64.efi

[action.updateinitramfs.root]
target = zfs.root_debian
mount1 = vfat.efi0 /boot/efi
```

`dbhost.answers.cfg`:

```ini
[answers]
targetdisks = %disksize:1TB%
hostname    = dbhost01
rootpw      = S3cret!
mgmtif      = %mac:00:11:22:33:44:55%

[done]
action     = reboot
failaction = wait
feedback   = /usr/local/bin/report-jump-result
```

In this example `%disksize:1TB%` selects both ~1 TB disks (the reference returns
all matching disks), `targetdisks` is trimmed to the required 2, and the
mirrored EFI entries are created on both disks with per-drive labels via
`%drive%`.

### Example 3 — interactive single-disk LUKS + ext4 + UEFI

A single SSD with an unencrypted EFI System Partition and an encrypted root
filesystem. Save as `crypthost.cfg` and run with `jump crypthost.cfg`.

```ini
[global]
description = Debian 12 host, LUKS-encrypted root (interactive)

[var.arch]
value = amd64

[var.rootfs_url]
value = http://boot.example.com/images/debian-12-rootfs-%arch%.tar.zst

[userinput.drives.target]
variable = targetdisks
prompt   = Select the SSD to install onto
number   = 1

[userinput.password.luks]
variable = luks_pass
prompt   = LUKS passphrase for the root device

[userinput.password.rootpw]
variable = rootpw
prompt   = Root password for the new system

[userinput.generic.hostname]
variable = hostname
prompt   = Hostname for the new system
length   = 63

[action.wipe.everything]
drives = %targetdisks%

[action.partition.main]
drives  = %targetdisks%
type    = gpt
part-1  = 512MB EF00
part-2  = remain 8300

[action.crypt.root]
drive      = %targetdisks[0]2%
passphrase = %luks_pass%

[action.vfat.efi]
drive = %targetdisks[0]1%
label = EFI

[action.ext4.system]
drive = crypt.root
label = system

[action.extract.rootfs]
source = %rootfs_url%
target = ext4.system

[action.file.hostname]
target  = ext4.system
file    = /etc/hostname
content = %hostname%

[action.password.root]
target   = ext4.system
user     = root
password = %rootpw%

[action.crypttab.root]
target   = ext4.system
drives   = crypt.root
remember = false

[action.updateinitramfs.root]
target = ext4.system
mount1 = vfat.efi /boot/efi
```

The `ext4.system` filesystem is created on the LUKS mapper `/dev/mapper/root`;
`jump` runs `cryptsetup luksOpen root` on first reference (the `ext4` action)
and `cryptsetup luksClose root` on exit. `crypttab.root` writes
`root UUID=… none luks,discard` so the installed system prompts for the
passphrase at boot, then `updateinitramfs` regenerates the initramfs with
`cryptsetup` support.