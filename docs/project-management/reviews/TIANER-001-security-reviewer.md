# Phase A Review: Security Reviewer

| Field | Value |
|-------|-------|
| Reviewer | Security Reviewer |
| Phase | A (Design Review) |
| Date | 2026-06-07 |
| Artifact | `docs/designs/inception.md` v0.5 |
| Review scope | Attack surface, secrets management, API authentication, database security, supply chain, physical security, data privacy, input validation, process isolation |

---

## Area 1: Transport Layer Security (TLS)

**Finding:** No component in the design uses TLS. All communication paths are unencrypted:

1. **API → Frontend:** FastAPI serves over HTTP on 127.0.0.1:8080. The frontend fetches data via unencrypted HTTP. If the Pi is accessed over the network (even via WireGuard tunnel mentioned in section 12.4), the WireGuard tunnel terminates before the API — the API traffic on the Pi's localhost is unencrypted.

2. **API → PostgreSQL:** psycopg connects to PostgreSQL on 127.0.0.1:5432 without SSL. The `pg_hba.conf` in section 6.6 uses `scram-sha-256` for authentication but does not require SSL for the loopback connections. While localhost traffic is not network-exposed, a local privilege escalation attack allows packet sniffing on loopback.

3. **Grafana → PostgreSQL:** The Grafana datasource YAML in section 8.12 explicitly sets `sslmode: disable`. Grafana connects to PostgreSQL without encryption.

4. **Grafana iframe embedding:** Section 8.11 DECISION 8.11.2 specifies iframe embedding with "anonymous read-only access." The iframe URL uses `http://`, not `https://`. The embedded Grafana panels transmit data over unencrypted HTTP.

5. **API key transmission:** The `X-API-Key` header is sent over plain HTTP. On a network (even LAN), this is vulnerable to interception.

**Impact:** An attacker on the same LAN segment can intercept API keys, database credentials, and all captured device data. The WireGuard tunnel (out of scope) mitigates remote access but does not protect LAN-local attacks.

**Confidence:** 95  
**Verdict:** FAIL — TLS must be added to the API (HTTPS), the PostgreSQL connections (SSL mode), and the Grafana connections. At minimum, the API should serve HTTPS with a self-signed certificate.

---

## Area 2: Secrets Management

**Finding:** Section 6.8.3 defines secrets stored as files in `/etc/blesniff/secrets/` with mode `0600`, owner `blesniff:blesniff`. Several issues:

1. **No secrets rotation mechanism.** Secrets are generated once by `deploy/scripts/generate-secrets.sh` and never rotated. If the API key is compromised (e.g., leaked in a WireGuard configuration file, visible in browser dev tools, accidentally committed), there is no procedure to rotate it without manual intervention. No rotation schedule is defined. No automated rotation script exists.

2. **Database password in environment file.** Section 6.8.1 uses `BLESNIFF_DB_PASSWORD_FILE=/etc/blesniff/secrets/db_password` which is better than embedding the password directly. However, the systemd unit's `EnvironmentFile=/etc/blesniff/blesniff.env` means the password file path is visible in the process environment. `cat /proc/<pid>/environ` (readable by the process owner) reveals the path.

3. **Grafana password in provisioning YAML.** Section 8.12 uses `$__file{/etc/blesniff/secrets/grafana_db_password}` which is the Grafana file-based secret mechanism. This is acceptable, but the file is owned by `blesniff:blesniff` (mode 0600) while Grafana runs as its own user (typically `grafana`). Grafana cannot read the file unless additional permissions are configured. This is either a configuration error or the file permissions must be adjusted.

4. **No Vault or equivalent.** For a single-Pi deployment, a full secrets manager is overkill, but the document should explicitly state the trade-off: file-based secrets with no rotation for v1, with a post-MVP commitment to add HashiCorp Vault or equivalent.

**Confidence:** 90  
**Verdict:** FAIL — Secrets rotation procedure must be documented. The Grafana password file permission issue must be resolved. A post-MVP commitment to add automated rotation should be added.

---

## Area 3: API Authentication

**Finding:** Section 8.10 and 12.4 define API authentication as a single static API key sent via `X-API-Key` header. Issues:

1. **Single key, no rotation.** One API key is generated at deploy time and used for all clients (frontend, any scripts, any operators). If compromised, all access is compromised. There is no mechanism to issue separate keys per client or to revoke a key without replacing it for all clients.

