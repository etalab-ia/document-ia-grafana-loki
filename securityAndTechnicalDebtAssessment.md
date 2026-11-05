# Security and Technical Debt Assessment

## Security Audit - Compliance and Remediation Report

**Date:** 2026-11-05  
**Repository:** https://github.com/etalab-ia/document-ia-grafana-loki  
**Audited revision:** `c21aaeea6bd15022029f9a13dec25993647d7367` on `main`  
**GitHub settings reviewed:** live snapshot on 2026-11-05  
**Scope:** all 12 tracked files, repository history and settings, deployment and runtime configuration, CI/CD, dependencies, upstream advisories, durability, and technical debt  
**Overall domain result:** 0/13 applicable domains OK (0%): 0 OK, 7 KO, 6 Partiel, 1 NA  
**Interpretation:** repository-level assessment, not an ANSSI certification or a production penetration test

## Executive summary

No unauthenticated Internet-facing compromise or public bypass of Caddy's Loki push policy was confirmed. The intended public route requires POST, the exact push path, an allowed client IP, and the exact bearer value; other `/loki/*` routes return 403. Scalingo overwrites `X-Real-IP`, which prevents the usual public spoofed-header path [E0028], [E0029], [E0030].

The highest-priority findings are:

1. **Docker images built with real `.env` files persist credentials in several image locations, and debug mode can print them into build logs.** Rotate any credentials used in shared images or logs, then remove secret material from the build context and image [E0001], [E0002], [E0038].
2. **Grafana 13.1.4 misses security fixes through 13.1.9.** Relevant issues include Editor-to-Org-Admin stored XSS, Viewer-triggered denial of service, contact-point authorization bypass, alert-rule disclosure, and paused-dashboard configuration disclosure [E0014], [E0015], [E0016], [E0017], [E0034].
3. **Loki is not loopback-only despite the README.** HTTP and an ephemeral gRPC listener bind to all interfaces with authentication disabled. A private-network peer that can reach them bypasses Caddy. The gRPC runtime is also affected by unauthenticated denial-of-service advisories [E0008], [E0009], [E0010], [E0022], [E0032], [E0059].
4. **Thirty-day physical log retention is not durable across restarts.** Pending-deletion markers are stored in `/tmp`, which Scalingo replaces on restart. Configure object-store deletion markers [E0023], [E0024], [E0053], [E0054].
5. **Shared Scalingo PostgreSQL connections are downgraded from `sslmode=prefer` to `disable`.** Use at least `require`, preferably `verify-full` with the correct CA configuration [E0003], [E0004], [E0005].
6. **Grafana bootstrap and encryption are not fail-closed.** Missing admin variables can produce `admin/admin`; missing PostgreSQL makes state ephemeral; missing `GF_SECURITY_SECRET_KEY` selects the published default encryption key [E0006], [E0007], [E0031], [E0041].
7. **Local Compose publishes a PostgreSQL superuser with `postgres/postgres` on host port 5430.** This is a development-only, host-network-dependent exposure [E0036], [E0069], [E0070], [E0071].

Positive controls include correct public Caddy route partitioning, Grafana loopback binding, anonymous access and embedding disabled, brute-force username protection, Git-ignored `.env`, managed PostgreSQL backup availability, a current PostgreSQL 15.19 engine, a supported Scalingo-24 stack, and successful GitGuardian checks on both pull requests [E0052], [E0057].

## Priority action plan

