# procd-lxcfs-fix

Local patch for OpenWrt [procd](https://git.openwrt.org/project/procd.git):
make `ubus call system info` report container-correct values under LXC/LXCFS.

## Problem

procd's `system_info()` reads data via `sysinfo(2)`, which returns **host-wide**
uptime, load, memory and swap even inside a container, while `/proc`
(virtualized by LXCFS) exposes the container's view. LuCI and other ubus
consumers therefore showed the host's RAM, uptime and swap inside the container.

## Fix

`system.c` reads from the standard `/proc` interfaces first and falls back to
`sysinfo(2)` per field when a source is unavailable:

| item   | source |
|--------|--------|
| uptime | `/proc/uptime` |
| load   | `/proc/loadavg` (1<<16 fixed-point scaling kept) |
| memory | `/proc/meminfo` (MemTotal, MemFree, Shmem, Buffers, MemAvailable, Cached) |
| swap   | `/proc/meminfo` -> `/proc/swaps` -> `sysinfo(2)` |

No container detection is performed — bare metal, VMs, LXC and LXC+LXCFS all
simply read their own `/proc`. The ubus JSON schema, field names, units and
load scaling are unchanged, so LuCI needs no modification.

## Base

- Upstream commit: `58eb263d` (openwrt-25.12 branch), file: `system.c` only
- Branch: `yt-procfs-fix`

## Build

Build the `procd` package with the matching OpenWrt SDK
(`openwrt-sdk-25.12.3-*`, target x86/64) and install the resulting package.