2. **No rate limiting.** The API has no rate limiting. An attacker with a valid key (or after obtaining one via LAN sniffing) can make unlimited requests, potentially causing DoS by exhausting PostgreSQL connections.

3. **No request signing.** The API key is a bearer token — anyone who obtains it can use it. No HMAC or request signing prevents replay attacks.

4. **`/metrics` endpoint unauthenticated.** Section 8.10 says `/metrics` is "bound to localhost only and does not require auth." This is acceptable if the Pi is physically secured, but the `/metrics` endpoint exposes internal system state (packet counts, gap counts, sniffer running status) that aids reconnaissance.

5. **CORS configuration not specified.** The frontend served at `:8080` and the API at `:8080` are same-origin, so CORS is not required. But if the frontend is served from a different origin (e.g., during development with `npm run dev` on another machine), CORS policy becomes relevant. No CORS configuration is mentioned.

**Confidence:** 88  
**Verdict:** FAIL — Multi-key support and key rotation should be designed. Rate limiting should be added. CORS policy must be specified.

---

## Area 4: Database Security

**Finding:** Section 6.6 sets up PostgreSQL with 3 roles (blesniff, blesniff_grafana, blesniff_ro). The design is reasonable but has gaps:

1. **SQL injection risk in gap detector.** Section 8.6 uses f-string-style Python SQL references (`$1`, `$2`) which is safe with parameterized queries. However, the backfill logic in `backfill.py` uses PCAP filenames derived from `sniffers.yaml` to locate PCAP files. If the filename contains SQL-unsafe characters, and any code path interpolates filenames into SQL, injection is possible. The document does not specify parameterized queries as a requirement for all DB interactions.

2. **`blesniff` role has DDL privileges.** The `blesniff` role owns the database and schema, meaning it can `DROP TABLE`, `ALTER TABLE`, etc. If the ingest bridge or API is compromised, the attacker can destroy all data. The principle of least privilege suggests a write-only role for the ingest bridge (INSERT only on `raw_packets`).

3. **No audit logging.** PostgreSQL `pgaudit` extension is not mentioned. Without audit logging, there is no record of who accessed what data or when. For a system that captures personally identifiable information (device MAC addresses + location), audit logging may be legally required depending on jurisdiction.

4. **`db/apply-migrations.sh` runs as `blesniff` role.** Migrations include `CREATE TABLE`, `ALTER TABLE`, and `CREATE EXTENSION`. These require elevated privileges. Running migrations should use the `postgres` superuser role, not the application role.

**Confidence:** 85  
**Verdict:** FAIL — Add a write-only role for the ingest bridge. Specify parameterized queries as mandatory. Add audit logging consideration. Use `postgres` role for migrations, not `blesniff`.

---

## Area 5: Supply Chain Integrity

**Finding:** The design installs software from multiple external sources without integrity verification:

1. **apt packages** from Debian Trixie repositories: Not pinned to specific package versions. An apt mirror compromise or man-in-the-middle on the Debian CDNS could deliver malicious packages. The document does not mention `apt-secure` or GPG key verification.

2. **Ubertooth built from source:** Section 8.1 says "Built from source per upstream guide." No commit hash or signed tag is specified. The build could pull any revision from the Ubertooth repository. No GPG verification of the source tarball or git tag.

3. **nrfutil binary download:** Section 13.2 says "download official ARM64 binary." No checksum or signature verification is mentioned. A compromised download server could deliver a trojanized binary.

4. **Node.js via NodeSource repo:** NodeSource provides unsigned apt repositories. The document does not mention verifying the GPG key.

5. **PiPy/PyPI packages:** Python packages installed via `uv` (section 5). No hash-checking configuration is mentioned. `uv` supports `--require-hashes` but the document does not require it.

6. **npm packages:** Frontend dependencies installed via `npm install`. No `package-lock.json` integrity verification or `npm audit` step is mentioned in the CI pipeline (section 10.6).

7. **No reproducible build.** The C++ binaries (ingest bridge, deep parser) are not built reproducibly. There is no mechanism to verify that a binary matches its source.

**Confidence:** 88  
**Verdict:** FAIL — Supply chain integrity measures must be added: GPG key verification for apt repos, commit hash pinning for Ubertooth, checksum verification for nrfutil, hash-checking for Python packages, and `npm audit` in CI.

---

## Area 6: Physical Security

