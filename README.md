# lvm

A wrapper around [Lima](https://github.com/lima-vm/lima) for running isolated Linux VMs scoped to specific directories. Designed for running Claude Code in total isolation — no access to your home directory, SSH keys, or anything outside the project folder.

## How it works

`cd` into a project directory, create a VM with a name you choose, then use that name from anywhere.

```
cd ~/Projects/my-app
lvm create myapp                  # creates a VM, mounts only this directory
lvm claude myapp                  # runs Claude Code inside the VM
lvm shell myapp                   # drops into a shell inside the VM
lvm stop myapp                    # stops the VM
lvm start myapp                   # starts it again
lvm port-forward myapp 3000       # expose host:3000 -> VM:3000 (permanent)
lvm provision myapp               # re-run provision script after template changes
lvm rename myapp my-app-v2        # renames the VM
lvm rm myapp                      # deletes the VM
lvm list                          # lists all lvm instances
```

## Prerequisites

- macOS 13+
- [Lima](https://github.com/lima-vm/lima): `brew install lima`

## Install

Clone the repo and add it to your PATH in `~/.zshrc`:

```bash
export PATH="$HOME/AI/limavm:$PATH"
```

Then reload your shell:

```bash
source ~/.zshrc
```

### Shell completions

For tab-completion of commands and VM names in zsh, add to `~/.zshrc`:

```bash
eval "$(lvm completions)"
```

## VM defaults

Configured in `template.yaml`:

| Setting   | Default                                |
|-----------|----------------------------------------|
| OS        | Ubuntu 24.04                           |
| CPUs      | 4                                      |
| Memory    | 8 GiB                                  |
| Disk      | 20 GiB                                 |
| VM type   | vz (Apple Virtualization.framework)    |
| Mount     | virtiofs                               |
| Provision | Node.js 22, git, build-essential, Claude Code |

Edit `template.yaml` to change these.

## Isolation

- Only the specified directory is mounted — nothing else from the host is visible
- No access to `~/.ssh`, `~/.aws`, `~/.config`, or any host credentials
- No ports are forwarded to the host by default — opt in per-port with `lvm port-forward` when you need to reach a service running inside the VM (e.g. a dev server)
- Outbound networking is available (needed for Claude API and package managers)

## Reaching services running inside the VM

Some workflows need the host browser to talk to something running inside the VM (a Next.js dev server, a Jupyter kernel, etc.). Add a port forward once and it's kept across VM restarts:

```bash
lvm port-forward myapp 3000        # host:3000 -> VM:3000
lvm port-forward myapp 3000 4000   # host:4000 -> VM:3000 (when 3000 is taken on the host)
```

Restarts the VM once to apply. The VM-side service can bind to `127.0.0.1` (the default for most dev servers).

To remove a forward, edit the VM config: `limactl edit <name>` and delete the entry under `portForwards`.

## Running commands

`lvm shell` accepts extra arguments to run a one-off command instead of an interactive shell:

```bash
lvm shell myapp git status
lvm shell myapp npm test
```

`lvm claude` passes extra arguments through to Claude Code:

```bash
lvm claude myapp --model opus
```

## Updating existing VMs

If you edit `template.yaml` (e.g., add new packages), existing VMs don't pick up the changes automatically. Re-run the provision script on them:

```bash
lvm provision myapp
```

This runs the provision script from the current `template.yaml` inside the VM via `sudo`.

## Renaming

Renaming stops the VM first (Lima identifies instances by directory name), then restarts it under the new name:

```bash
lvm rename old-name new-name
lvm start new-name
```