| Priority | Action | Evidence |
|---|---|---|
| Immediate | Rotate credentials potentially present in images or build logs; remove `.env` from Docker context and final images; redact debug output | [E0001], [E0002], [E0039] |
| Immediate | Upgrade Grafana to **13.1.9** (or a later supported release), Loki to **3.7.9**, and Caddy to **2.11.9**; test dashboards, alerting, push, S3, and Grafana Live | [E0014], [E0032], [E0037], [E0049], [E0060] |
| Immediate | Set Loki `http_listen_address` and `grpc_listen_address` to `127.0.0.1`; do not treat gRPC port `0` as disabled | [E0022], [E0059], [E0062] |
| Immediate | Replace the PostgreSQL substitution with `sslmode=require` or a verified `verify-full` setup | [E0003], [E0005] |
| Immediate | Set `compactor.deletion_marker_object_store_prefix` to an S3 prefix ending in `/`; verify retention across a forced restart | [E0024], [E0054] |
| Immediate | Require explicit admin credentials, a unique stable `GF_SECURITY_SECRET_KEY`, PostgreSQL, S3, token, CIDR, and port settings before startup | [E0006], [E0031], [E0042] |
| High | Enable Scalingo Force HTTPS and set `GF_SECURITY_COOKIE_SECURE=true`; define a tested CSP and protect or disable unauthenticated metrics | [E0031], [E0035], [E0066], [E0067] |
| High | Verify downloads with repository-pinned digests, use Caddy Sigstore verification, pin base-image digests, and generate/scan a final SBOM | [E0012], [E0013], [E0044], [E0075] |
| High | Require PostgreSQL rather than ephemeral SQLite; enforce a single replica unless Loki is redesigned for distribution; add readiness and graceful-drain handling | [E0019], [E0020], [E0021], [E0041] |
| High | Bind the Compose database to `127.0.0.1:5430:5432` or remove its host publication; use non-default development credentials | [E0036], [E0070], [E0072] |
| Moderate | Replace irrelevant failing CodeQL jobs with ShellCheck, Dockerfile/config validation and integration tests; protect `main` and require review/checks | [E0025], [E0055], [E0056] |
| Moderate | Triage the 71 alerts tied to absent `go.mod`; scan actual binaries/SBOM; add a license and `SECURITY.md`; set Actions read-only by default | [E0026], [E0027], [E0076] |

## ANSSI domain summary

Statuses are **OK / KO / Partiel / NA**. KO distinguishes an exploitable path from a missing defense-in-depth control. Organizational controls that cannot be assessed from the repository are identified rather than presumed absent.

| # | Domain | Status | Detail |
|---|---|---|---|
| 1 | TLS / HTTPS | **KO** | Shared-addon PostgreSQL TLS is explicitly disabled; Force HTTPS and Secure cookies are not configured in the repository |
| 2 | Cryptography | **Partiel** | No custom cryptographic construction found, but Grafana's published default key remains available unless explicitly overridden |
| 3 | Secret management | **KO** | `.env` and expanded secret files persist in images; debug output can disclose values; Grafana's default key is not overridden |
| 4 | Authentication, MFA and passwords | **KO** | Missing admin variables fall back to `admin/admin`; documented deployment is password-based with no MFA/SSO setup |
| 5 | Input validation and databases | **Partiel** | No application input/SQL code is present; Compose exposes a known superuser, and backend TLS is weakened |
| 6 | Dependencies and third-party components | **KO** | Core components miss security fixes; archives and base images lack immutable integrity verification |
| 7 | Logging | **Partiel** | Centralization and 30-day retention exist, but deletion markers are ephemeral and internal Loki APIs bypass access controls |
| 8 | Browser and API security | **Partiel** | Good route partitioning and safe defaults coexist with disabled enforcing CSP, insecure cookie/HTTP mode, public metrics, and a token wildcard footgun |
| 9 | System isolation | **KO** | Loki HTTP and gRPC listen on all interfaces without authentication, allowing direct internal access outside Caddy |
| 10 | Containers and deployment | **Partiel** | Loopback Grafana and read-only-compatible paths exist, but durability, readiness, single-replica, and graceful-drain invariants are unenforced |
| 11 | Development pipeline (CI/CD) | **KO** | No relevant functional tests; CodeQL repeatedly fails; `main` is unprotected; Actions defaults are broader than necessary |
| 12 | Developer workstation | **NA** | Hardware, disk encryption, endpoint protection, and workstation administration are outside repository evidence |
| 13 | Backup and continuity | **Partiel** | Managed PostgreSQL backups exist, but the addon is optional and Loki/S3 recovery, WAL durability, and restore testing are undocumented |
| 14 | Incident management | **KO** | No repository security contact, incident/recovery runbook, or vulnerability disclosure policy; organizational procedures remain unknown |

No conditional C, Rust, or CMS module applies to this repository.

## Detailed non-conformities

### [KO] Domain 1 - TLS / HTTPS

