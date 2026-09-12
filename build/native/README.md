# Native device helpers

The source manifests (`*.sources`) are shared by the device build and host
tests. Each manifest links one static executable. Object files, headers and
libraries are never deployed. No helper uses pthreads.

| Executable | Owner |
| --- | --- |
| `mdns-advertiser` | DNS-SD records, packet responses, multicast sockets and Apple port takeover |
| `nbns-advertiser` | NetBIOS queries, node status and IPv4 response selection |
| `service` | On-demand NT hashing, live CIDRs and Samba bind selection |
| `telemetry` | Heartbeat collection/POST and scheduling |

`common/` contains live interface discovery and the shared LAN/WAN policy.
It has no daemon, cache or runtime state file. The advertisers independently
observe current interfaces. `service` is a command-line helper, not an IPC
server. Protocol-specific socket-family probes remain with their advertisers.

Module headers declare cross-module functions. `TC_LOCAL` keeps internal
helpers static in device builds; only host regression tests define
`TC_NATIVE_TEST` to link selected internal functions. This matters on NetBSD 4,
where the linker deliberately does not use section garbage collection because
it can discard required ELF notes. Do not include implementation `.c` files.

The device manager runs `mdns-advertiser` from Flash. NBNS, service, telemetry and Samba
are copied to RAM from the disk before use; service is staged before auth and
bind probes. This leaves Flash space for the next atomic mDNS update. Telemetry
locks the existing `/mnt/Memory` directory to exclude concurrent cycles,
including manual runs, without creating a lock file or job directory. That
directory must be root-owned, with sticky permissions if writable by other users.

## Telemetry protocol

`telemetry --daemon` sends a boot heartbeat and then one every 12 hours.
`--once [reason]` performs one cycle; `--print-payload [reason]` only prints.
`--cleanup` removes stale debug files without collecting or posting telemetry.
An existing cycle or inherited debug lock causes `--once` and `--cleanup` to
exit 75. Cleanup errors return 1 and prevent a new cycle from starting.

The POST to `/v1/router-heartbeats` preserves schema version 1 and its device
fields. It adds `target_lane` (`6`, `4le`, `4be`) and `debug_nonce` (16 random
bytes as 32 lowercase hex characters). Empty or legacy successful responses end
the cycle. JSON responses are validated with strict bounds and duplicate-field
checks, then ignored for execution control.

Telemetry does not download or execute remote binaries. Response bodies are
bounded to 4096 bytes. Curl config files are disabled; fixed endpoints are used
without following redirects.

TERM cancels preparation and stops future scheduling. Recovery runs before each
cycle and every 30 seconds while the daemon is idle, without PID files or
age-based guesses.

Reset and uninstall stop telemetry scheduling with TERM. Uninstall checks
`--cleanup` before deleting the runtime. Migration removes the old
`/mnt/Memory/tc-telemetry` tree recursively only after legacy
telemetry/debug/heartbeat processes have exited and no nested mounts remain.
New telemetry never recreates that tree.

## Builds and checks

Run the artifact helper in the existing NetBSD VM as root, for example
`./build/telemetry.sh`, `./build/telemetryoldle.sh`, or
`./build/telemetryoldbe.sh`. `make -C build advertisers-all` builds all four
helpers for all three lanes. It never downloads or rebuilds a toolchain.

After copying stripped outputs back, wait five seconds and refresh the artifact
manifest hashes. All 12 installed artifacts must be static ARM ELF executables
with the correct byte order and NetBSD note.

From the repository root:

```sh
./build/native/host-check.sh
.venv/bin/pytest
.venv/bin/pytest -n 4 --dist loadfile
TC_NATIVE_SANITIZERS=1 UBSAN_OPTIONS=halt_on_error=1 .venv/bin/pytest tests/native tests/test_deploy_modules.py
make coverage-native
```

The native suite includes checked-in packet and interface regression cases,
pure parser/scheduler tests, a local HTTP server and real curl. Test children
have deadlines and isolated workspaces. Coverage uses LLVM tools and retains its
HTML report under the temporary output directory printed at completion.