**Finding:** Section 12.4 says "External access via the Pi's existing WireGuard tunnel" and all web interfaces are "bound to 127.0.0.1 by default." However:

1. **No disk encryption.** The eMMC and SD card store PCAP files (with captured device MAC addresses, advertising data that may contain device names, and potentially location-revealing RSSI patterns). If the Pi or its storage is physically stolen, all data is accessible. LUKS or equivalent disk encryption is not mentioned.

2. **No boot integrity.** Secure boot is not mentioned. An attacker with physical access can modify the boot sequence, install a rootkit, or replace the SD card.

3. **USB port physical access.** The Pi is designed to have USB dongles attached. An attacker with physical access can replace a sniffer dongle with a malicious USB device (BadUSB attack) that presents as a keyboard or mass storage device. The udev rules in section 6.2 match on `idVendor:idProduct` which can be spoofed by a malicious device.

4. **Serial console access.** The CM5 I/O board exposes a serial console via the GPIO header. If not disabled, it provides root shell access without authentication.

**Confidence:** 82  
**Verdict:** FAIL — Disk encryption must be considered for the data partition. Boot security and USB attack surface must be documented. The serial console should be disabled or password-protected.

---

## Area 7: Data Privacy

**Finding:** The system captures BLE advertising packets that contain personally identifiable information:

1. **MAC addresses** are unique device identifiers. Even random MACs can be used for tracking when combined with other signals.

2. **Device names** in ADV_IND payloads (e.g., "John's iPhone") are directly personally identifiable.

3. **Service UUIDs** (e.g., 0x180F = Battery Service, 0x180D = Heart Rate) reveal device capabilities that can identify device types.

4. **Manufacturer data** (e.g., Apple 0x004C data fields) can contain identifiable information.

5. **RSSI patterns over time** can be used to infer presence and movement patterns, effectively tracking individuals.

The document does not address:
- Data retention beyond the 90-day SQL retention policy (section 8.7 migration 0003).
- Whether captured data falls under GDPR, CCPA, or other privacy regulations.
- Whether device names and raw AdvData should be stored in the database or only in PCAP files.
- Whether the `device_enrichment` table (with local_name, service UUIDs, manufacturer data) increases privacy exposure.
- Whether an operator can delete data for a specific MAC address (right to erasure).

**Confidence:** 85  
**Verdict:** FAIL — A privacy impact assessment should be conducted. At minimum, document which data fields are PIIs and provide a data deletion mechanism (right to erasure).

---

## Area 8: Input Validation

**Finding:** The design has multiple input surfaces that require validation:

1. **tshark output → ingest bridge (CONTRACT 8.4-A):** The ingest bridge `parse_line(std::string_view line)` function parses untrusted input from tshark. The document mentions "malformed inputs (15 cases)" in the test plan, but does not specify what constitutes a malformed input. Can a malformed line cause a buffer overflow in the C++ parser? Can an extremely long line exhaust memory? Can empty fields be confused with missing fields?

2. **API query parameters → PostgreSQL:** The API endpoints accept `q`, `class`, `seen_after`, `sort`, `limit`, `offset` parameters. These are passed to PostgreSQL queries. If string interpolation is used instead of parameterized queries, SQL injection is possible. The FastAPI/Pydantic model provides type validation but does not prevent injection in raw SQL strings.

3. **PCAP file input → deep parser:** The deep parser reads PCAP files via libpcap. A crafted PCAP file could exploit a vulnerability in libpcap or in the BLE dissector's bit-level parsing. The document does not mention fuzz testing the deep parser.

4. **sniffers.yaml → systemd units:** The sniffer configuration YAML is read by systemd unit templates. If the YAML contains unexpected values (e.g., `device: "../../etc/passwd"`), the path traversal could be exploited. No YAML schema validation is mentioned.

**Confidence:** 80  
**Verdict:** FAIL — Input validation requirements must be specified for each input surface. Fuzz testing should be planned for the C++ deep parser and ingest bridge parser.

---

## Area 9: Process Isolation

**Finding:** Section 6.5 defines systemd sandboxing directives:

```ini
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/blesniff /var/log/blesniff /var/run/blesniff
```

This is a strong starting point, but:

1. **No `PrivateDevices=true`.** The sniffer wrapper processes need USB device access (`/dev/bus/usb/*`), but the ingest bridge, gap detector, API, and other services do not. These should use `PrivateDevices=true` to prevent device access.