- **Rule:** protect database traffic in transit [ESS-BDD]; HTTPS on production exchanges [WEB R1]; HSTS [WEB R2]; Secure cookies [WEB R29].
- **Observation:** `bin/run:30-35` rewrites `sslmode=prefer` to `disable`. Scalingo shared addons document `prefer` as their default URI mode and support TLS. Scalingo also serves HTTP and HTTPS by default unless Force HTTPS is enabled [E0003], [E0004], [E0035].
- **Exploitability:** conditional on a shared addon and an on-path attacker for confidentiality/integrity impact; Force TLS instead causes an availability failure. HTTP exposure is conditional on missing external Force HTTPS/HSTS [E0040], [E0068].
- **Risk:** database credentials, session data, queries, and Grafana records can be exposed or modified; HTTP clients can transmit credentials, cookies, or bearer tokens without encryption.
- **Remediation:** use `require` at minimum and `verify-full` with the correct CA where possible; enable Force HTTPS and `GF_SECURITY_COOKIE_SECURE=true` [E0005], [E0066], [E0067].
- **Priority:** **High**.

### [KO] Domain 3 - Secret management

- **Rule:** no secrets in code, job logs, or repositories [ESS-DEVSECOPS 4]; change default authentication elements [HYGIENE M12].
- **Observation:** `Dockerfile:2-6` copies the entire context and `.env`, and `bin/env.sh:25-34` expands every entry into `/env`. Debug mode prints complete key/value pairs. README examples place admin, push-token, and S3 credentials in `.env` [E0001].
- **Exploitability:** exploitable by anyone who can pull/export the image or read applicable build logs. No real credential was found in tracked source, and the native Scalingo slug path may not use this Dockerfile [E0039].
- **Risk:** compromise of Grafana, database, object-storage, or Loki ingestion credentials. Missing `GF_SECURITY_SECRET_KEY` also selects Grafana's published default encryption key [E0031].
- **Remediation:** remove `.env` from context and image, add a strict `.dockerignore`, use runtime-only injection and `.env.example`, redact debug output, and set a unique stable Grafana key with a documented data-key rotation process. Rotate credentials used in shared images/logs; deleting a file in a later layer is insufficient [E0002], [E0038].
- **Priority:** **High**, becoming **Critical for any image/log known to contain real credentials**.

### [KO] Domain 4 - Authentication, MFA and passwords

- **Rule:** change default credentials [HYGIENE M12]; MFA on privileged access [MFA R1] [ESS-DEVSECOPS 8].
- **Observation:** admin variables are documented but not enforced. Grafana 13.1.4 defaults to `admin/admin` on first initialization. Without PostgreSQL, Scalingo's ephemeral filesystem can recreate that initial state and lose later password changes [E0006], [E0007], [E0041].
- **Exploitability:** remotely exploitable when defaults are present and strong values were never supplied. A persistent database with a changed password is not reset; no current production credential state was inspected [E0042].
- **Risk:** full Grafana administration, log disclosure, datasource modification, malicious dashboards, and persistence.
- **Remediation:** fail startup when admin variables are absent, require PostgreSQL, verify the current credential, and integrate an identity provider that enforces MFA. Keep any break-glass admin unique and monitored.
- **Priority:** **High**.

### [KO] Domain 6 - Dependencies and third-party components

- **Rule:** evaluate and apply security fixes before deployment [ESS-DEVSECOPS 5]; maintain components [WEB R62] [HYGIENE M34]; preserve artifact integrity [ESS-DEVSECOPS 7].
- **Observation:** Grafana 13.1.4, Loki 3.7.7, and Caddy 2.11.4 are behind security/reliability releases. `bin/compile` extracts downloaded binaries without checksum/signature verification, and images use mutable tags rather than digests [E0012], [E0013].
- **Exploitability:** varies by advisory and role. Relevant confirmed paths include Editor stored XSS, Viewer or authenticated denial of service, contact-point permission bypass, private-network gRPC denial of service, and conditional reverse-proxy response poisoning. Non-enabled SAML, SCIM, Auth Proxy, Azure Monitor, Git Sync, SQL-datasource, FastCGI, and file-server branches are excluded [E0034], [E0048], [E0051], [E0062], [E0065].
- **Risk:** privilege escalation, credential disclosure, persistent outage, and supply-chain code execution.
- **Remediation:** upgrade all three components, verify artifacts before extraction, use Caddy's Sigstore procedure, pin image digests, and scan a generated SBOM [E0044], [E0060], [E0064], [E0075].
- **Priority:** **High**.

