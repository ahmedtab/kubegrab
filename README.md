# kubegrab

A single-file Bash tool that grabs a kubeconfig — from a local file or
**directly from a pipe** — and merges it into your local `~/.kube/config`,
embedding all certificates so the result is fully portable.

No external tools beyond `kubectl`, `base64`, `gzip`, and `mktemp` are required.

```sh
ssh user@host -- cat .kube/config | kubegrab
```

---

## Why this exists

The standard workflow for adding a remote cluster is tedious:

```sh
# old way — temp files, manual merges, easy to mess up
scp user@host:.kube/config /tmp/remote.yaml
KUBECONFIG=~/.kube/config:/tmp/remote.yaml kubectl config view --flatten > /tmp/merged.yaml
mv /tmp/merged.yaml ~/.kube/config
rm /tmp/remote.yaml
```

With kubegrab it becomes one line:

```sh
ssh user@host -- cat .kube/config | kubegrab
```

---

## Features

| Feature | Details |
|---|---|
| **Pipe support** | `cat config.yaml \| kubegrab` — stdin is detected automatically |
| **Auto-backup** | `~/.kube/config.<timestamp>.bak.gz` is written before every change |
| **Multi-cluster source** | If the source has multiple clusters, kubegrab lists them and lets you pick |
| **Duplicate detection** | Warns and asks for confirmation before overwriting an existing context |
| **Name-based cert extraction** | Reads certs by cluster/user name, not fragile array index |
| **`--remove` subcommand** | Cleanly deletes a context, its cluster entry, and its user entry |
| **`--switch` flag** | Auto-runs `kubectl config use-context` after a successful import |
| **Interactive prompts via `/dev/tty`** | Prompts always reach you even when stdin is a pipe |
| **Zero extra dependencies** | Only `kubectl`, `base64`, `gzip`, `mktemp` |

---

## Installation

```sh
curl -o ~/.local/bin/kubegrab \
  https://raw.githubusercontent.com/ahmedtab/kubegrab/main/kubegrab
chmod +x ~/.local/bin/kubegrab
```

Or clone and symlink:

```sh
git clone https://github.com/ahmedtab/kubegrab.git
ln -s "$PWD/kubegrab/kubegrab" ~/.local/bin/kubegrab
```

Make sure `~/.local/bin` is on your `PATH`.

---

## Usage

```
kubegrab [FLAGS] [SOURCE] [CLUSTER_NAME] [CLUSTER_USER] [SERVER_URL] [INSECURE]
cat ./config.yaml | kubegrab [FLAGS] [-] [CLUSTER_NAME] [CLUSTER_USER] [SERVER_URL] [INSECURE]
kubegrab --remove <CONTEXT_NAME>
```

### Flags

| Flag | Description |
|---|---|
| `-h`, `--help` | Print help and exit |
| `--switch` | Auto-switch to the imported context after import |
| `--remove <name>` | Remove a context and its cluster + user from `~/.kube/config` (backs up first) |

### Positional arguments

All are optional — any value not supplied as an argument is prompted for
interactively.

| Position | Description | Default |
|---|---|---|
| `SOURCE` | Source kubeconfig path, or `-` to force stdin | prompted |
| `CLUSTER_NAME` | Name for the cluster/context entry | random UUID |
| `CLUSTER_USER` | Name for the credentials entry | `<CLUSTER_NAME>-user` |
| `SERVER_URL` | Override the API server URL | kept from source |
| `INSECURE` | `true`/`false` — skip TLS verification | `false` |

---

## Examples

### Remote import over SSH — the primary use case

```sh
ssh user@192.168.10.11 -- cat .kube/config | kubegrab
```

You are prompted for the cluster name, user, etc. interactively (via `/dev/tty`)
while the config streams through stdin.

### SSH import with name and auto-switch

```sh
ssh user@192.168.10.11 -- cat .kube/config | kubegrab --switch - prod
```

### Import from a local file

```sh
kubegrab ./config.yaml
```

### Fully non-interactive

```sh
kubegrab ./config.yaml prod prod-user https://10.0.0.5:6443 false
```

### Import and immediately switch context

```sh
kubegrab --switch ./config.yaml staging
```

### Remove a context

```sh
kubegrab --remove staging
# Backup created: /home/user/.kube/config.20260702143200.bak.gz
# Deleted context staging.
# Deleted cluster staging.
# Deleted user staging-user.
# Removed context 'staging' (cluster: staging, user: staging-user)
```

### Multi-cluster source

When the source kubeconfig has more than one cluster entry, kubegrab lists
them and asks which one to import:

```
Source kubeconfig contains 3 clusters:
  [1] cluster-a
  [2] cluster-b
  [3] cluster-c
Select cluster [1]: 2
Cluster name [random-uuid]: cluster-b-prod
...
```

---

## Backup behaviour

Before **any** modification to `~/.kube/config`, a compressed copy is created:

```
~/.kube/config.20260702143200.bak.gz
```

Permissions are set to `600`. To restore:

```sh
gunzip -c ~/.kube/config.20260702143200.bak.gz > ~/.kube/config
```

---

## Requirements

- Bash 4.0+
- `kubectl`
- `base64` (GNU coreutils or macOS built-in)
- `gzip`
- `mktemp`

---

## License

MIT — see [LICENSE](LICENSE).
