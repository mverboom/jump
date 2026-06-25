# Command-line interface

```
usage: jump [--validate] [--unattended [answers]] <config-url-or-path>
       jump --validate <config-url-or-path>
         Validate config (and answers) without running actions
       jump --unattended [answers] <config-url>
          Run unattended with answers file
       Alternatively, set jumpconfig=<url> on the kernel command line.
       If jumpunattended=<url> is on the kernel command line, unattended
       mode is enabled automatically (no --unattended flag needed).
```

`jump` takes a single positional configuration argument and two flags. Every
option can also be supplied from the kernel command line, which makes `jump`
suitable for dedicated jumpstart boot images.

## Synopsis

```
jump [--validate] [--unattended [answers-file]] <config-url-or-path>
```

- `<config-url-or-path>` is a local file path or an `http://`/`https://` URL.
  URL configurations are fetched with `curl` to a temporary file, processed,
  and removed on exit. Local files are used in place.
- Flags may appear in any order before the positional argument, except that
  `--unattended` may consume the next token as its optional answers-file
  argument (see below).
- Unknown flags (anything else starting with `-`) print the usage message and
  exit with a non-zero status.

## Positional argument: `<config-url-or-path>`

The configuration file describing the inputs to collect and the actions to
perform. See [CONFIG.md](CONFIG.md) for the file format.

- A local path is read directly:

  ```sh
  jump /etc/jump/myhost.cfg
  ```

- An `http://`/`https://` URL is fetched with `curl --fail` to a temporary
  file, parsed, and cleaned up on exit. When the config is fetched from a URL,
  `jump` automatically defines the `%starturl%` and `%startbaseurl%` variables
  (see [CONFIG.md](CONFIG.md) › *Reference syntax*) so the recipe can build
  relative URLs to archives and includes:

  ```sh
  jump http://boot.example.com/jumpstart/myhost.cfg
  ```

If no positional argument is given, `jump` falls back to the `jumpconfig=`
kernel parameter (below). If neither is present, it shows a dialog and exits.

### Example

```sh
jump /srv/jump/webhost.cfg
```

## `--validate`

Validate the configuration **without running any actions**. No filesystems are
mounted and nothing on the system is modified. This is the recommended way to
lint a recipe in CI or before pointing it at real hardware.

- With `--validate`, `jump` does **not** require root (it never runs actions).
- The full include/var-expansion pipeline runs, so URL configs, includes, and
  `%var%` references are validated exactly as they would be at run time.
- When combined with `--unattended [answers]`, the answers file is also parsed
  and checked against every `[userinput.*]` variable (each must have an
  answer), and the `[done]` section is validated.
- On success it prints `Configuration is valid.` (or
  `Configuration and answers are valid.` with `--unattended`) and exits `0`;
  on failure it prints `error:` lines and exits `1`.

### Examples

Validate a local config only:

```sh
jump --validate webhost.cfg
```

Validate a URL config and its answers file together:

```sh
jump --validate --unattended webhost.answers.cfg http://boot.example.com/jumpstart/webhost.cfg
```

## `--unattended [answers-file]`

Run in **unattended mode**: no `whiptail` prompts are shown. Answers are read
from an answers file (local path or URL) instead of being collected from the
operator. After all actions complete successfully, `jump` performs the
`[done] action` (reboot, poweroff, or wait). See [CONFIG.md](CONFIG.md) for the
`[answers]` and `[done]` sections.

- `--unattended` accepts an **optional** answers-file argument: the token
  immediately following the flag, as long as it does not itself start with
  `--`. This lets you write either form:

  ```sh
  jump --unattended webhost.answers.cfg webhost.cfg
  jump --unattended           webhost.cfg      # answers file comes from kernel cmdline
  ```

- The answers file has the same INI syntax as the main config. It must contain
  an `[answers]` section (one key per `[userinput.*]` variable) and a `[done]`
  section. It is parsed *before* the main config so that the `[done]`
  `failaction` is already known if a fatal error occurs while fetching or
  validating the main config.
- If no answers file is given on the command line, `jump` falls back to the
  `jumpunattended=` kernel parameter. If neither is present, it exits with an
  error.
- In unattended mode the log file is **kept** after exit (under `$TMPDIR` or
  `/tmp`) for diagnostics. (In interactive mode the log is treated as a
  temporary file and removed.)
- `whiptail` is *not* required in unattended mode.

### Examples

Run unattended with an answers file next to the config:

```sh
jump --unattended webhost.answers.cfg webhost.cfg
```

Run unattended where the answers file is served over HTTP:

```sh
jump --unattended http://boot.example.com/jumpstart/webhost.answers.cfg \
     http://boot.example.com/jumpstart/webhost.cfg
```

## Kernel command-line parameters

When `jump` is invoked from a dedicated boot environment (e.g. an initramfs
hook or a custom kernel cmdline), the same information can be supplied without
command-line flags by setting these parameters on the kernel command line
(`/proc/cmdline`):

| Parameter                | Meaning                                                          |
| ------------------------ | ---------------------------------------------------------------- |
| `jumpconfig=<url>`       | The configuration URL/path to use when no positional argument is given. |
| `jumpunattended=<url>`   | Enables unattended mode and provides the answers file URL/path. If set, `--unattended` is implied automatically (no flag needed). |

Resolution order:

- **Config:** positional `<config-url-or-path>` takes precedence; otherwise
  `jumpconfig=` is used; otherwise a "no configuration found" dialog is shown
  and `jump` exits.
- **Unattended mode:** the `--unattended` flag enables it; otherwise, if
  `jumpunattended=` is present on the kernel command line, unattended mode is
  enabled automatically and the answers file is taken from that value. The
  `--unattended` flag's optional argument always wins over the kernel
  parameter.

### Examples

A kernel command line for a fully unattended jumpstart:

```
jumpconfig=http://boot.example.com/jumpstart/webhost.cfg jumpunattended=http://boot.example.com/jumpstart/webhost.answers.cfg
```

With those parameters present, the boot script can simply run:

```sh
jump
```

and `jump` will fetch the config, fetch the answers, run unattended, and reboot
when done (per the `[done]` section).

## Exit behavior

- `--validate` exits `0` on a valid configuration (and answers, if
  `--unattended`), `1` on any validation error.
- In run mode, a fatal error calls the configured `[done] failaction` (default
  `wait`, which drops to a shell) after logging the failure and, in unattended
  mode, invoking the optional `feedback` script. A successful run performs the
  `[done] action`.
- Unknown flags or a missing configuration both print the usage message to
  stderr and exit non-zero.