### [KO] Domain 9 - System isolation

- **Rule:** default-deny and expose only required interfaces [CLOIS R1, R7]; apply least privilege [HYGIENE M9].
- **Observation:** Loki has `auth_enabled: false` and no `http_listen_address`. The pinned server always creates HTTP and gRPC listeners; empty addresses bind all interfaces, and gRPC port `0` selects an ephemeral port rather than disabling it [E0008], [E0009], [E0022].
- **Exploitability:** conditional on direct private-network, accidental port-publication, or SSRF reachability. Public Caddy does not forward these ports. Scalingo private-network containers can communicate on arbitrary ports [E0010], [E0043].
- **Risk:** unauthenticated log query/push and operational/destructive Loki API access; gRPC 1.82.1 also permits unauthenticated heap exhaustion, with additional Go HTTP/2 crash advisories [E0032], [E0059], [E0061].
- **Remediation:** set both listen addresses to `127.0.0.1`, retain network restrictions, and test the effective sockets. If gRPC is unnecessary, disable it through a supported mechanism instead of port `0` [E0062].
- **Priority:** **High**.

### [KO] Domain 11 - Development pipeline (CI/CD)

- **Rule:** automated security regression and IaC tests [ESS-DEVSECOPS 6]; protected branches and blocking controls [DINUM].
- **Observation:** there are no tracked tests or workflows. Dynamic CodeQL runs fail because configured languages have no source, including the 2026-10-31 scheduled run. `main` has no protection or rulesets, and Actions defaults allow write and PR approval [E0025], [E0055].
- **Exploitability:** no direct attack was established; this is a control failure that lets regressions merge and normalizes ignoring CI.
- **Risk:** routing, authentication, retention, persistence, and shutdown regressions reach production without relevant verification.
- **Remediation:** replace irrelevant CodeQL languages with ShellCheck, Dockerfile/Caddy/Loki/Compose validation, and integration tests for startup, allowed/denied push, queries, persistence, retention, and signals. Require those checks and review on `main`, with read-only Actions permissions by default [E0056].
- **Priority:** **Moderate**. GitGuardian and one approval on PR #1 are retained as counterevidence [E0057].

### [KO] Domain 14 - Incident management

- **Rule:** incident contact/procedure and capacity to revoke access [DINUM]; regular corrective controls [HYGIENE M38, reinforced level].
- **Observation:** no `SECURITY.md`, recovery runbook, retention validation, or public/private vulnerability-reporting route exists; standalone issues are disabled [E0027].
- **Exploitability:** missing response process, not a direct exploit. Organizational procedures may exist outside the repository and were not assessed.
- **Risk:** delayed credential revocation, inconsistent recovery, and uncoordinated disclosure.
- **Remediation:** publish a private security contact and supported-version policy, document revocation/rotation and recovery, and run restoration/incident exercises.
- **Priority:** **Moderate**.

## Partial conformities and technical debt

