# Template reference: Alcatel Stellar AP by SNMP

Full item/trigger/macro catalogue for `template_alcatel_stellar_ap_snmp.yaml`. See
[README.md](README.md) for setup steps, the interface filter, and known limitations.

Depends on two stock templates, linked automatically on import:
- `Generic by SNMP`
- `Interfaces by SNMP`

## 1. Collected items

### 1.1 Defined in the template (per AP)

| Key | Name | Type | Interval | Units |
|---|---|---|---|---|
| `ap.firmware` | AP: Firmware version | SNMP (sysDescr) | 1h | — |
| `ap.model` | AP: Model | SNMP (sysDescr) | 1h | — |
| `ap.cpu.util` | AP: CPU utilization | SNMP (100 − ssCpuIdle) | 1m | % |
| `ap.cpu.user` | AP: CPU user time | SNMP | 1m | % |
| `ap.cpu.system` | AP: CPU system time | SNMP | 1m | % |
| `ap.cpu.load1` / `load5` / `load15` | Load average 1/5/15 m | SNMP | 1m | — |
| `ap.mem.total` | AP: Memory total | SNMP | 5m | B |
| `ap.mem.avail` | AP: Memory available | SNMP | 5m | B |
| `ap.mem.cached` | AP: Memory cached | SNMP | 5m | B |
| `ap.mem.buffers` | AP: Memory buffers | SNMP | 5m | B |
| `ap.mem.pused` | AP: Memory utilization | Calculated | 5m | % |

`ap.mem.pused` nets out cache and buffers (memory actually in use). AP1221 units have
~232 MB of RAM; AP1101 units ~120 MB.

### 1.2 Storage (LLD: `ap.storage.discovery`)

Discovers filesystems from `hrStorageTable`, filtered to **fixed disks** and excluding
`/dev`, `/sys`, `/run`, `/proc` and `*/shm`. Per filesystem:

| Key | Name | Type | Units |
|---|---|---|---|
| `ap.fs.total[{#SNMPINDEX}]` | Total space | SNMP | B |
| `ap.fs.used[{#SNMPINDEX}]` | Used space | SNMP | B |
| `ap.fs.pused[{#SNMPINDEX}]` | Space utilization | Calculated | % |

### 1.3 Interfaces (`Interfaces by SNMP` module)

See the interface filter section in the main README. For each **surviving** interface:

| Key | What it measures |
|---|---|
| `net.if.in[ifHCInOctets.*]` / `net.if.out[ifHCOutOctets.*]` | Inbound / outbound traffic (bps) |
| `net.if.in.errors[*]` / `net.if.out.errors[*]` | Error packets |
| `net.if.in.discards[*]` / `net.if.out.discards[*]` | Discarded packets |
| `net.if.speed[ifHighSpeed.*]` | Negotiated speed (Mbps) |
| `net.if.status[ifOperStatus.*]` | Operational status |
| `net.if.type[*]` / `net.if.name[*]` / `net.if.alias[*]` | Type, name and alias |

### 1.4 Provided by `Generic by SNMP`

System uptime, SNMP availability, ICMP ping and their triggers. Not duplicated here.

## 2. Triggers

### 2.1 Defined in the template

| Alert | Severity | Condition | Default threshold |
|---|---|---|---|
| AP: Firmware differs from expected | Info | `ap.firmware` ≠ expected | `{$AP.FW.EXPECTED}` = `4.0.7` |
| AP: High CPU usage | Warning | CPU > threshold, sustained 15 min | `{$AP.CPU.UTIL.WARN}` = `85%` |
| AP: High memory usage | Warning | Memory > threshold, sustained 10 min | `{$AP.MEM.UTIL.WARN}` = `90%` |
| {#FSNAME}: High space usage | Warning | FS > threshold, sustained 10 min | `{$AP.FS.PUSED.WARN}` = `90%` |

### 2.2 From the `Interfaces by SNMP` module (per surviving interface)

| Alert | Severity | Condition |
|---|---|---|
| Link down | High | `ifOperStatus` ≠ up |
| High bandwidth utilization | Warning | > `{$IF.UTIL.MAX}` (90) |
| High error rate | Warning | in/out errors > `{$IF.ERRORS.WARN}` (2) over 5 min |

## 3. Macros

| Macro | Default | Purpose |
|---|---|---|
| `{$AP.FW.EXPECTED}` | `4.0.7` | Expected firmware version (drift detection) |
| `{$AP.CPU.UTIL.WARN}` | `85` | CPU threshold (%) |
| `{$AP.MEM.UTIL.WARN}` | `90` | Memory threshold (%) |
| `{$AP.FS.PUSED.WARN}` | `90` | Filesystem threshold (%) |
| `{$AP.FS.NAME.NOT_MATCHES}` | `^(/dev\|/sys\|/run\|/proc\|.+/shm$)` | Excluded filesystems |
| `{$NET.IF.IFNAME.MATCHES}` | `^(eth[01]\|wifi[01]\|br-wan)$` | Discovered interfaces |
| `{$NET.IF.IFNAME.NOT_MATCHES}` | `^$` | Neutralized (filtering is done via MATCHES) |
| `{$SNMP_COMMUNITY}` | — | SNMP community (set at host or global level) |
| `{$IF.UTIL.MAX}`, `{$IF.ERRORS.WARN}`, `{$IFCONTROL}` | from the module | Interface-module thresholds |
