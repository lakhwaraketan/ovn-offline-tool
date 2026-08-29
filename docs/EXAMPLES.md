# Sample commands

> **Prerequisite — install the OVN/OVS packages first.** `ovn-analyzer` wraps the
> standard OVN client tools; it does not bundle them. Install
> `openvswitch ovn ovn-central` (RPM) or `openvswitch-common ovn-common ovn-central`
> (deb) before running any command below, or the tool exits with code `127`
> listing the missing binaries. See the [Requirements](../README.md#requirements)
> section for the full list.

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

### Sample output — cross-node TCP trace

> Hostnames below are masked to the `example.com` domain. The `10.128.0.0/14`
> (pod network) and `100.88.0.0/16` (IC transit) addresses are OVN-Kubernetes'
> default, non-routable internal ranges — they identify nothing external.

```console
$ ./ovn-analyzer --nb node_ovnnb_db.db --sb node_ovnsb_db.db \
    trace --friendly-names --ct new \
    'inport=="openshift-network-diagnostics_network-check-target-7s8dz" &&
     eth.src==0a:58:0a:80:00:03 && eth.dst==0a:58:0a:80:00:01 &&
     ip4.src==10.128.0.3 && ip4.dst==10.129.0.3 && ip.ttl==64 &&
     tcp && tcp.src==40000 && tcp.dst==8080'
[ovn-analyzer] IC pairing OK: node master2.example.com
# tcp,reg14=0x2,vlan_tci=0x0000,dl_src=0a:58:0a:80:00:03,dl_dst=0a:58:0a:80:00:01,nw_src=10.128.0.3,nw_dst=10.129.0.3,nw_tos=0,nw_ecn=0,nw_ttl=64,nw_frag=no,tp_src=40000,tp_dst=8080,tcp_flags=0

ingress(dp="master2.example.com", inport="openshift-network-diagnostics_network-check-target-7s8dz")
---------------------------------------------------------------------------------------------------
 0. ls_in_check_port_sec (northd.c:9488): 1, priority 50, uuid ac3fadb7
    reg0[15] = check_in_port_sec();
    next;
 4. ls_in_pre_acl (northd.c:6129): ip, priority 100, uuid 150fd16a
    reg0[0] = 1;
    next;
 5. ls_in_pre_lb (northd.c:6339): ip, priority 100, uuid 3b3079d3
    reg0[2] = 1;
    next;
 6. ls_in_pre_stateful (northd.c:6369): reg0[2] == 1, priority 110, uuid 45db0806
    ct_lb_mark;

ct_lb_mark
----------
 7. ls_in_acl_hint (northd.c:6438): ct.new && !ct.est, priority 7, uuid f74fda7f
    reg0[7] = 1;
    reg0[9] = 1;
    reg0[1] = 1;
    next;
 8. ls_in_acl_eval (northd.c:7569): ip && !ct.est, priority 1, uuid df83e58f
    next;
10. ls_in_acl_action (northd.c:7386): 1, priority 0, uuid 0d0adf58
    reg8[16] = 0;
    reg8[17] = 0;
    reg8[18] = 0;
    next;
15. ls_in_pre_hairpin (northd.c:8439): ip && ct.trk, priority 100, uuid fc066757
    reg0[6] = chk_lb_hairpin();
    reg0[12] = chk_lb_hairpin_reply();
    next;
20. ls_in_acl_after_lb_action (northd.c:7397): reg8[30..31] == 0, priority 500, uuid 545613a4
    reg8[30..31] = 1;
    next(18);
20. ls_in_acl_after_lb_action (northd.c:7397): reg8[30..31] == 1, priority 500, uuid 3748d27a
    reg8[30..31] = 2;
    next(18);
20. ls_in_acl_after_lb_action (northd.c:7386): 1, priority 0, uuid 01ce5103
    reg8[16] = 0;
    reg8[17] = 0;
    reg8[18] = 0;
    reg8[30..31] = 0;
    next;
21. ls_in_stateful (northd.c:8381): reg0[1] == 1 && reg0[13] == 0, priority 100, uuid a97de1d0
    ct_commit { ct_mark.blocked = 0; ct_mark.allow_established = reg0[20]; ct_label.acl_id = reg2[16..31]; };
    next;
28. ls_in_l2_lkup (northd.c:10411): eth.dst == { 0a:58:a9:fe:01:01, 0a:58:0a:80:00:01 } && is_chassis_resident("cr-stor-master2.example.com"), priority 50, uuid b7fa5c54
    outport = "stor-master2.example.com";
    output;

egress(dp="master2.example.com", inport="openshift-network-diagnostics_network-check-target-7s8dz", outport="stor-master2.example.com")
-------------------------------------------------------------------------------------------------------------------------------------
 2. ls_out_pre_acl (northd.c:5973): ip && outport == "stor-master2.example.com", priority 110, uuid eaec1999
    next;
 3. ls_out_pre_lb (northd.c:5973): ip && outport == "stor-master2.example.com", priority 110, uuid a6ba17d4
    next;
 5. ls_out_acl_hint (northd.c:6438): ct.new && !ct.est, priority 7, uuid 66136597
    reg0[7] = 1;
    reg0[9] = 1;
    reg0[1] = 1;
    next;
 6. ls_out_acl_eval (northd.c:7571): ip && !ct.est, priority 1, uuid 24095ed7
    next;
 8. ls_out_acl_action (northd.c:7397): reg8[30..31] == 0, priority 500, uuid ca3ed72a
    reg8[30..31] = 1;
    next(6);
 6. ls_out_acl_eval (northd.c:7571): ip && !ct.est, priority 1, uuid 24095ed7
    next;
 8. ls_out_acl_action (northd.c:7397): reg8[30..31] == 1, priority 500, uuid 6fd60df1
    reg8[30..31] = 2;
    next(6);
 6. ls_out_acl_eval (northd.c:7571): ip && !ct.est, priority 1, uuid 24095ed7
    next;
 8. ls_out_acl_action (northd.c:7386): 1, priority 0, uuid 97abc0bc
    reg8[16] = 0;
    reg8[17] = 0;
    reg8[18] = 0;
    reg8[30..31] = 0;
    next;
10. ls_out_stateful (northd.c:8386): reg0[1] == 1 && reg0[13] == 0, priority 100, uuid 90933a46
    ct_commit { ct_mark.blocked = 0; ct_mark.allow_established = reg0[20]; ct_label.acl_id = reg2[16..31]; };
    next;
11. ls_out_check_port_sec (northd.c:5933): 1, priority 0, uuid 680cf2b9
    reg0[15] = check_out_port_sec();
    next;
12. ls_out_apply_port_sec (northd.c:5941): 1, priority 0, uuid 65928183
    output;
    /* output to "stor-master2.example.com", type "patch" */

ingress(dp="ovn_cluster_router", inport="rtos-master2.example.com")
-------------------------------------------------------------------
 0. lr_in_admission (northd.c:13591): eth.dst == { 0a:58:a9:fe:01:01, 0a:58:0a:80:00:01 } && inport == "rtos-master2.example.com" && is_chassis_resident("cr-rtos-master2.example.com"), priority 50, uuid df178b96
    reg9[1] = check_pkt_larger(1414);
    xreg0[0..47] = 0a:58:0a:80:00:01;
    next;
 1. lr_in_lookup_neighbor (northd.c:13787): 1, priority 0, uuid e45869a1
    reg9[2] = 1;
    next;
 2. lr_in_learn_neighbor (northd.c:13797): reg9[2] == 1, priority 100, uuid 9b2a37fc
    mac_cache_use;
    next;
14. lr_in_ip_routing_pre (northd.c:14054): 1, priority 0, uuid 9cf74f7a
    reg7 = 0;
    next;
15. lr_in_ip_routing (northd.c:11968): reg7 == 0 && ip4.dst == 10.129.0.0/23, priority 188, uuid 54590787
    ip.ttl--;
    reg8[0..15] = 0;
    reg0 = 100.88.0.2;
    reg5 = 100.88.0.3;
    eth.src = 0a:58:64:58:00:03;
    outport = "rtots-master2.example.com";
    flags.loopback = 1;
    reg9[9] = 1;
    next;
16. lr_in_ip_routing_ecmp (northd.c:14067): reg8[0..15] == 0, priority 150, uuid bc697947
    next;
17. lr_in_policy (northd.c:10936): ip4.src == 10.128.0.0/14 && ip4.dst == 10.128.0.0/14, priority 102, uuid 1e7c8119
    reg8[0..15] = 0;
    next;
18. lr_in_policy_ecmp (northd.c:14447): reg8[0..15] == 0, priority 150, uuid 36ba6310
    next;
21. lr_in_arp_resolve (northd.c:14877): outport == "rtots-master2.example.com" && reg0 == 100.88.0.2, priority 100, uuid adfebd29
    eth.dst = 0a:58:64:58:00:02;
    next;
25. lr_in_network_id (northd.c:15451): outport == "rtots-master2.example.com" && ip4 && reg0 == 100.88.0.3/16, priority 110, uuid 8371d3ca
    flags.network_id = 0;
    next;
27. lr_in_ecmp_stateful_egr (northd.c:15412): 1, priority 0, uuid a9971fac
    output;

egress(dp="ovn_cluster_router", inport="rtos-master2.example.com", outport="rtots-master2.example.com")
------------------------------------------------------------------------------------------------------
 0. lr_out_chk_dnat_local (northd.c:17087): 1, priority 0, uuid d5a205a2
    reg9[4] = 0;
    next;
 6. lr_out_delivery (northd.c:15547): outport == "rtots-master2.example.com", priority 100, uuid a300bbb3
    output;
    /* output to "rtots-master2.example.com", type "patch" */

ingress(dp="transit_switch", inport="tstor-master2.example.com")
----------------------------------------------------------------
 0. ls_in_check_port_sec (northd.c:5850): inport == "tstor-master2.example.com", priority 70, uuid 4f1a750e
    reg0[18] = 1;
    next;
 5. ls_in_pre_lb (northd.c:5970): ip && inport == "tstor-master2.example.com", priority 110, uuid 28d18ea8
    next;
28. ls_in_l2_lkup (northd.c:10423): eth.dst == 0a:58:64:58:00:02, priority 50, uuid 9aed7984
    outport = "tstor-master1.example.com";
    output;

egress(dp="transit_switch", inport="tstor-master2.example.com", outport="tstor-master1.example.com")
---------------------------------------------------------------------------------------------------
11. ls_out_check_port_sec (northd.c:5933): 1, priority 0, uuid 680cf2b9
    reg0[15] = check_out_port_sec();
    next;
12. ls_out_apply_port_sec (northd.c:5941): 1, priority 0, uuid 65928183
    output;
    /* output to "tstor-master1.example.com", type "remote" */
```

The trace crosses the node boundary: it enters `master2`'s logical switch,
routes through `ovn_cluster_router`, and leaves via the `transit_switch` toward
`master1` (`type "remote"`). To follow the far side, re-run the trace against
`master1`'s NB/SB DB pair.

> The startup `ovsdb_idl|WARN ... lacks <column> (database needs upgrade?)` lines
> are harmless: they appear when the host OVN client is a different version than
> the captured DB schema. Match the host OVN version to the cluster's to silence
> them.

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
