# nano-exporter2

A minimalistic exporter of node metrics for the Prometheus monitoring
system.

## Notice

This is a fork of [nano-exporter] (written by Heikki Kallasjoki <fis@zem.fi>),
with additional plugins written by Leah Neukirchen <leah@vuxu.org>.

[nano-exporter]: https://github.com/fis/nano-exporter

## Word of Warning

There are no particularly compelling reasons to use this software over
the alternatives. On a Real Computer™, the standard [Prometheus node
exporter](https://github.com/prometheus/node_exporter) is perfectly
adequate, and has a much larger feature set, not to mention adoption
rate. On OpenWRT, the standard community-supported package repository
already contains a lightweight Lua rewrite as
[prometheus-node-exporter-lua](https://openwrt.org/packages/pkgdata/prometheus-node-exporter-lua),
which is even smaller in disk footprint, if you don't count the Lua
interpreter.

The only reason why you would *need* this version is if you were
building for a particularly constrained system, and did not want to
include the Lua interpreter, or *any* foreign dependencies (other than
a C runtime library).

Other than that, you might choose to use it for æsthetic reasons.

## Installation

If you are able to compile code for your target system of choice, and
are happy to include all collectors, you may simply compile all the
`.c` files together.

You can also use the bundled `Makefile`, which likely requires GNU
Make. You can comment out collectors in the Makefile to leave them out
of the build if you don't need them.

The metrics collection is highly Linux-specific. You probably won't
have much luck on other operating systems.

## Collectors

For the most part, the produced metrics try to loosely (but not
slavishly) adhere to the format used by the standard Prometheus node
exporter (as of version v0.16).

This is an overview of the available collectors. See the [Collector
Reference](#collector-reference) below for detailed documentation,
including generated metrics, labels and configuration options.

| Name | Description |
| ---- | ----------- |
| [`conntrack`](#conntrack) | Connection tracking (NAT) statistics. |
| [`cpu`](#cpu) | CPU usage from `/proc/stat` and CPU frequency scaling data from sysfs. |
| [`diskstats`](#diskstats) | Disk I/O statistics from `/proc/diskstats`. |
| [`filefd`](#filefd) | File descriptor usage statistics. |
| [`filesystem`](#filesystem) | Statistics of mounted filesystems from `statvfs(2)`. |
| [`hwmon`](#hwmon) | Temperature, fan and voltage sensors from `/sys/class/hwmon`. |
| [`loadavg`](#loadavg) | Load average statistics. |
| [`mdstat`](#mdstat) | Multiple Device driver (Software RAID) information. |
| [`meminfo`](#meminfo) | Memory usage statistics from `/proc/meminfo`. |
| [`netdev`](#netdev) | Network device transmit/receive statistics from `/proc/net/dev`. |
| [`netif`](#netif) | Network interface information. |
| [`pressure`](#pressure) | Pressure Stall Information metrics. |
| [`rapl`](#rapl) | Running Average Power Limit statistics. |
| [`schedstat`](#schedstat) | Scheduler statistics. |
| [`stat`](#stat) | Basic statistics from `/proc/stat`. |
| [`textfile`](#textfile) | Custom metrics from `.prom` text files dropped in a directory. |
| [`timex`](#timex) | Selected adjtimex(2) system call stats. |
| [`uname`](#uname) | Node information returned by the `uname` system call. |
| [`vmstat`](#vmstat) | Virtual Memory statistics. |
| [`zfs`](#zfs) | Basic ZFS information. |

## Usage

By default, all collectors are enabled. Use a command line argument of
the form `--{collector}-off` (e.g., `--cpu-off`) to selectively
disable specific collectors. Alternatively, if you explicitly enable
any collector with `--{collector}-on`, only such explicitly enabled
collectors are actually enabled. You probably don't want to mix both
`on` and `off` flags in the same invocation.

The general command line arguments are documented below. Specific
collectors may also accept further arguments, which will always have
the prefix `--{collector}-`. They are documented in the [Collector
Reference](#collector-reference) section.

| Flag          | Description                                            |
| ----          | -----------                                            |
| `--port=X`    | Listen on port *X* instead of the default port (9100). |
| `--host=HOST` | Bind on HOST only (by default: all interfaces).        |
| `--stdout`    | Gather metrics once and print them to standard output. |

## Collector Reference

### `conntrack`

* `node_nf_conntrack_entries`: Number of currently allocated flow entries in conntrack.
* `node_nf_conntrack_max`: Maximum size of the connection tracking table.

### `cpu`

Metrics and labels:

* `node_cpu_seconds_total{cpu=N,mode=M}`: Number of CPU seconds spent
  in various different modes, as reported in `/proc/stat`. *N* is the
  CPU index, a number between 0 and one less than the number of
  (logical) CPUs in the system. The mode *M* is one of `user`, `nice`,
  `system`, `idle`, `iowait`, `irq`, `softirq` or `steal`. There will
  be one row for each CPU and each mode in the scrape.
* `node_cpu_frequency_hertz{cpu=N}`: The current CPU clock frequency
  in Hertz at the time of the scrape. This is the
  `cpufreq/scaling_cur_freq` value under the CPU-specific sysfs
  directory.

### `diskstats`

The metrics correspond to the columns of `/proc/diskstats`:

* `node_disk_reads_completed_total`: Total number of successfully
  completed disk reads.
* `node_disk_reads_merged_total` Total number of adjacent reads merged
  together.
* `node_disk_read_bytes_total`: Total number of bytes read from the
  device.
* `node_disk_read_time_seconds_total`: Total time spent in read
  requests.
* `node_disk_writes_completed_total`: Total number of successfully
  completed disk writes.
* `node_disk_writes_merged_total`: Total number of adjacent writes
  merged together.
* `node_disk_written_bytes_total`: Total number of bytes written to
  the device.
* `node_disk_write_time_seconds_total`: Total time spent in write
  requests.
* `node_disk_io_now`: Number of I/O operations currently in progress.
* `node_disk_io_time_seconds_total`: Total time spent in disk I/O.
* `node_disk_io_time_weighted_seconds_total`: Time spent in disk I/O
  weighted by the number of pending operations.
* `node_disk_discards_completed_total`: Total number of discard
  operations completed successfully.
* `node_disk_discards_merged_total`: Total number of adjacent discard
  operations merged.
* `node_disk_discarded_sectors_total`: Total number of discarded
  sectors. Note that this is in sectors, not bytes, unlike the
  corresponding read/write metrics.
* `node_disk_discard_time_seconds_total`: Total time spent in discard
  operations.

See the kernel's
[Documentation/iostats.txt](https://www.kernel.org/doc/Documentation/iostats.txt)
file for more details. The collector assumes the read/write totals are
reported using a sector size of 512 bytes.

All metrics have one label, `device`, containing the device name from
`/proc/diskstats`.

The `--diskstats-include=` and `--diskstats-exclude=` command line
arguments can be used to select which devices to report on. The format
for both is a comma-separated list of device names (e.g.,
`--diskstats-include=sda,sdb`). If an include list is provided, only
those devices explicitly listed are included. Otherwise, all devices
not mentioned on the exclude list are included. If the given value
ends in `*`, it matches any string that begins with the part before
the `*`; otherwise, the match must be exact.

By default, if the device is entirely unused (all metrics are 0), it's
omitted. Use `--diskstats-keep-unused` to include even those devices.

### `filefd`

Metrics:

* `node_filefd_allocated`: Number of currently used file descriptors.
* `node_filefd_maximum`: Maximum number of file descriptors available.

### `filesystem`

Metrics:

* `node_filesystem_device_error`: If set, `errno` during metrics collection.
* `node_filesystem_size_bytes`: Total size of the filesystem.
* `node_filesystem_free_bytes`: Number of free bytes in the
  filesystem.
* `node_filesystem_avail_bytes`: Number of free bytes available to
  unprivileged users.
* `node_filesystem_files`: Total number of inodes supported by the
  filesystem.
* `node_filesystem_files_free`: Number of free inodes.
* `node_filesystem_readonly`: Whether the filesystem is mounted
  read-only: `0` (rw) or `1` (ro).

Labels:

* `device`: Device node path mounted at the location.
* `fstype`: Mounted filesystem type.
* `mountpoint`: Location where the filesystem is mounted.

By default, all mounts where the device path starts with an `/` are
included. Command line arguments of the form
`--filesystem-{in,ex}clude-{device,type,mount}=X,Y,Z` can be used to
define comma-separated inclusion and exclusion lists for the device
path, filesystem type and mountpoint labels, respectively. Each of the
three categories is tested independently, and only if a mounted
filesystem passes all three tests, it is included in the metrics. For
each category, if an include list is specified, only the explicitly
listed values are accepted -- this overrides even the `/` prefix test
for devices. If an include list is not set, all values not on the
exclusion list are accepted. If the given value ends in `*`, it
matches any string that begins with the part before the `*`;
otherwise, the match must be exact.

The data is derived from scanning `/proc/mounts` and calling
`statvfs(2)` on all lines that pass the inclusion checks.

### `hwmon`

The `hwmon` collector pulls data from all the sysfs subdirectories
under `/sys/class/hwmon`. The supported entry types are temperature
(`temp*`), fan (`fan*` and voltage (`in*`) sensors.

Metrics:

* `node_hwmon_temp_celsius`: Current temperature in degrees Celsius.
* `node_hwmon_fan_rpm`: Current fan speed in RPM.
* `node_hwmon_fan_min_rpm`: Threshold for minimum fan speed.
* `node_hwmon_fan_alarm`: Active fan alarm signal: `0`/`1`.
* `node_hwmon_in_volts`: Input voltage measurement.
* `node_hwmon_in_min_volts`: Lower threshold for a voltage alarm.
* `node_hwmon_in_max_volts`: Upper threshold for a voltage alarm.
* `node_hwmon_in_alarm`: Active voltage alarm signal: `0`/`1`.

All the metrics have the same two labels: `chip` and `sensor`. The
`chip` label is derived from the sysfs directory path, while the
`sensor` label designates a specific sensor on the same (logical)
chip.

The values are by default directly as reported in sysfs: there's no
built-in scaling.

(TODO: potential future feature: configurable scaling via command line
options.)

### `loadavg`

Metrics:

* `node_load1`: Load average for the last minute.
* `node_load5`: Load average for the last 5 minutes.
* `node_load15`: Load average for the last 15 minutes.

### `meminfo`

The `meminfo` collector exposes all the rows from `/proc/meminfo`
under the metric name `node_memory_X`. The part *X* corresponds to the
label in `/proc/meminfo`, with the exception that non-alphanumeric
characters are replaced with `_`, and any remaining trailing `_`s are
removed.

If the line in `/proc/meminfo` has a `kB` suffix, the suffix `_bytes`
is also appended to the metric name, and the value multiplied by 1024
to convert it to bytes.

### `mdstat`

Multiple Device driver (Software RAID) information.

Metrics:

* `node_md_level`: `raid1` etc.
* `node_md_disks`: number of disks in RAID
* `node_md_metadata_version`
* `node_md_state`
* `node_md_chunk_size`
* `node_md_degraded_disks`: number of degraded disks in RAID
* `node_md_sync_action`
* `node_md_sync_completed`
* `node_md_sync_speed`
* `node_md_disk_state`

Labels:

* `device`
* `disk`
* `state`

### `netdev`

Metrics and labels:

* `node_network_receive_X{device=D}`: Metrics related to receiving
  data on network interface *D*.
* `node_network_transmit_Y{device=D}`: Metrics related to sending data
  on network interface *D*.

The exact set of metrics (`X` and `Y` above) depends on the columns
included in your `/proc/net/dev` file. A normal set is:

| Receive | Transmit | Metric | Description |
| ------- | -------- | ------ | ----------- |
| X | X | `bytes` | Byte counter |
| X | X | `packets` | Packet counter |
| X | X | `errs` | Errors while receiving/transmitting |
| X | X | `drop` | Dropped frame count |
| X | X | `fifo` | ? |
| X |   | `frame` | ? |
| X |   | `compressed` | ? |
| X |   | `multicast` | Byte count |
|   | X | `colls` | Collisions while transmitting |
|   | X | `carrier` | ? |

By default, statistics are reported for all network interfaces except
the loopback interface (`lo`). The `--netdev-include=` and
`--netdev-exclude=` options can be used to define a comma-separated
list of interface names to explicitly include and exclude,
respectively. If an include list is set, only those interfaces are
included. Otherwise, all interfaces *not* mentioned in the exclude
list are included. If the given value ends in `*`, it matches any
string that begins with the part before the `*`; otherwise, the match
must be exact.

### `netif`

Network interface statistics:

Metrics:

* `node_network_mtu_bytes`
* `node_network_carrier`
* `node_network_carrier_changes_total`
* `node_network_up`
* `node_network_speed_bytes`

Labels:

* `device`

### `pressure`

PSI (Pressure Stall Information) statistics.

Metrics:

* `node_pressure_cpu_waiting_seconds_total`
* `node_pressure_memory_waiting_seconds_total`
* `node_pressure_memory_stalled_seconds_total`
* `node_pressure_io_waiting_seconds_total`
* `node_pressure_io_stalled_seconds_total`

### `rapl`

Running Average Power Limit.

Metrics:

* `node_rapl_joules_total`

Labels:

* `rapl_zone`
* `name`

### `schedstat`

Scheduler statistics.

Metrics:

* `node_schedstat_running_seconds_total`
* `node_schedstat_waiting_seconds_total`
* `node_schedstat_timeslices_total`

### `stat`

This collectors exports the following metrics from `/proc/stat`:

* `node_boot_time_seconds`: System boot time as a Unix timestamp (seconds since 1970).
* `node_context_switches_total`: Total number of context switches done by the system.
* `node_forks_total`: Total number of forks since boot.
* `node_intr_total`: Total number of interrupts serviced.
* `node_procs_blocked`: Number of processes currently blocked for I/O.
* `node_procs_running`: Number of processes currently in runnable state.

### `textfile`

The `textfile` collector can be used to conveniently export custom
node-bound metrics. Metrics in any files ending in `.prom` in the
designated directory are included in the scrape. Generally you should
write to a file with a different suffix (say `.prom.tmp`) and then
atomically rename the file, to prevent the server from sending data
from incomplete metrics files.

The implementation in this program copies the file contents directly
to the outgoing HTTP response. It is your responsibility to make sure
the files conform to the Prometheus exposition format. The only
modification done is to add a terminating newline to the file, is one
is not already present.

### `timex`

Selected adjtimex(2) system call stats.

Metrics:

* `node_timex_sync_status`
* `node_timex_estimated_error_seconds`
* `node_timex_frequency_adjustment_ratio`
* `node_timex_loop_time_constant`
* `node_timex_maxerror_seconds`
* `node_timex_offset_seconds`
* `node_timex_status`
* `node_timex_tai_offset_seconds`
* `node_timex_tick_seconds`
* `node_time_seconds`
* `node_clock_tai_seconds`
* `node_clock_monotonic_seconds`
* `node_clock_boottime_seconds`
* `node_timex_pps_calibraton_total`
* `node_timex_pps_error_total`
* `node_timex_pps_frequency_hertz`
* `node_timex_pps_jitter_seconds`
* `node_timex_pps_jitter_total`
* `node_timex_pps_shift_seconds`
* `node_timex_pps_stability_exceeded_total`
* `node_timex_pps_stability_hertz`

### `uname`

The `uname` collector exports data from the eponymous system call as
labels attached to the metric `node_uname_info`, which always has the
value 1. The attached labels are:

* `machine`
* `nodename`
* `release`
* `sysname`
* `version`

See your `uname(2)` man page for details of the values.

### `vmstat`

Virtual memory statistics.

Metrics:

* `node_vmstat_oom_kill`: Number of ouf-of-memory kills.
* `node_vmstat_pgfault`: Number of page faults.
* `node_vmstat_pgmajfault`: Number of major page faults.
* `node_vmstat_pgpgin`: Number of page-ins.
* `node_vmstat_pgpgout`: Number of page-outs.
* `node_vmstat_pswpin`: Number of swap-ins.
* `node_vmstat_pswpout`: Number of swap-outs.

### `zfs`

ZFS Pool state.

Metrics:

* `node_zfs_zpool_state`