| Area | What works | Missing or risky element | Remediation |
|---|---|---|---|
| Log retention | TSDB v13, S3, 24-hour index, retention and delete-request store are correctly enabled [E0023] | Deletion markers live in ephemeral `/tmp`; pending physical deletions can be lost [E0024], [E0053] | Use S3 deletion markers; test a restart during retention; only use a carefully scoped chunk lifecycle backstop [E0054] |
| Durability and scaling | PostgreSQL and S3 are the intended persistent backends; PostgreSQL managed backups exist [E0052] | PostgreSQL is optional, WAL is local with `flush_on_shutdown=false`, and the single-process ring/replication model is unsafe above one replica [E0020], [E0021] | Require managed persistence; document/enforce one replica and the compactor singleton; test RPO/RTO |
| Shutdown and readiness | Child failure stops the rest of the stack; all children receive SIGTERM | Startup has no readiness ordering or healthchecks; shutdown has no drain/deadline and can report normal termination as failure | Stop Caddy first, drain, stop Grafana/Loki, wait for every child with bounded timeouts, and add healthchecks |
| Browser and API | Anonymous access and embedding are disabled; brute-force username protection is enabled; route partitioning is correct [E0028] | Enforcing CSP is disabled, cookie Secure/HSTS are not set, metrics are unauthenticated, and a token ending in `*` becomes a prefix matcher [E0031], [E0058] | Define/test CSP, protect metrics, force HTTPS, and reject wildcard characters in tokens |
| Compose development | PostgreSQL is supplied and only Caddy is intentionally web-published | Superuser is `postgres/postgres`, host port 5430 is published, there is no named volume, and binaries are amd64-only [E0069] | Bind to loopback or omit the port, change development credentials, document disposability, and support/document emulation |
| Documentation | Versions, main routes, and provisioning are described | README says Caddy terminates SSL although Scalingo does; "one click" omits addon/buildpack setup; retention deletion and single-replica constraints are absent | Correct the deployment guide and add required environment, network, retention, and recovery sections |
| Build maintenance | Versions are pinned and downloads use HTTPS with retries [E0045] | `CACHE_DIR` and `STACK` are unused, `KEYCLOAK_PATH` is unrelated, `/build` remains in the image, and no dependency lock manifest exists | Remove dead code and duplicate files; add version/digest locks and update automation |
| Security telemetry | GitGuardian succeeds on PRs; vulnerability alerting is active | All 71 open Dependabot alerts reference absent `go.mod`, so they do not describe deployed binaries [E0026], [E0076] | Confirm alert provenance and clean stale data; scan actual binaries/final SBOM |
| Governance | Commit SHAs and signatures provide an audit anchor | No license, `SECURITY.md`, tags, releases, changelog, CODEOWNERS, or issue templates; PR #2 has been open without review since 2026-09-07 | Choose a license, add ownership/disclosure docs, tag reviewed releases, and set Actions least privilege |

## Vulnerability and lifecycle details

### Core component status

| Component | Audited pin | Recommended target | Assessment |
|---|---:|---:|---|
| Grafana OSS | 13.1.4 | **13.1.9** or later supported | 13.1 branch has patch support through 2027-03-20; 13.1.4 misses multiple relevant fixes |
| Loki | 3.7.7 | **3.7.9** | Updates gRPC, x/net, Go toolchain, WAL handling, and request-body limits |
| Caddy | 2.11.4 | **2.11.9** | Current project policy supports only the latest 2.x; includes the relevant Go and proxy fixes |
| Scalingo base | implicit `latest` on scalingo-24 | Pin v11-or-later digest at update time | Supported stack; actual deployed digest cannot be inferred [E0075] |
| PostgreSQL (Compose) | 15.19 tag | Current 15.x; pin digest | Current minor, supported through 2027-11-11; pin for reproducibility |

### Grafana advisories relevant to the enabled configuration

| Advisory | Vendor severity | Practical precondition | Fixed in 13.1 line |
|---|---|---|---|
| CVE-2026-15815 plugin Zip Slip/RCE | High 8.8 | Operator installs a crafted plugin, including a seemingly legitimate signed plugin; not triggered by checked-in files [E0017] | 13.1.6 |
| CVE-2026-76154 Geomap stored XSS | High 7.3 | Editor hosts malicious style; victim interaction; can escalate to Org Admin [E0015] | 13.1.6 |
| CVE-2026-13719 restricted alert-rule disclosure | Medium 4.3 | Authenticated user and restricted alert rules exist; no datasource credentials exposed [E0034] | 13.1.7 |
| CVE-2026-13720 forged provisioning provenance | Medium 5.4 | Editor uses dashboard API; administrators then cannot edit/delete via UI [E0034] | 13.1.7 |
| CVE-2026-81841 paused shared-dashboard disclosure | Medium 5.3 | Existing paused public link; impact is reduced here because Loki uses proxy access and has no stored credential [E0034] | 13.1.7 |
| CVE-2026-33374 conversion API OOM | Medium 6.5 | Authenticated user sends unbounded body to built-in endpoint [E0034] | 13.1.8 |
| CVE-2026-100105 contact-point authorization bypass | High 7.7 | User can edit contact points but not protected fields; credentials and notification destinations at risk [E0034] | 13.1.9 |
| CVE-2026-88858 expression parser DoS | Medium 6.5 | Existing Viewer account; provisioning can make the outage persistent [E0016] | 13.1.9 |

