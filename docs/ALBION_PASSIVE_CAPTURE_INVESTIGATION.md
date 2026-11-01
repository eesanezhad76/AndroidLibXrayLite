# Albion passive capture: Xray TUN ingress investigation

Date: 2026-10-30

## Scope

This note records a source-level investigation of the exact Xray-core revision pinned by this AndroidLibXrayLite checkout. The goal is a passive, opt-in copy of packets already being read by the existing V2RayNG TUN path. It must not create a second Android VPN service, read from a duplicated TUN descriptor, alter packet bytes, delay forwarding, or block the packet-processing goroutine.

This is an investigation checkpoint, not an implementation or a tested APK.

## Pinned dependency

The checked-out go.mod pins:

- github.com/xtls/xray-core v1.260327.1-0.20261021115930-1b1647cd9ddf
- upstream source commit: 1b1647cd9ddf0abc34b981d4611fc03bcac1ca74

## Confirmed read path

In Xray-core at that commit:

- `proxy/tun/tun_android.go` implements AndroidTun using the existing file descriptor passed through `xray.tun.fd`.
- AndroidTun duplicates the descriptor with `unix.FcntlInt(..., unix.F_DUPFD_CLOEXEC, 0)`. That duplicate is used by Xray's own reader; it is not a packet mirror.
- `proxy/tun/tun_fd.go`, `(*fdTun).ReadPacket`, calls `t.file.Read(b)` and returns the packet bytes after the optional vnet header.
- `proxy/tun/stack.go`, `(*stack).read`, calls `s.tun.ReadPacket(b)` and immediately passes the returned slice to `s.Input(p)`.

The safest source-level observation point is after a successful read in `fdTun.ReadPacket`, before returning the packet slice. The observer must copy bytes into its own bounded storage because the caller reuses the read buffer.

## Required design constraints

1. Disabled by default; explicitly enabled by the Android app.
2. Observe only packets read from the existing TUN descriptor. Do not add another VpnService.
3. Copy only a bounded number of bytes per packet into a preallocated bounded queue.
4. The read path must never wait for the observer. On queue full, drop the observation copy and increment a counter; never drop or delay the original packet.
5. Do not log payloads, credentials, or unrelated app traffic. Apply a narrow destination/UDP filter in the observer and make the feature visibly opt-in.
6. Provide a lifecycle-safe callback or drain API through gomobile. Avoid calling Java/Kotlin synchronously from the packet-read goroutine.
7. Validate packet direction and packet format with synthetic tests before testing against Albion.

## Integration blocker

The read path is inside the separate `github.com/xtls/xray-core` module. Adding code only to this AndroidLibXrayLite wrapper cannot insert an observer inside `fdTun.ReadPacket`.

A maintainable implementation therefore needs one of:

- a controlled fork of the exact Xray-core dependency, referenced with a Go `replace` directive; or
- an upstream extension point added to Xray-core and consumed by this wrapper.

The first route requires a separate fork of Xray-core and a pinned commit. Do not silently patch module cache contents or duplicate-read the TUN fd.

## Next implementation sequence

1. Create a fork of XTLS/Xray-core at the pinned revision.
2. Add a small observer interface and bounded nonblocking queue at the confirmed read point, with tests for disabled mode, queue overflow, byte-copy ownership, and unchanged packet return.
3. Add a narrow AndroidLibXrayLite API to enable/disable and drain copied packets without blocking Xray.
4. Point this wrapper at the pinned fork revision and run Go tests and the existing gomobile Android AAR workflow.
5. Only after a successful AAR build, wire the API into the V2RayNG Android app and test on-device.
6. Implement the Albion Photon/ENet parser separately. Do not claim resource locations, enchantment levels, or player coordinates until validated against real captured traffic.

## Current status

- Source path identified and verified against the pinned dependency.
- No observer code has been added.
- No Xray-core fork has been created yet.
- No AAR or APK has been built or device-tested.
