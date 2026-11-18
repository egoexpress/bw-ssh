# SSH key passphrases for Bitwarden

## About this fork

This is a fork of [Glandos/bw-ssh](https://framagit.org/Glandos/bw-ssh) on
Framagit. GitHub cannot fork a GitLab project directly, so this repository was
imported with its complete history and `upstream` points at the original:

```sh
git remote add upstream https://framagit.org/Glandos/bw-ssh.git
```

The sections below the upstream README describe what this fork changes, how to
install it, and how to apply the changes back to upstream.

## Description
This will help you having private keys with passphrase stored in your Bitwarden vault.

This does **not** store and extract private keys. Private keys are, well, private, and should be generated uniquely on each machine. But the passphrase can be stored in Bitwarden.

There are 2 binaries (in fact, the 2 are combined):
1. An `ssh-askpass` compatible implementation to ask for **passphrase** stored into your Bitwarden vault.
2. An `ssh-keygen` wrapper that will generate a passphrase, store it in the vault, and generate the key with it.

## Requirements
- Python 3, any version
- [amoffat/sh](https://github.com/amoffat/sh/) (both 1.x and 2.x work, see
  [Changes in this fork](#changes-in-this-fork))
- [rbw](https://github.com/doy/rbw). It's an implementation of Bitwarden in CLI, supporting both terminal and graphical master password dialog. It's way faster than the official implementation.

## Usage
### keygen
- `bw-keygen -t ed25519 -f /home/user/.ssh/test_ed25519`

### askpass
- `SSH_ASKPASS_REQUIRE=prefer SSH_ASKPASS="bw-askpass" ssh-add /home/user/.ssh/test_ed25519`
- `SSH_ASKPASS_REQUIRE=prefer SSH_ASKPASS="bw-askpass" ssh -i /home/user/.ssh/test_ed25519 user@hostname`

You can add environment variables in your shell profile.

## Changes in this fork

The upstream tool assumes that every prompt it receives is a key passphrase
prompt. That breaks as soon as ssh asks for something else, which happens with
`SSH_ASKPASS_REQUIRE=prefer`: ssh then routes **all** prompts through the
askpass helper, including the host key confirmation for an unknown host,
password authentication and keyboard-interactive prompts. On first contact
with a new host this ended in

```text
AttributeError: 'NoneType' object has no attribute 'group'
Host key verification failed.
```

because the passphrase regex found no key path in
`Are you sure you want to continue connecting (yes/no/[fingerprint])?`.

| Commit | Change |
| --- | --- |
| `fix(askpass): answer non-passphrase prompts on the terminal` | Only real passphrase prompts are looked up in the vault. Host key confirmation, passwords and keyboard-interactive prompts are asked on `/dev/tty`, with echo disabled for secrets. Without a terminal the prompt is printed and the helper exits 1 instead of raising. |
| `fix(askpass): strip quotes from ssh-keygen passphrase prompts` | `ssh-keygen`, which git uses for SSH commit signing, quotes the key path with `"` instead of `'`. The quotes ended up in the vault lookup, so the entry was not found and signing failed. |
| `fix(askpass): exit when the vault has no passphrase entry` | An empty `rbw get` result is reported as a missing vault entry instead of being returned as an empty passphrase. |
| `fix(keygen): read rbw history result with sh 2.x` | `sh` 2.0 returns a plain `str` unless `_return_cmd` is set. Reading `exit_code` and `stderr` raised `AttributeError`, so `bw-keygen` could not detect an existing vault entry. |

The changes are also available as patches against upstream `main` in
[`patches/`](patches), one file per commit.

## Installation

Copy the script to a directory in `PATH` and create the `bw-keygen` symlink:

```sh
install -m 755 bw-askpass ~/.local/bin/bw-askpass
ln -sf bw-askpass ~/.local/bin/bw-keygen
```

Then point ssh at it, for example in `~/.config/fish/config.fish`:

```fish
set -gx SSH_ASKPASS bw-askpass
set -gx SSH_ASKPASS_REQUIRE prefer
```

## Prompt handling

The helper decides by the prompt text what to do:

- `Enter passphrase for key '<path>':` and
  `Bad passphrase, try again for '<path>':` from ssh and ssh-add, and the
  double-quoted `Enter passphrase for "<path>":` from ssh-keygen (used by git
  for SSH commit signing), are looked up in the Bitwarden vault, using the
  key's file name inside the `ssh` folder (`rbw get --folder ssh <name>`). The
  passphrase is verified with `ssh-keygen -y` before it is returned, so a
  wrong entry cannot make `ssh-add` loop.
- Host key confirmation is shown verbatim on the terminal, so the fingerprint
  can be compared before answering `yes`. Trusting a host on first sight is
  the same decision ssh asks for without an askpass helper.
- Password and keyboard-interactive prompts are read with echo disabled.
- If no terminal is available (cron jobs, git hooks without a TTY), the prompt
  is written to stderr and the helper exits 1.

## Applying the changes upstream

```sh
git clone https://framagit.org/Glandos/bw-ssh.git bw-ssh-upstream
cd bw-ssh-upstream
git am /path/to/bw-ssh/patches/*.patch
```