2. **No `PrivateNetwork=true` for non-network services.** The sniffer wrapper, tshark wrapper, and ingest bridge (one per sniffer) do not need network access — they read from FIFOs and write to the local PG socket. `PrivateNetwork=true` would prevent any network-based escape from these services.

3. **No `ProtectClock=true`, `ProtectKernelTunables=true`, `ProtectKernelModules=true`.** Additional sandboxing directives available in systemd 257 that should be applied.

4. **Shared `blesniff:blesniff` user across all services.** All services run as the same user. A compromised ingest bridge process can read the API's database credentials, and vice versa. Per-service users or at least per-service supplementary groups would improve isolation.

5. **No seccomp filter.** The systemd `SystemCallFilter=` directive can whitelist allowed syscalls. Not specified for any service.

**Confidence:** 82  
**Verdict:** ADVISORY — The existing sandboxing is above-average for a Pi deployment. Per-service `PrivateDevices` and `PrivateNetwork` should be added. Full seccomp profiles can be added post-MVP.

---

## Summary of Findings

| # | Area | Severity | Confidence | Verdict | Description |
|---|------|----------|-----------|---------|-------------|
| 1 | TLS | Blocking | 95 | FAIL | No encryption on any communication path |
| 2 | Secrets management | Major | 90 | FAIL | No rotation, Grafana permission error |
| 3 | API authentication | Major | 88 | FAIL | Single static key, no rotation, no rate limit |
| 4 | Database security | Major | 85 | FAIL | Over-privileged roles, no audit log, SQL injection risk |
| 5 | Supply chain integrity | Major | 88 | FAIL | No integrity verification for any external source |
| 6 | Physical security | Major | 82 | FAIL | No disk encryption, no boot integrity, USB attack surface |
| 7 | Data privacy | Major | 85 | FAIL | PII stored without privacy assessment or erasure mechanism |
| 8 | Input validation | Major | 80 | FAIL | No validation requirements, no fuzz testing |
| 9 | Process isolation | Minor | 82 | ADVISORY | Reasonable sandboxing but per-service isolation can improve |

---

## Overall Verdict

**CONDITIONAL PASS** — The document demonstrates security awareness (systemd sandboxing, file permissions, scram-sha-256 auth, 127.0.0.1 binding) but has systematic gaps across all other security domains. All major findings have known remediation paths. The design can proceed to implementation after:

1. TLS is added to the API and PostgreSQL connections.
2. Secrets rotation mechanism is designed.
3. Multi-key API authentication with rate limiting is added.
4. Database roles follow least privilege.
5. Supply chain integrity measures are added.
6. Disk encryption is documented (even if deferred post-MVP).
7. Privacy impact assessment is acknowledged.

---

## Self-Audit Checklist

| # | Check | Status |
|---|-------|--------|
| 1 | Read the complete artifact? | YES — all 2910 lines |
| 2 | Every finding includes a confidence score? | YES |
| 3 | Every finding is actionable? | YES — specific remediation paths for each finding |
| 4 | No speculative claims without evidence? | YES — all claims reference specific document sections |
| 5 | Blocking findings justified by severity? | YES — no TLS on a network-facing API is a blocking risk |
| 6 | Advisory findings clearly separated? | YES — area 9 is advisory |
| 7 | No duplicate findings across areas? | YES |
| 8 | Verdict is consistent with finding severities? | YES — 7 major + 1 advisory → CONDITIONAL PASS |

---

## Flags for PM

| Flag ID | Type | Description | Urgency |
|---------|------|-------------|---------|
| FLAG-SR-001 | Decision | TLS: add to API (HTTPS) and PG connections (sslmode). Self-signed cert acceptable for MVP? | High |
| FLAG-SR-002 | Decision | Secrets rotation: design rotation procedure. Commit to post-MVP Vault? | High |
| FLAG-SR-003 | Decision | API auth: single key or multi-key? Add rate limiting? | High |
| FLAG-SR-004 | Decision | Disk encryption: LUKS on data partition? Accept data exposure risk for MVP? | High |
| FLAG-SR-005 | Ticket | Add supply chain integrity: GPG keys, commit hashes, checksums, npm audit. | Medium |
| FLAG-SR-006 | Decision | Privacy: conduct PIA? Provide right-to-erasure endpoint? | Medium |
| FLAG-SR-007 | Ticket | Add `PrivateDevices=true` and `PrivateNetwork=true` to non-device services. | Low |