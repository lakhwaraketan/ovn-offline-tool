# Sample commands

All examples assume two extracted database files in the current directory:

```
node_ovnnb_db.db   # OVN Northbound, extracted from an ovnkube-node pod
node_ovnsb_db.db   # OVN Southbound, extracted from the SAME ovnkube-node pod
```

Set a shorthand:

```bash
OVN="./ovn-analyzer --nb node_ovnnb_db.db --sb node_ovnsb_db.db"
```

## 0. Verify the pair (IC pairing checklist)

```bash
$OVN check
```

Confirms both files have the correct schema and belong to the same
`ovnkube-node`. Exit `0` = valid pair, non-zero = mismatch (with details). Add
`--force` to any command to bypass a failed checklist.

## 1. Northbound (`nbctl`)

```bash
$OVN nbctl show                              # logical topology
$OVN nbctl ls-list                           # logical switches
$OVN nbctl lr-list                           # logical routers
$OVN nbctl list Logical_Switch               # full Logical_Switch rows
$OVN nbctl list Logical_Switch_Port
$OVN nbctl find ACL direction=to-lport       # filtered ACLs
$OVN nbctl list NAT                           # NAT rules
$OVN nbctl list Load_Balancer
```

## 2. Southbound (`sbctl`)

```bash
$OVN sbctl show                              # chassis + port bindings
$OVN sbctl lflow-list                        # compiled logical flows
$OVN sbctl list Chassis                      # all known chassis
$OVN sbctl list Datapath_Binding
$OVN sbctl list Port_Binding
```

## 3. Packet trace (`trace`)

```bash
# Discover a switch and one of its ports first
$OVN nbctl ls-list
$OVN nbctl show <switch-name>

# Broadcast trace on that switch
$OVN trace --detailed <switch-name> \
    'inport=="<lsp-name>" && eth.dst==ff:ff:ff:ff:ff:ff'

# Unicast IP trace
$OVN trace <switch-name> \
    'inport=="<lsp>" && eth.src==<src-mac> && eth.dst==<gw-mac> &&
     ip4.src==<src-ip> && ip4.dst==<dst-ip> && ip.ttl==64'
```

> In IC clusters a trace terminates at the `transit_switch` boundary for
> cross-node destinations. Re-run the trace against the **destination** node's
> DB pair to follow the far side.

## 4. Control socket (`appctl`)

```bash
$OVN appctl nb ovsdb-server/list-dbs
$OVN appctl sb memory/show
$OVN appctl nb ovsdb-server/get-db-storage-status OVN_Northbound
$OVN appctl nb list-commands
```

## 5. Interactive session

```bash
$OVN interactive
# inside the subshell OVN_NB_DB / OVN_SB_DB / OVN_NB_CTL / OVN_SB_CTL are set:
#   ovn-nbctl show
#   ovn-sbctl lflow-list
#   appctl-nb memory/show
#   exit          # tears down the transient servers
```

## 6. Per-node IC dump & compare

```bash
# snapshot two nodes
$OVN dump ./dumpA
./ovn-analyzer --nb other_ovnnb_db.db --sb other_ovnsb_db.db dump ./dumpB

# diff them: invariant-table drift is flagged, per-node deltas are expected
./ovn-analyzer compare ./dumpA ./dumpB
```

Cluster-wide sweep (dump every node pair, then compare each to a reference):

```bash
for nb in *_ovnnb_db.db; do
  sb="${nb/_ovnnb_/_ovnsb_}"
  ./ovn-analyzer --nb "$nb" --sb "$sb" dump "./dump-${nb%%_*}"
done

ref=$(ls -d ./dump-* | head -1)
for d in ./dump-*; do
  [ "$d" = "$ref" ] && continue
  echo "== $ref vs $d =="
  ./ovn-analyzer compare "$ref" "$d" | tail -1
done
```

## Options

```
--nb <path>     Northbound database file (required for live commands)
--sb <path>     Southbound database file (required for live commands)
--keep          Keep the /tmp runtime directory on exit (debugging)
--force         Run even if the IC pairing checklist fails
-v, --verbose   Verbose diagnostics on stderr
-h, --help      Help
```

## Exit codes

| Code | Meaning                                             |
|------|-----------------------------------------------------|
| 0    | success (or `check`/`compare` reported consistency) |
| 1    | general error / failed checklist                    |
| 2    | `compare` detected IC drift in invariant tables     |
| 127  | required OVN binary not found on the host           |