The official advisory fixed-version tables are authoritative. GitHub Advisory and OSV mirrors are derivative and do not independently reproduce exploitation [E0046], [E0047].

### Loki dependency and reliability findings

| Issue | Reachability and impact | Fix |
|---|---|---|
| CVE-2026-84304 gRPC DATA fragmentation | Unauthenticated peer can exhaust heap memory if it reaches the ephemeral gRPC listener; not exposed through Caddy [E0032], [E0059], [E0062] | gRPC 1.83.1; included in Loki 3.7.8+ |
| Go HTTP/2 and HPACK issues, including CVE-2026-97032 | Malicious HTTP/2 client can crash the server when the listener is reachable [E0061] | Go 1.26.9+ and updated x/net; Loki 3.7.9 |
| CVE-2026-111943 x509 trust update | Requires a chain through known-untrusted intermediates and a system trust store that has not removed them [E0061] | Go 1.26.10; Loki 3.7.9 |
| Concurrent WAL Append inconsistency | Can produce inconsistent order and irrecoverable WAL state; relevant to acknowledged-log recovery [E0021] | Loki 3.7.9 |
| Unbounded ruler/LogQL Analyzer bodies | Availability hardening; those APIs are blocked by the checked-in public Caddy policy | Loki 3.7.9 |

No new first-party Loki CVE directly affecting 3.7.7 was found in the Loki v3 module query. This does not establish that the prebuilt binary and embedded dependencies are clean.

### Caddy findings

| Advisory | Practical assessment | Fix |
|---|---|---|
| CVE-2026-55458 WebSocket hop-by-hop header leak, Low | Affects the pin and Grafana Live exists, but requires malicious/malformed upstream behavior. This Caddyfile is plaintext HTTP without H2C, limiting HTTP/2/3 amplification [E0018], [E0051] | Use 2.11.9; the advisory's 2.11.5 target has no available public tag |
| CVE-2026-56866 CONNECT response poisoning, High 7.5 | Caddy's SBOM confirms Go 1.26.3. Exploitation needs a precise CONNECT-with-body, upstream rejection, and pooled-connection reuse sequence; not normal application traffic and not dynamically tested [E0033], [E0064], [E0065] | Go 1.26.9+; included in current Caddy |

## Claim verification matrix

| Claim | Evidence | Counterevidence checked | Confidence |
|---|---|---|---|
| Docker builds can disclose `.env` secrets | [E0001], [E0002], [E0038] | `.env` is ignored and untracked; native Scalingo path not proven [E0039] | High for Dockerfile builds; unknown for production slug |
| Shared PostgreSQL TLS is disabled | [E0003], [E0004], [E0005] | `require`/`verify-full` unaffected; dedicated addon uses `require` [E0040] | High, conditional on shared addon |
| Grafana can bootstrap default admin and lose state | [E0006], [E0007], [E0041] | Strong values and persistent changed-password DB avoid it [E0042] | High as a configuration hazard; current state unknown |
| Loki bypass is possible internally | [E0008], [E0010], [E0022] | No public Caddy bypass [E0043] | High for listener behavior; reachability conditional |
| Downloads are not authenticated or reproducible | [E0012], [E0013], [E0044], [E0075] | Versions and HTTPS are pinned; no direct exploit found [E0045] | High |
| Grafana 13.1.4 has relevant known vulnerabilities | [E0014], [E0015], [E0016], [E0034] | Non-enabled branches and earlier fixed issues excluded [E0048] | High for affected versions; exploit not independently reproduced |
| Retention can leave physical chunks beyond 30 days | [E0023], [E0024], [E0053] | Flushed data persists; orphan chunks need not be query-visible; S3 markers/lifecycle mitigate [E0054] | High for marker durability; exact residue depends on restart timing |
| Functional CI is ineffective | [E0025], [E0055], [E0056] | GitGuardian and PR #1 approval exist [E0057] | High |
| Public push route lacks a generic bypass | [E0028], [E0029], [E0030] | Direct private-peer spoofing and trailing-`*` token caveat remain [E0058] | High for ordinary tokens on the normal router |
| Loki gRPC dependencies are vulnerable | [E0032], [E0059], [E0060], [E0061] | No public gRPC route; xDS/SSH-specific findings excluded [E0062] | High for affected code; exploit requires reachability |
| Caddy has a conditional CONNECT-poisoning path | [E0033], [E0064] | Precise sequence required and not tested [E0065] | High for runtime presence; exploitability unknown |
| Dependabot alerts do not describe the deployed binaries | [E0026], [E0076] | Alerting itself is active; provenance needs cleanup | High for absent-manifest fact; alert origin unknown |

