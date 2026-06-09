# Podman Container Security Recommendation for Tian'er

**Author:** Security Reviewer
**Date:** 2026-06-07
**Trigger:** User decision: ALL components run in Podman containers, USB devices passed through from host.

## Summary

Rootless Podman with --group-add keep-groups, zero --privileged containers, digest-pinned images, two-pod network architecture.

## Key Recommendations

1. ROOTLESS PODMAN — containers run as tianer user. USB passthrough via --device and --group-add keep-groups. No sudo, no rootful.
2. ZERO PRIVILEGED CONTAINERS — --cap-drop ALL as default. C03 only: CAP_NET_RAW + CAP_NET_ADMIN for dumpcap.
3. CUSTOM SECCOMP for C03 — add USB ioctl syscalls to default profile. Do NOT use unconfined.
4. VOLUME MOUNTS — /etc/tianer/ is :ro everywhere. /var/lib/tianer/pcap/ is :ro for C06/C07. Never mount podman socket.
5. TWO-POD NETWORK — tianer-capture pod (Network=none), tianer-platform pod (bridge network), PostgreSQL standalone on tianer-net. Only API port 8080 published to host.
6. DIGEST-PIN ALL IMAGES — Image=...@sha256: only. No :latest or floating tags.
7. MULTI-STAGE CONTAINERFILES — From debian:12-slim, one Containerfile per component.
8. QUADLET STRUCTURE — .image for pre-pulled, .build for locally-built, .container for runtime. Deploy to /etc/containers/systemd/.
9. COSIGN SIGNING — Post-MVP. Digest pinning sufficient for MVP.
10. HOST udev — Provide persistent device names via udev rules on host. Containers receive symlinks.
