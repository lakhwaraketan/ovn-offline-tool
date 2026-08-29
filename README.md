# ovn-offline-tool

`ovn-analyzer` — a single-file, host-native CLI for **offline** inspection and
post-mortem analysis of OVN (Open Virtual Network) databases extracted from an
OpenShift / Kubernetes cluster.

It serves extracted `ovnnb_db.db` / `ovnsb_db.db` files locally through
transient `ovsdb-server` processes and wraps the standard OVN clients
(`ovn-nbctl`, `ovn-sbctl`, `ovn-trace`, `ovs-appctl`) against them — with **no
live cluster, no container runtime, and no network daemons** required.

Built for OVN-Kubernetes **Interconnect (IC)** clusters (OCP 4.14+), where each
`ovnkube-node` pod has its own per-node Northbound and Southbound database.

> ## ⚠️ Prerequisite — OVN/OVS packages are REQUIRED
>
> `ovn-analyzer` is a wrapper: it does **not** bundle any OVN binaries. Before
> using it you **must install** the host packages that provide `ovsdb-server`,
> `ovsdb-tool`, `ovs-appctl`, `ovn-nbctl`, `ovn-sbctl`, and `ovn-trace`:
>
> ```bash
> # RHEL / Fedora / CentOS Stream
> sudo dnf install -y openvswitch ovn ovn-central
>
> # Debian / Ubuntu
> sudo apt-get install -y openvswitch-common ovn-common ovn-central
> ```
>
> Without these packages the tool cannot run — it will exit with an error
> listing the missing binaries. See [Requirements](#requirements) for details.

## Features

- **`nbctl` / `sbctl`** — run any `ovn-nbctl` / `ovn-sbctl` command against the
  offline databases (`show`, `list`, `find`, `lflow-list`, …).
- **`trace`** — offline `ovn-trace` packet simulation against the local SB state.
- **`appctl`** — `ovs-appctl` against a transient server's control socket.
- **`interactive`** — a subshell with `OVN_NB_DB` / `OVN_SB_DB` (and the control
  sockets) pre-exported for multi-command sessions.
- **`check`** — IC pairing checklist: verifies both files have the right schema
  and belong to the **same** `ovnkube-node` (catches accidentally mixing an NB
  and SB from two different pods).
- **`dump`** — snapshot a node's NB/SB into normalized, UUID-scrubbed, sorted
  files for stable diffing.
- **`compare`** — offline side-by-side diff of two dumps that flags drift in the
  cluster-synced (invariant) tables separately from expected per-node deltas.
- **Transient & self-cleaning** — spins up isolated `ovsdb-server` instances in a
  temporary directory and guarantees teardown on exit, Ctrl-C, or pipe close.
- **Non-destructive** — always serves copies; the extracted evidence is never
  modified. Clustered/Raft databases are auto-converted to standalone for
  serving.

## Requirements

**Package installation is required.** `ovn-analyzer` shells out to the standard
OVN/OVS command-line tools and will not work unless they are installed on the
host first. It runs a preflight check and exits with code `127` if any of these
binaries are missing:

| Binary          | Provided by (RPM)   | Provided by (deb)         |
|-----------------|---------------------|---------------------------|
| `ovsdb-server`  | `openvswitch`       | `openvswitch-common`      |
| `ovsdb-tool`    | `openvswitch`       | `openvswitch-common`      |
| `ovs-appctl`    | `openvswitch`       | `openvswitch-common`      |
| `ovn-nbctl`     | `ovn` / `ovn-central` | `ovn-common` / `ovn-central` |
| `ovn-sbctl`     | `ovn` / `ovn-central` | `ovn-common` / `ovn-central` |
| `ovn-trace`     | `ovn` / `ovn-central` | `ovn-common` / `ovn-central` |

Install them before running the tool:

```bash
# RHEL / Fedora / CentOS Stream
sudo dnf install -y openvswitch ovn ovn-central

# Debian / Ubuntu
sudo apt-get install -y openvswitch-common ovn-common ovn-central
```

Verify they are on `PATH`:

```bash
for b in ovsdb-server ovsdb-tool ovs-appctl ovn-nbctl ovn-sbctl ovn-trace; do
  command -v "$b" || echo "MISSING: $b"
done
```

> Match the host OVN version to the cluster's OVN version where possible. A host
> client older than the captured DB schema may warn about missing columns
> (e.g. `lacks nb_uuid column`) — harmless, but a matching version silences it.

## Install

```bash
git clone git@github.com:lakhwaraketan/ovn-offline-tool.git
cd ovn-offline-tool
chmod +x ovn-analyzer

# optional: put it on PATH and install the man page
sudo install -m 0755 ovn-analyzer /usr/local/bin/ovn-analyzer
sudo install -m 0644 man/ovn-analyzer.1 /usr/local/share/man/man1/ovn-analyzer.1
man ovn-analyzer
```

## Usage

```
ovn-analyzer --nb <ovnnb_db.db> --sb <ovnsb_db.db> [options] <command> [args...]
```

See [`docs/EXAMPLES.md`](docs/EXAMPLES.md) for a full set of sample commands, and
the [man page](man/ovn-analyzer.1) for complete reference.

### Quick examples

```bash
# Verify the NB/SB pair belongs to the same node before anything else
./ovn-analyzer --nb node_ovnnb_db.db --sb node_ovnsb_db.db check

# Northbound / Southbound inspection
./ovn-analyzer --nb node_ovnnb_db.db --sb node_ovnsb_db.db nbctl show
./ovn-analyzer --nb node_ovnnb_db.db --sb node_ovnsb_db.db sbctl lflow-list

# Offline packet trace
./ovn-analyzer --nb node_ovnnb_db.db --sb node_ovnsb_db.db \
  trace --detailed <switch> 'inport=="<lsp>" && eth.dst==ff:ff:ff:ff:ff:ff'

# Interactive session
./ovn-analyzer --nb node_ovnnb_db.db --sb node_ovnsb_db.db interactive

# Per-node IC diffing
./ovn-analyzer --nb nodeA_ovnnb_db.db --sb nodeA_ovnsb_db.db dump ./dumpA
./ovn-analyzer --nb nodeB_ovnnb_db.db --sb nodeB_ovnsb_db.db dump ./dumpB
./ovn-analyzer compare ./dumpA ./dumpB
```

## How it works

1. Copies each supplied DB into a private `/tmp/ovn-analyzer.XXXX` directory.
2. If a DB is clustered (Raft), converts it to standalone with
   `ovsdb-tool cluster-to-standalone`.
3. Starts one detached `ovsdb-server` per DB on a unique Unix domain socket.
4. Exports `OVN_NB_DB` / `OVN_SB_DB` (and, for `appctl`/`interactive`, the
   `OVN_NB_CTL` / `OVN_SB_CTL` control sockets).
5. Runs the requested client command against those sockets.
6. On exit / `SIGINT` / `SIGTERM` / `SIGHUP` / `SIGPIPE`, terminates every
   `ovsdb-server` and removes the temporary directory.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