## Evidence table

| ID | Source and exact locator | Use |
|---|---|---|
| E0001 | `Dockerfile:2-6`; `bin/env.sh:13-45`; `README.MD:95-104` at commit c21aaee | Secret material copied and expanded into images |
| E0002 | Docker, **Build secrets**, opening and secret mounts | Independent remediation guidance |
| E0003-E0005 | `bin/run:29-35`; Scalingo shared PostgreSQL **Connecting**; PostgreSQL **SSL Support**, Table 32.1 | TLS downgrade, affected addon, and protection semantics |
| E0006-E0007 | `conf/Caddyfile:29-31`; `bin/run`; Grafana v13.1.4 `conf/defaults.ini:393-478` | Public Grafana route and inherited security defaults |
| E0008-E0011 | `conf/loki-config.yaml:1-5`; Loki v3.7.7 configuration; Scalingo Private Networks; Go `net.Listen` | Loki internal listener exposure |
| E0012-E0013 | `bin/compile:26-62`; Grafana/Loki/Caddy release artifacts | Missing verification and available checksums/signatures/SBOM |
| E0014-E0017 | Grafana release notes and official advisories for CVE-2026-76154, CVE-2026-88858, CVE-2026-15815 | Update gap and three principal vulnerabilities |
| E0018 | Caddy GHSA-8qx7-j2v2-x6q5, affected `<=2.11.4` | WebSocket header leakage |
| E0019-E0021 | Scalingo filesystem/backup docs; local persistence config; Loki WAL defaults and 3.7.9 notes | Ephemeral state, managed-backup counterevidence, WAL risk |
| E0022 | Loki v3.7.7 `vendor/.../dskit/server/server.go:191-210,309-336,677-683` | Empty addresses and ephemeral gRPC listener |
| E0023-E0024 | Local compactor config; Loki v3.7.7 `retention.md:43-67` | Ephemeral deletion markers and S3 alternative |
| E0025-E0027 | CodeQL runs and logs; branch/rules APIs; Dependabot API; repository tree; GitHub licensing guide | CI, alert-noise, and governance debt |
| E0028-E0030 | Local Caddyfile; Caddy matchers; Scalingo routing/headers | Public-route counterevidence |
| E0031 | Grafana defaults and local startup settings | Default key, cookie, CSP, and metrics posture |
| E0032 | GHSA-vp52-pcj8-j9qc and Loki `go.mod` | gRPC fragmentation affected/fixed versions |
| E0033 | GO-2026-6605 and Caddy SBOM `goCompiledVersion=go1.26.3` | CONNECT-poisoning runtime presence |
| E0034 | Grafana advisories CVE-2026-13719, CVE-2026-13720, CVE-2026-81841, CVE-2026-33374, CVE-2026-100105 | Additional relevant dashboard/alerting fixes |
| E0035-E0036 | Scalingo Force HTTPS; Compose file | HTTP default and local database exposure |
| E0037 | Loki build workflow and Go advisories | Go 1.26.5 -> 1.26.10 build-provenance and fixes |
| E0038-E0045 | ANSSI DevSecOps PDF; local counterevidence; Caddy signature procedure | Secret-management standard, scope limits, and stronger verification |
| E0046-E0051 | GitHub Advisory/OSV mirrors; local feature review; Caddy impact constraints | Derivative corroboration, exclusions, and conditional impact |
| E0052-E0057 | Scalingo backups/filesystem; Loki retention lifecycle; local CI tree; ANSSI CI text; PR checks | Backup counterevidence, retention remediation, and CI status |
| E0058 | Caddy header matcher and Scalingo headers | Direct-peer and token-wildcard caveats |
| E0059-E0065 | gRPC/Go advisories; Loki build workflows; Caddy SBOM and transport path | Dependency reachability and runtime confirmation |
| E0066-E0072 | Scalingo Force HTTPS; Grafana cookie defaults; Compose file/docs; Postgres official-image docs | Transport hardening and Compose-only exposure |
| E0073-E0074 | OSV no-fixed entry versus Grafana CVE-2026-42127 fixed `>=13.0.2` | Counterexample proving why scanner results need filtering |
| E0075-E0076 | Scalingo v11 registry record; live Dependabot alert aggregation | Immutable base availability and 71 absent-manifest alerts |

