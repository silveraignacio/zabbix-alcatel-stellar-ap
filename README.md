# Alcatel Stellar AP monitoring for Zabbix 7.2

A Zabbix template for monitoring Alcatel-Lucent Enterprise Stellar access points
(OAW-AP1101 / OAW-AP1221, AWOS 4.0.7.x) running in Express mode (PVM/SVM/Member
cluster) over SNMPv2c.

| | |
|---|---|
| Template | `Alcatel Stellar AP by SNMP` |
| Zabbix | 7.2 |
| Validated firmware | AWOS 4.0.7.x |
| MIBs used | MIB-II, HOST-RESOURCES-MIB, UCD-SNMP-MIB, IF-MIB |

Every OID in this template was checked against real hardware with `snmpwalk` before
being added — see [Confirmed limitations](#confirmed-limitations) below for what
doesn't work and why.

## Files

| File | What it is |
| --- | --- |
| `template_alcatel_stellar_ap_snmp.yaml` | The template, ready to import into Zabbix 7.2 |
| `TEMPLATE_REFERENCE.md` | Full item/trigger/macro catalogue |

## What it monitors

Links the stock templates `Generic by SNMP` and `Interfaces by SNMP`, and adds:

- **CPU**: aggregate utilization, user, system, load average 1/5/15.
- **Memory**: total, available, cached, buffers, and real usage % (netting out
  cache/buffers, otherwise the number is misleading).
- **Storage**: filesystem discovery with usage %.
- **Firmware and model**, parsed from `sysDescr`, with a drift trigger.

Uptime and SNMP availability are **not** defined here — `Generic by SNMP` already
provides `system.hw.uptime[hrSystemUptime.0]` and `zabbix[host,snmp,available]`,
each with its own trigger. Duplicating them fails the import on a key collision.

### Interface filter — important

Stellar APs expose **~40 interfaces each** (VAPs `ath0xx`, `br-vlanXX` bridges,
`eth0-XX` subinterfaces, tunnels). Without a filter, a handful of APs turns into
thousands of items.

The `{$NET.IF.IFNAME.MATCHES}` macro limits discovery to 4-5 interfaces:

```
^(eth[01]|wifi[01]|br-wan)$
```

| Interface | What it is |
|---|---|
| `eth0` / `eth1` | Wired uplinks |
| `wifi0` | 2.4 GHz radio |
| `wifi1` | 5 GHz radio |
| `br-wan` | Uplink bridge |

A host-level macro always wins over the template's and over any linked parent
template's, so when you create hosts, set `{$NET.IF.IFNAME.MATCHES}` explicitly at
host level too rather than relying on the template default — that way the filter
doesn't depend on macro precedence between templates. If you need a specific VAP,
add it to the macro **at host level**, not in the template.

### Alerts

| Alert | Severity | Condition | Default threshold |
|---|---|---|---|
| AP: Firmware differs from expected | Info | `ap.firmware` ≠ expected | `{$AP.FW.EXPECTED}` = `4.0.7` |
| AP: High CPU usage | Warning | CPU > threshold, sustained 15 min | `{$AP.CPU.UTIL.WARN}` = `85%` |
| AP: High memory usage | Warning | Memory > threshold, sustained 10 min | `{$AP.MEM.UTIL.WARN}` = `90%` |
| {#FSNAME}: High space usage | Warning | Filesystem > threshold, sustained 10 min | `{$AP.FS.PUSED.WARN}` = `90%` |
| Link down (per interface, from `Interfaces by SNMP`) | High | `ifOperStatus` ≠ up | — |
| High bandwidth utilization (per interface) | Warning | > `{$IF.UTIL.MAX}` | 90 |
| High error rate (per interface) | Warning | errors in/out > `{$IF.ERRORS.WARN}` for 5 min | 2 |

## Setup

1. **Make sure the stock Zabbix 7.2 templates this one links against already exist.**
   IMPORTANT: Zabbix upgrades do NOT reimport templates. An instance upgraded from 5.x
   may still carry the old module name "Template SNMP Interfaces" instead of
   "Interfaces by SNMP", which this template links by name. If it's missing, the
   import fails with "Template not found".
   Data collection -> Templates, look for both:
   `Generic by SNMP` | `Interfaces by SNMP`
   If either is missing, import it first from the official repo:
   `git.zabbix.com/projects/ZBX/repos/zabbix/raw/templates/module/interfaces_snmp/template_module_interfaces_snmp.yaml`
   (release/6.2 branch, or the one matching your Zabbix version).
   Do NOT also link "ICMP Ping": "Generic by SNMP" already provides the
   icmpping/icmppingloss/icmppingsec items and their triggers, and linking both fails
   the import on a key collision.

2. **Verify the OIDs respond on your real APs** (especially on lower-memory models),
   e.g. with `snmpwalk`/`snmpget` against `sysDescr`, `ssCpuIdle`, `memTotalReal`, etc.
   — the exact OIDs are in `template_alcatel_stellar_ap_snmp.yaml`
   ([item reference](TEMPLATE_REFERENCE.md)).

3. **Import the template.**
   Frontend: Data collection -> Templates -> Import -> `template_alcatel_stellar_ap_snmp.yaml`.
   Keep "Create new" and "Update existing" checked for all sections.

5. **Create the remaining hosts** (frontend, API, or your own automation), linking the
   template and setting `{$SNMP_COMMUNITY}` and `{$NET.IF.IFNAME.MATCHES}` at host level.

## Confirmed limitations

Everything below was verified with `snmpwalk` against real hardware — these are not
gaps in the template, they're the honest ceiling of what SNMP exposes on these APs.

1. **The ALE private branch (`enterprises.6486`) is unusable.** The agent echoes back
   the queried OID (`snmpgetnext .6486.801` returns `.6486.801` instead of a larger
   OID), and a plain `snmpget` on `.6486.802` returns "No Such Object". Zabbix can't
   read it with a normal GET.
2. **Cluster role (PVM/SVM/Member) and failover can't be detected via SNMP.** PVM and
   SVM are monitored as separate hosts, alerting on unavailability of each. For real
   failover detection, use the cluster's web UI or OmniVista.
3. **Associated client count isn't available.** The 802.11 MIB only exposes
   configuration/PHY groups, not an associated-stations table. For per-AP client
   counts, use the AP's web UI or OmniVista.
4. **Firmware drift is detected at the minor-version level, not patch level.**
   `sysDescr` returns `4.0.7`, without the `.14`.

## License

MIT — see [LICENSE](LICENSE).