## Contradictions, rejected hypotheses, and uncertainty

- **Public Loki bypass:** rejected for the normal Scalingo route. Direct internal listeners remain a separate finding [E0043].
- **All 71 Dependabot alerts indicate deployed vulnerabilities:** rejected because every alert references a `go.mod` absent from current and fetched branch history [E0026], [E0076].
- **Broad OSV results are sufficient proof:** rejected. One entry lacked a fixed event while Grafana explicitly fixed the issue in 13.0.2, demonstrating overmatching [E0073], [E0074].
- **No backups exist:** rejected for PostgreSQL when the managed addon is used. Restore testing and S3 recovery remain unverified [E0052].
- **Shutdown always waits only for Caddy:** too broad. Normal Caddy exit reaches the later `wait` for all children; error/timeout paths can still bypass complete reaping, and no drain/deadline exists.
- **Every newer CVE applies:** rejected for non-enabled SAML/SCIM, Auth Proxy, OAuth, Azure Monitor, SQL datasource, Git Sync, FastCGI, file-server, xDS, and SSH branches [E0048].
- **Current production uses default admin credentials or shares secret-bearing images:** unknown. Runtime secrets, dashboards, plugins, user roles, and actual images were not inspected.
- **Current production network exposure:** unknown beyond source-established listener behavior. Private-network policy, SSRF, port publication, and firewall state were not tested.
- **Base-image package vulnerabilities:** exact deployed Scalingo and Postgres digests are unknown, so no package-level CVE verdict is made. No application path to the Scalingo Brotli issue was established.
- **Dynamic verification:** no repository code, stack, image, exploit, or live production probe was executed. Shell syntax and ShellCheck were the only local static checks; syntax passed and ShellCheck reported one unused `KEYCLOAK_PATH` plus informational trap warnings.

## Method appendix

- **Effort:** exhaustive source-backed review, stopped after 16 meaningful hops when all high-impact claims had primary locators, counterevidence, and explicit preconditions.
- **Source classes:** tracked source/history, GitHub live repository APIs, official platform documentation, versioned upstream source/configuration, first-party and independent advisory databases, release workflows, CycloneDX SBOM, registry metadata, and ANSSI standards.
- **Review tracks:** deployment security, dependency/lifecycle research, and maintainability/technical debt, followed by independent parent verification of every included finding.
- **Verification:** exact commit and line locators; fixed-version tables; release build metadata; route/trust-boundary analysis; adversarial checks for false positives and non-applicable branches; managed-backup and GitGuardian counterevidence.
- **Ledger:** 76 evidence entries across seven source types; final lint reports no errors and no unresolved content warnings.
- **Freshness:** upstream releases and GitHub settings checked on 2026-11-05, including Caddy 2.11.9 released that day.

## Suggested verification before production closure

1. Confirm the deployed commit, image digests, Go/runtime versions, replica count, effective listeners, and absence of secret-bearing image layers/log history.
2. Verify Force HTTPS, PostgreSQL TLS, MFA/SSO, admin state, Grafana key management, network policy, S3 bucket access/lifecycle/versioning, and backup settings without exposing their values.
3. Run staging tests for allowed/denied ingestion, private-port isolation, startup/readiness, graceful termination, WAL recovery, 30-day physical deletion, PostgreSQL PiTR, and clean-environment recovery.
4. Re-run dependency and image scans against the final SBOM, then require the new functional checks and review on a protected `main`.

👾 Generated with [Letta Code](https://letta.com)
