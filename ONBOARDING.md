# Boulder Onboarding Guide

Generated from `.understand-anything/knowledge-graph.json` (commit [`3647aeed`](https://github.com/letsencrypt/boulder/commit/3647aeed1eb896ea0d96e645d9e54b22c33373d7), analyzed 2026-05-04).

## Project Overview

- Name: boulder
- Description: Boulder is the software that runs Let's Encrypt — an implementation of an ACME-based Certificate Authority. It is composed of gRPC-connected services (WFE, RA, VA, CA, SA, Publisher, CRL Updater) that together validate domain control and issue X.509 certificates.
- Languages: go, sql, protobuf, yaml, javascript, html, css, shell, makefile, markdown, json
- Frameworks/libraries: gRPC, Go modules, MySQL driver, `borp`, `miekg/dns`, `miekg/pkcs11`, `eggsampler/acme`, `zlint`, `go-redis`, Prometheus, OpenTelemetry, AWS SDK v2, Docker Compose, GitHub Actions

The codebase has >100 source files. For narrow tasks, scope reading to a single service subdirectory.

## Architecture Layers

Boulder is a set of gRPC microservices split by security zone. Internet-facing binaries (WFE, VA, Publisher, CRL Storer) are deliberately isolated from the signing path (CA) by a gRPC-only boundary, and all durable state flows through the Storage Authority.

### 1. Web Frontend — `wfe2/`, `sfe/`, `web/`
Public ACME HTTP surface and subscriber self-service UI. Terminates subscriber traffic, verifies JWS, and forwards ACME operations to the RA.

Key files: `wfe2/wfe.go`, `wfe2/verify.go`, `wfe2/cache.go`, `wfe2/stats.go`, `web/context.go`, `web/probs.go`, `web/send_error.go`, `web/server.go`, `sfe/sfe.go`, `sfe/overrides.go`, `sfe/overridesimporter.go`, `sfe/zendesk/zendesk.go`.

### 2. Orchestration (RA) — `ra/`
The Registration Authority coordinates ACME operations: asks SA to create orders, VA to validate, CA to sign, Publisher to log, SA to persist. Enforces rate limits and policy before issuance.

Key files: `ra/ra.go`, `ra/proto/ra.proto`.

### 3. Identifier Validation — `va/`, `bdns/`, `iana/`
Executes ACME challenges (HTTP-01, DNS-01, DNS-ACCOUNT-01, DNS-PERSIST-01, TLS-ALPN-01) and CAA rechecks. Runs Multi-Perspective Issuance Corroboration (MPIC) against remote-VA perspectives, and enforces anti-SSRF/IANA reserved-address rules.

Key files: `va/va.go`, `va/http.go`, `va/dns.go`, `va/dns_persist.go`, `va/caa.go`, `va/tlsalpn.go`, `va/utf8filter.go`, `va/config/config.go`, `bdns/dns.go`, `bdns/servers.go`, `bdns/problem.go`, `bdns/mocks.go`, `iana/iana.go`, `iana/ip.go`, `va/proto/va.proto`.

### 4. Certificate Issuance — `ca/`, `issuance/`, `policy/`, `linter/`, `ctpolicy/`, `goodkey/`, `precert/`, `csr/`
Signs certificates and CRLs. Enforces Policy Authority checks, key-quality rules, and a zlint-backed pre-flight linter. Nothing reaches an HSM without clearing the linter.

Key files: `ca/ca.go`, `ca/crl.go`, `ca/proto/ca.proto`, `issuance/issuer.go`, `issuance/cert.go`, `issuance/crl.go`, `policy/pa.go`, `linter/linter.go`, plus per-lint files under `linter/lints/{cabf_br,chrome,cpcps,rfc}/`, `ctpolicy/ctpolicy.go`, `ctpolicy/loglist/loglist.go`, `ctpolicy/loglist/lintlist.go`, `goodkey/good_key.go`, `goodkey/sagoodkey/good_key.go`, `precert/corr.go`, `csr/csr.go`.

### 5. Storage (SA + DB) — `sa/`, `db/`
The single source of truth for durable state. All other services reach the database only through the SA's read-write (`sa.go`) and read-only (`saro.go`) gRPC services.

Key files: `sa/sa.go`, `sa/saro.go`, `sa/model.go`, `sa/database.go`, `sa/metrics.go`, `sa/sysvars.go`, `sa/type-converter.go`, `sa/proto/sa.proto`, `sa/proto/sadb.proto`, `sa/db/01-boulder_sa.sql`, `sa/db/01-boulder_sa_next.sql`, `sa/db/01-incidents_sa.sql`, `sa/vtschema/*/vschema.json`, `db/gorm.go`, `db/interfaces.go`, `db/map.go`, `db/multi.go`, `db/qmarks.go`, `db/rollback.go`, `db/transaction.go`.

Principal tables (defined in `sa/db/01-boulder_sa.sql`): `registrations`, `authz2`, `orders`, `orderFqdnSets`, `replacementOrders`, `certificates`, `precertificates`, `certificateStatus`, `serials`, `issuedNames`, `fqdnSets`, `keyHashToSerial`, `blockedKeys`, `revokedCertificates`, `crls`, `crlShards`, `overrides`, `paused`, `incidents`.

### 6. Publication and Revocation — `publisher/`, `crl/`
Submits precertificates to CT logs for SCTs, and runs the sharded-CRL pipeline.

Key files: `publisher/publisher.go`, `publisher/proto/publisher.proto`, `crl/crl.go`, `crl/idp/idp.go`, `crl/checker/checker.go`, `crl/updater/updater.go`, `crl/updater/batch.go`, `crl/updater/continuous.go`, `crl/storer/storer.go`, `crl/storer/proto/storer.proto`.

### 7. Supporting Services — `nonce/`, `ratelimits/`, `observer/`, `unpause/`, `salesforce/`, `allowlist/`, `revocation/`
Cross-cutting infrastructure: ACME replay-nonce minting, Redis-backed GCRA rate limits, synthetic probers, subscriber pause-lift flows, and Pardot/Salesforce integration.

Key files: `nonce/nonce.go`, `nonce/proto/nonce.proto`, `ratelimits/limiter.go`, `ratelimits/gcra.go`, `ratelimits/limit.go`, `ratelimits/transaction.go`, `ratelimits/source.go`, `ratelimits/source_redis.go`, `ratelimits/names.go`, `ratelimits/utilities.go`, `observer/observer.go`, `observer/monitor.go`, `observer/mon_conf.go`, `observer/obs_conf.go`, `observer/obsclient/obsclient.go`, per-prober files under `observer/probers/{aia,ccadb,crl,dns,http,tls,mock}/`, `unpause/unpause.go`, `salesforce/exporter.go`, `salesforce/pardot.go`, `salesforce/cache.go`, `allowlist/main.go`, `revocation/reasons.go`.

### 8. Binary Entry Points — `cmd/`
One directory per service, each registering itself with `cmd.RegisterCommand` during `init()`. A unified `boulder` binary dispatches by argv.

Service binaries: `cmd/boulder-wfe2/`, `cmd/boulder-ra/`, `cmd/boulder-va/`, `cmd/boulder-ca/`, `cmd/boulder-sa/`, `cmd/boulder-publisher/`, `cmd/boulder-observer/`, `cmd/nonce-service/`, `cmd/sfe/`, `cmd/remoteva/`, `cmd/crl-updater/`, `cmd/crl-storer/`, `cmd/crl-checker/`, `cmd/email-exporter/`.

Operator/admin tools: `cmd/admin/` (with subcommands `cert.go`, `key.go`, `dryrun.go`, `overrides_*.go`, `pause_identifier.go`, `unpause_account.go`), `cmd/ceremony/` (offline HSM key-ceremony tool: `cert.go`, `crl.go`, `ecdsa.go`, `rsa.go`, `file.go`, `key.go`, `main.go`), `cmd/bad-key-revoker/`, `cmd/cert-checker/`, `cmd/log-validator/`, `cmd/reversed-hostname-checker/`.

Shared shell: `cmd/boulder/main.go`, `cmd/registry.go`, `cmd/shell.go`, `cmd/config.go`.

### 9. Shared Libraries — `core/`, `grpc/`, `log/`, `metrics/`, `errors/`, `probs/`, `identifier/`, `features/`, `config/`, `redis/`, `must/`, `strictyaml/`, `pkcs11helpers/`, `privatekey/`
High-fan-in foundation used by every Boulder service.

Key files: `core/objects.go`, `core/interfaces.go`, `core/challenges.go`, `core/util.go`, `core/proto/core.proto`, `grpc/client.go`, `grpc/server.go`, `grpc/interceptors.go`, `grpc/errors.go`, `grpc/pb-marshalling.go`, `grpc/skew.go`, `grpc/creds/creds.go`, `grpc/noncebalancer/noncebalancer.go`, `grpc/internal/resolver/dns/dns_resolver.go`, `log/log.go`, `log/mock.go`, `log/validator/validator.go`, `log/validator/tail_logger.go`, `metrics/scope.go`, `metrics/measured_http/http.go`, `errors/errors.go`, `probs/probs.go`, `identifier/identifier.go`, `features/features.go`, `config/duration.go`, `redis/config.go`, `redis/lookup.go`, `redis/metrics.go`, `strictyaml/yaml.go`, `must/must.go`, `pkcs11helpers/helpers.go`, `privatekey/privatekey.go`.

### 10. Infrastructure and Ops — top-level files, `docs/`, `.github/`, `data/`
Build, CI, local dev, operator docs, email templates.

Key files: `README.md`, `Makefile`, `Containerfile`, `docker-compose.yml`, `docker-compose.next.yml`, `go.mod`, `data/production-email.template`, `data/staging-email.template`, docs under `docs/` (notably `DESIGN.md`, `ISSUANCE-CYCLE.md`, `CRLS.md`, `multi-va.md`, `acme-divergences.md`, `acme-implementation_details.md`, `config-validation.md`, `error-handling.md`, `health.md`, `logging.md`, `redis.md`, `release.md`), workflows under `.github/workflows/` (`boulder-ci.yml`, `release.yml`, `codeql.yml`, `cps-review.yml`, `check-iana-registries.yml`, `issue-for-sre-handoff.yml`, `merged-to-main-or-release-branch.yml`, `try-release.yml`).

## Key Concepts

- Unified binary, BusyBox-style. Every `cmd/<service>/main.go` registers itself via `cmd.RegisterCommand` in `init()`. `cmd/boulder/main.go` imports them all and dispatches on `argv[0]` or `argv[1]`. The shared lifecycle (config load, logger/metrics/tracing setup, signal handling) lives in `cmd/shell.go`; typed config structs in `cmd/config.go`.
- gRPC-first service boundary. Every inter-service call goes through a `.proto` contract (`ra/proto/ra.proto`, `sa/proto/sa.proto`, `va/proto/va.proto`, `ca/proto/ca.proto`, `publisher/proto/publisher.proto`, `nonce/proto/nonce.proto`, `crl/storer/proto/storer.proto`, `salesforce/email/proto/emailexporter.proto`). Generated `.pb.go` is intentionally excluded from the knowledge graph; read the `.proto` file to see the real API surface. Hand-written converters between domain types and protobufs live in `grpc/pb-marshalling.go`.
- Interfaces as service contracts. `core/interfaces.go` declares the Go interfaces the RA depends on (CertificateAuthority, ValidationAuthority, StorageAuthority, Publisher, etc.). In production those are gRPC clients; in unit tests they are mocks; in the dev binary they are in-process. Read `core/objects.go` for the canonical ACME domain types (Registration, Authorization, Challenge, Order, Certificate).
- Security zone split. WFE, VA, Publisher, CRL Storer are internet-adjacent; RA, CA, SA are internal. The CA never talks to the internet — it only accepts gRPC from the RA. The SA is the only process allowed to open a DB connection.
- Durable state is exactly five ACME object types. Accounts, authorizations, challenges, orders, certificates. Everything else in `sa/db/01-boulder_sa.sql` is bookkeeping around those.
- Precertificate + SCT flow. The CA first signs a precert with the CT poison extension; the Publisher submits it to enough CT logs to reach an SCT quorum per `ctpolicy/ctpolicy.go`; only then does the CA issue a final certificate with SCTs embedded. `precert/corr.go` enforces the precert↔final correspondence invariant.
- Pre-issuance linting. `linter/linter.go` wraps zlint, registers Boulder-specific lints under `linter/lints/{cabf_br,chrome,cpcps,rfc}`, builds a throwaway signed cert/CRL with a matching ephemeral key, and lints that before the real HSM signature commits. Failure aborts issuance.
- MPIC (Multi-Perspective Issuance Corroboration). `va/va.go` fans DCV and CAA out to remote VA perspectives in diverse networks (and RIR-diverse buckets), requires quorum agreement, and exposes a shadow/experimental-VA mode for safe rollouts. Defends against localized BGP hijacks of the validation network path.
- Sharded CRLs. `crl/updater/updater.go` leases shards from the SA, streams revoked serials per shard, asks the CA's `ca/crl.go` + `issuance/crl.go` to sign each shard, then hands the DER to `crl/storer/storer.go` for S3 upload. Shard assignment is stable so a given cert's AIA CRL URL never changes.
- GCRA-based rate limits. `ratelimits/gcra.go` implements the Generic Cell Rate Algorithm; `ratelimits/limiter.go` wraps it with Redis (`ratelimits/source_redis.go`) or in-memory (`ratelimits/source.go`) storage. The WFE and RA call into it before expensive operations.
- JWS-verified ACME requests. `wfe2/verify.go` parses the flattened JWS on every authenticated request, validates the nonce (via the `nonce/` service with its per-instance prefix and the `grpc/noncebalancer/` prefix-routed balancer), enforces supported algorithms, verifies the `url` header matches the request URL, and handles the double-JWS dance for account-key rollover.
- Structured audit logging. `log/log.go` defines `AuditInfo`/`AuditErr` entrypoints whose output includes a checksum; `cmd/log-validator/main.go` tails those lines and re-verifies each checksum so Splunk/WebTrust pipelines can prove no audit entries were lost or tampered with.
- Feature flags. `features/features.go` is a global, mutex-guarded bool table loaded from config at startup. New code ships dark and is enabled per-deployment without a rebuild.

## Guided Tour

The tour is the recommended reading order for a first pass through Boulder.

1. Project overview and component map — `README.md`. Seven-service architecture; subscriber → WFE → RA → {VA, SA, CA} → Publisher/CRL; split along security-trust boundaries.
2. Unified boulder binary and command dispatch — `cmd/boulder/main.go`, `cmd/registry.go`, `cmd/shell.go`, `cmd/config.go`. BusyBox-style dispatch; `init()`-time registration; shared lifecycle.
3. Core domain types shared across every service — `core/objects.go`, `core/interfaces.go`, `core/challenges.go`. Registration/Authorization/Challenge/Order/Certificate plus the interfaces each service must satisfy.
4. Web Front End — the ACME HTTP surface — `wfe2/wfe.go`, `wfe2/verify.go`, `web/context.go`, `web/probs.go`, `web/send_error.go`. Every ACME endpoint; JWS verification; problem-document rendering.
5. Registration Authority — the orchestrator — `ra/ra.go`, `ra/proto/ra.proto`. Drives the full ACME flow; rate-limit and policy enforcement point.
6. Storage Authority — the durable state boundary — `sa/sa.go`, `sa/saro.go`, `sa/proto/sa.proto`, `sa/db/01-boulder_sa.sql`. The only DB consumer; read-write and read-only gRPC surfaces.
7. Validation Authority — challenges and MPIC — `va/va.go`, `va/http.go`, `va/dns.go`, `bdns/dns.go`. Challenge execution, CAA, DNSSEC, MPIC quorum, anti-rebind protections.
8. Certificate Authority — signing, policy, and linting — `ca/ca.go`, `issuance/issuer.go`, `issuance/cert.go`, `policy/pa.go`, `linter/linter.go`. HSM-backed signing; Policy Authority; pre-flight zlint gate.
9. Publisher and Certificate Transparency — `publisher/publisher.go`, `publisher/proto/publisher.proto`. Precert submission, SCT quorum, final-cert emission.
10. Revocation and the CRL pipeline — `crl/updater/updater.go`, `crl/storer/storer.go`, `issuance/crl.go`. Sharded CRL generation, signing, upload.
11. Rate limits, nonces, and feature flags — `ratelimits/limiter.go`, `ratelimits/gcra.go`, `nonce/nonce.go`, `features/features.go`. Redis GCRA limiter; replay-nonce service; dark-launch feature gating.
12. How Boulder builds, tests, and runs — `docker-compose.yml`, `Makefile`, `.github/workflows/boulder-ci.yml`. Local dev stack (MariaDB, Redis, mock DNS, all-in-one boulder); build/test entry points; upstream CI.

## File Map (key files by layer)

Web Frontend
- `wfe2/wfe.go` — ACME v2 HTTP server: directory, newNonce, newAccount, newOrder, newAuthz, authz, challenge, finalize, cert, revoke-cert, key-change, renewalInfo, CORS, ARI replacement, pause/unpause, rate limiting.
- `wfe2/verify.go` — RFC 8555 JWS verification: flattened parse, algorithm allow-list, nonce check, `url`-header match, account key resolution, double-JWS rollover.
- `wfe2/cache.go` — In-process TTL cache wrapping `SA.GetRegistration` so WFE2 does not hit the DB on every request.
- `wfe2/stats.go` — WFE2 Prometheus metrics.
- `web/context.go` — `RequestEvent` + `TopHandler` middleware capturing timing, status, real IP, UA, and structured audit log entries.
- `web/probs.go` — Maps `berrors` → RFC 8555 ACME problem documents with correct `urn:ietf:params:acme:error:*` types and HTTP statuses.
- `web/send_error.go` — Writes an ACME problem+json error, logging internal vs user-visible errors distinctly.
- `sfe/sfe.go` — Subscriber Front End HTTP service: `/unpause` JWT flows, rate-limit override self-service.
- `sfe/overrides.go` / `sfe/overridesimporter.go` — Override request forms and Zendesk-driven importer.
- `sfe/zendesk/zendesk.go` — Thin Zendesk REST client used by SFE.
- `unpause/unpause.go` — Shared JWT helper (HS256) for unpause links issued by WFE2 and redeemed by SFE.

Orchestration (RA)
- `ra/ra.go` — RA implementation: orchestrates ACME issuance end to end, account lifecycle, orders, challenges, finalize, revocation, admin actions.
- `ra/proto/ra.proto` — Canonical RA gRPC service contract.

Validation
- `va/va.go` — `ValidationAuthorityImpl`, MPIC fan-out with quorum + RIR diversity, experimental-VA shadow mode, challenge dispatcher, `DoDCV` gRPC entry point.
- `va/http.go` — HTTP-01: pre-resolved-IP dialer, IPv6-then-IPv4 fallback, redirect target validation, body length limits, anti-SSRF reserved-IP checks.
- `va/dns.go` — DNS-01 and DNS-ACCOUNT-01 validation; A/AAAA resolution for HTTP-01/TLS-ALPN-01 with IANA-reserved address filtering.
- `va/dns_persist.go` — Draft DNS-PERSIST-01 challenge (long-lived `_acme-persist` TXT record).
- `va/caa.go` — CAA rechecking: `DoCAA` gRPC, recursive tree-climb, record parsing, MPIC-aware fan-out.
- `va/tlsalpn.go` — TLS-ALPN-01 per RFC 8737: `acme-tls/1` ALPN dial, self-signed cert with `id-pe-acmeIdentifier` extension check.
- `va/utf8filter.go` — UTF-8-safe response handling helpers.
- `va/config/config.go` — VA config structs.
- `bdns/dns.go` — Boulder's DNS resolver: `miekg/dns` wrapper with retries, DoH, Prometheus metrics, CAA/DNSSEC handling.
- `bdns/servers.go` — Resolver address pools; static and dynamic (SRV-refresh) `ServerProvider`.
- `bdns/problem.go` / `bdns/mocks.go` — DNS error classification; test mocks.
- `iana/iana.go`, `iana/ip.go` — IANA special-use TLD and reserved-IP tables used to reject invalid identifiers.

Issuance
- `ca/ca.go` — CA gRPC service: CSR validation via `goodkey`, profile/serial/SKID selection, precert issuance, structured audit events.
- `ca/crl.go` — `CRLGenerator` gRPC: streams revoked entries, signs each shard via `issuance.GenerateCRL`, streams DER back.
- `issuance/issuer.go` — `Issuer`: intermediate cert + HSM-backed signer, PKCS11/file key loading, `SubjectNameID`/`IssuerNameID` hashes.
- `issuance/cert.go` — `Profile`/`ProfileConfig`, `IssuanceRequest`, `Prepare` (template + lint), `Issue` (final signature), `RequestFromPrecert`.
- `issuance/crl.go` — CRL profile/request, `tbsCertList` build, CRL linter run, issuer signature.
- `policy/pa.go` — Policy Authority: identifier syntax + PSL, exact/wildcard/high-risk/admin-prefix blocklists from hot-reloadable YAML, challenge-type selection per identifier.
- `linter/linter.go` — zlint harness; registers Boulder-specific lints; builds a throwaway signed cert/CRL and lints before the real signer commits.
- `linter/lints/{cabf_br,chrome,cpcps,rfc}/*.go` — Individual Boulder lints enforcing CA/B Forum BR, Chrome profile, and Boulder CP/CPS rules (CRL validity window, IDP extension, no delta CRLs, no AIA on CRLs, reason-code handling, SCTs from same operator, cert/CRL validity bounds, etc.).
- `goodkey/good_key.go` — `KeyPolicy`: RSA/ECDSA curve and modulus rules, small-prime and Fermat close-factor checks, blocked-key enforcement.
- `goodkey/sagoodkey/good_key.go` — SA-backed variant that checks the blockedKeys table.
- `ctpolicy/ctpolicy.go` — CT log submission policy: operator diversity, staggered parallel submissions, SCT collection.
- `ctpolicy/loglist/loglist.go` — Parses the Chrome-style CT log list JSON into Boulder's `Log` structures.
- `ctpolicy/loglist/lintlist.go` — Policy-lint view of the log list.
- `ctpolicy/ctconfig/ctconfig.go` — CT configuration structs.
- `csr/csr.go` — CSR validation helpers shared by WFE/RA/CA.
- `precert/corr.go` — Precert ↔ final cert correspondence checker (audit invariant).

Storage (SA + DB)
- `sa/sa.go` — Read-write SA gRPC service.
- `sa/saro.go` — Read-only SA gRPC service used by components that must not mutate.
- `sa/model.go` — gorp row models and conversions to/from protobuf.
- `sa/database.go` — MariaDB connection setup, required session vars, type-converter registration.
- `sa/metrics.go` — SA Prometheus metrics.
- `sa/sysvars.go` — Required MySQL session variables enforced on connect.
- `sa/type-converter.go` — Boulder ↔ MariaDB type conversions.
- `sa/migrations.sh` — Local schema migration helper.
- `sa/proto/sa.proto`, `sa/proto/sadb.proto` — SA gRPC and internal DB message contracts.
- `sa/proto/subsets.go` — Protobuf-subset helpers.
- `sa/db/01-boulder_sa.sql` — Authoritative production schema.
- `sa/db/01-boulder_sa_next.sql` — Next-schema variant used in staging/dev before promotion.
- `sa/db/01-incidents_sa.sql` / `01-incidents_sa_next.sql` — Incident tables.
- `sa/db/00-create-databases.sql`, `sa/db/02-users.sql`, `sa/db/02-users_next.sql` — Database and role setup.
- `sa/vtschema/*/vschema.json` — Vitess schema configs.
- `db/gorm.go` — Reflection-based struct-to-column mapping.
- `db/map.go` — gorp `DbMap`/transaction wrappers with context and tracing.
- `db/interfaces.go` — DB abstractions.
- `db/multi.go` — Multi-shard helpers.
- `db/qmarks.go` — Query placeholder helpers.
- `db/rollback.go` — `RollbackError` and rollback helper.
- `db/transaction.go` — `WithTransaction` helper.

Publication and Revocation
- `publisher/publisher.go` — Publisher gRPC service: precert submission to CT logs via RFC 6962 `add-pre-chain`, chain building, SCT collection.
- `publisher/proto/publisher.proto` — Publisher gRPC contract.
- `crl/updater/updater.go` — crl-updater core: shard leasing, revoked-serial streaming, CA signing call, storer upload, shard commit.
- `crl/updater/batch.go`, `crl/updater/continuous.go` — One-shot and continuous update modes.
- `crl/storer/storer.go` — crl-storer: validates signed CRL against expected issuer + previous S3 object (monotonic number, IDP URIs), uploads DER.
- `crl/storer/proto/storer.proto` — crl-storer gRPC contract.
- `crl/crl.go` — Shared CRL helpers.
- `crl/idp/idp.go` — Issuing Distribution Point extension helpers.
- `crl/checker/checker.go` — CRL verification helper (used by `cmd/crl-checker`).

Supporting Services
- `nonce/nonce.go` — AES-GCM authenticated replay-nonces keyed by HMAC-derived prefix; sliding-window heap to reject replays; `Nonce`/`Redeem` gRPCs.
- `nonce/proto/nonce.proto` — Nonce service contract.
- `ratelimits/limiter.go` — Top-level limiter: check/spend/refund/batch transactions over Redis or in-memory source.
- `ratelimits/gcra.go` — GCRA core: `maybeSpend`, `maybeRefund`.
- `ratelimits/limit.go`, `ratelimits/names.go`, `ratelimits/transaction.go`, `ratelimits/utilities.go` — Limit catalog, naming, transaction plumbing.
- `ratelimits/source.go`, `ratelimits/source_redis.go` — In-memory and Redis backends.
- `observer/observer.go`, `observer/monitor.go`, `observer/obs_conf.go`, `observer/mon_conf.go` — boulder-observer runtime and config.
- `observer/obsclient/obsclient.go` — Observer client helper.
- `observer/probers/{aia,ccadb,crl,dns,http,tls,mock}/` — Per-protocol synthetic probers; `prober.go` defines the interface.
- `salesforce/exporter.go` — `salesforce.Exporter` gRPC server: bounded LIFO queue + worker pool draining to Pardot with rate limiting.
- `salesforce/pardot.go` — Salesforce Pardot HTTP client (OAuth2, prospect upserts, support Cases).
- `salesforce/cache.go` — Salesforce-side cache.
- `salesforce/email/proto/emailexporter.proto`, `salesforce/proto/exporter.proto` — Email/exporter gRPC contracts.
- `allowlist/main.go` — Generic string-set allowlist (slice or strict-YAML file).
- `revocation/reasons.go` — RFC 5280 CRL reason codes and allowed subsets.

Binary Entry Points (service mains)
- `cmd/boulder-wfe2/main.go` — Dials RA/SA/nonce/email-exporter, loads issuer chains, wires rate-limit Redis, serves ACME.
- `cmd/boulder-ra/main.go` — Wires gRPC clients to VA/CA/SA/Publisher, rate-limit Redis, CT log list, policy.
- `cmd/boulder-va/main.go` — Wires recursive DNS resolvers and remote VA clients; serves VA + CAA gRPC.
- `cmd/boulder-ca/main.go` — Loads issuance profiles + issuer keys, serves CA cert and CRL signing services.
- `cmd/boulder-sa/main.go` — Opens read-only and read-write DB handles, serves SA RW + RO gRPC.
- `cmd/boulder-publisher/main.go` — Loads issuer chains, serves Publisher gRPC.
- `cmd/boulder-observer/main.go` — Parses YAML config, runs synthetic probers.
- `cmd/nonce-service/main.go` — Small gRPC server minting and validating ACME nonces under a per-instance prefix.
- `cmd/sfe/main.go` — SFE HTTP server: unpause flows, rate-limit override import/export.
- `cmd/remoteva/main.go` — Stripped-down VA answering MPIC RPCs from the primary VA.
- `cmd/crl-updater/main.go`, `cmd/crl-storer/main.go`, `cmd/crl-checker/main.go` — CRL pipeline.
- `cmd/email-exporter/main.go` — Subscriber-email forwarder to Pardot.

Operator tools
- `cmd/admin/main.go` + subcommands (`cert.go` revoke-cert, `key.go` block-key, `dryrun.go`, `overrides_*.go`, `pause_identifier.go`, `unpause_account.go`) — Administrative actions against RA/SA with audit logging.
- `cmd/ceremony/main.go` + helpers (`cert.go`, `crl.go`, `ecdsa.go`, `rsa.go`, `file.go`, `key.go`) — Offline HSM key-ceremony tool.
- `cmd/bad-key-revoker/main.go` — Daemon that scans `blockedKeys` and revokes matching unexpired certs via RA.
- `cmd/cert-checker/main.go` — Samples issued certs from SA and reruns policy/lint/BR checks.
- `cmd/log-validator/main.go` — Tails syslog and re-verifies each `LogLineChecksum`.
- `cmd/reversed-hostname-checker/main.go` — Flags reversed-label hostnames in SA.

Shared cmd shell
- `cmd/boulder/main.go` — Unified dispatcher importing every boulder-* service for side-effect registration.
- `cmd/registry.go` — Command table and config-validator registry.
- `cmd/shell.go` — Logger/metrics/tracing setup, panic handling, signal handling, version reporting.
- `cmd/config.go` — Shared config structs (DBConfig, ServiceConfig, TLSConfig, GRPCClientConfig, …) and validators.

Shared Libraries
- `core/objects.go` — Domain model: Registration, Authorization, Challenge, Certificate, CertificateStatus, ValidationRecord, RenewalInfo.
- `core/interfaces.go` — `PolicyAuthority` and other cross-service interfaces.
- `core/challenges.go` — Challenge constructors (HTTP-01, DNS-01, TLS-ALPN-01, DNS-ACCOUNT-01, DNS-PERSIST-01).
- `core/util.go` — Token/serial generation, SHA-256 fingerprints, JWK digests, build metadata, `RetryBackoff`, `UniqueLowerNames`, `HashIdentifiers`, `LoadCert`.
- `core/proto/core.proto` — Core shared protobuf types.
- `grpc/server.go` — Shared gRPC server: mTLS creds, health checks, auth/metric interceptors, graceful stop.
- `grpc/client.go` — Shared gRPC client: TLS creds, resolver/balancer registration, metrics.
- `grpc/interceptors.go` — Deadline/trace/skew metadata, metrics, SAN-to-service allow-list auth, error wrap/unwrap.
- `grpc/creds/creds.go` — mTLS credential loader.
- `grpc/errors.go` — gRPC ↔ Go error conversion.
- `grpc/pb-marshalling.go` — Domain-type ↔ protobuf converters for every inter-service RPC.
- `grpc/skew.go`, `grpc/skew_integration.go` — Clock-skew metadata interceptors.
- `grpc/noncebalancer/noncebalancer.go` — Prefix-based balancer routing `Redeem` to the right nonce-service shard.
- `grpc/internal/resolver/dns/dns_resolver.go` — SRV-aware resolver (`dns`, `nonce-srv`, `nonce-srv-v2` schemes).
- `grpc/generate.go`, `grpc/protogen.sh` — Protobuf generation.
- `identifier/identifier.go` — `ACMEIdentifier` DNS/IP type, proto marshalling, CSR/cert extraction, `Normalize`/`ToValues`.
- `log/log.go` — Structured syslog-flavored logger; `AuditInfo`/`AuditErr` with checksums.
- `log/mock.go` — Test logger.
- `log/prod_prefix.go`, `log/test_prefix.go` — Production vs test log prefixing.
- `log/validator/validator.go` — Re-verifies each `LogLineChecksum` and exports a Prometheus counter.
- `log/validator/tail_logger.go` — hpcloud/tail ↔ Boulder logger adapter.
- `metrics/scope.go` — Prometheus no-op registerer for tests and short-lived commands.
- `metrics/measured_http/http.go` — HTTP handler metrics wrapper.
- `errors/errors.go` — `BoulderError` + `ErrorType` (Malformed, NotFound, RateLimit, CAA, BadPublicKey, …) + gRPC status marshalling.
- `probs/probs.go` — RFC 7807 ACME problem types returned to clients.
- `features/features.go` — Mutex-guarded global feature-flag table.
- `config/duration.go` — `config.Duration` with JSON/YAML (un)marshalling.
- `redis/config.go`, `redis/lookup.go`, `redis/metrics.go` — Redis client setup and metrics.
- `strictyaml/yaml.go` — Strict YAML parser used across Boulder configs.
- `must/must.go` — Panic-on-error helpers for test/setup code.
- `pkcs11helpers/helpers.go` — PKCS#11/HSM helpers: session setup, key discovery, RSA/ECDSA extraction and signing; `x509Signer` adapter and `MockCtx`.
- `privatekey/privatekey.go` — Loads ceremony-generated private keys from PEM/DER and round-trip-verifies them.

Infrastructure and Ops
- `README.md` — Component map, security-zone rationale, quick start.
- `Makefile` — Build targets for admin, boulder, ceremony, chall-test-srv, ct-test-srv, salesforce-test-srv, pardot-test-srv, zendesk-test-srv; `ldflags`-injected BuildID/Host/Time; `.deb`/`.tar` packaging.
- `Containerfile`, `docker-compose.yml`, `docker-compose.next.yml` — Local dev stack (MariaDB, Redis, mock DNS, all-in-one boulder).
- `go.mod` — Module and dependency pin.
- `data/production-email.template`, `data/staging-email.template` — Subscriber email templates.
- `docs/DESIGN.md`, `docs/ISSUANCE-CYCLE.md`, `docs/CRLS.md`, `docs/multi-va.md`, `docs/acme-divergences.md`, `docs/acme-implementation_details.md`, `docs/config-validation.md`, `docs/error-handling.md`, `docs/health.md`, `docs/logging.md`, `docs/redis.md`, `docs/release.md`, `docs/CONTRIBUTING.md`, `docs/CODE_OF_CONDUCT.md` — Architecture and operator reference.
- `.github/workflows/boulder-ci.yml` — Unit tests (multiple Go versions), integration tests via docker-compose, `go generate` freshness check.
- `.github/workflows/release.yml`, `try-release.yml`, `merged-to-main-or-release-branch.yml`, `issue-for-sre-handoff.yml` — Release/handoff workflows.
- `.github/workflows/codeql.yml` — CodeQL scanning.
- `.github/workflows/cps-review.yml`, `check-iana-registries.yml` — Periodic review/reconciliation workflows.

## Complexity Hotspots

These files are flagged `complex` in the knowledge graph. Approach them with the guided tour order in mind, and lean on the surrounding package docs under `docs/` before editing.

Inter-service orchestration and transport
- `wfe2/wfe.go`, `wfe2/verify.go` — Every ACME endpoint and the JWS verification core.
- `web/context.go` — Per-request middleware with audit logging.
- `ra/ra.go` — End-to-end ACME orchestration; rate-limit and policy enforcement.
- `grpc/server.go`, `grpc/interceptors.go`, `grpc/pb-marshalling.go`, `grpc/internal/resolver/dns/dns_resolver.go`, `grpc/noncebalancer/noncebalancer.go` — Service transport plumbing used by every binary.

Validation
- `va/va.go`, `va/http.go`, `va/dns.go`, `va/dns_persist.go`, `va/caa.go`, `va/tlsalpn.go` — Challenge/MPIC/CAA core.
- `bdns/dns.go`, `bdns/servers.go` — DNS resolver and address pools.

Issuance
- `ca/ca.go`, `ca/crl.go` — CA gRPC + CRL generator.
- `issuance/cert.go`, `issuance/issuer.go` — Profile, signing, HSM handle.
- `policy/pa.go` — Policy Authority.
- `linter/linter.go` and representative lints `linter/lints/cpcps/lint_crl_has_idp.go`, `linter/lints/rfc/lint_crl_has_valid_timestamps.go` — Pre-issuance gate.
- `goodkey/good_key.go` — Key quality policy.
- `ctpolicy/ctpolicy.go`, `ctpolicy/loglist/loglist.go` — CT submission + log list parsing.
- `precert/corr.go` — Precert↔final correspondence invariant.

Storage
- `sa/sa.go`, `sa/saro.go`, `sa/model.go` — Storage Authority read-write, read-only, and row-model/protobuf conversion.
- `sa/db/01-boulder_sa.sql`, `sa/db/01-boulder_sa_next.sql` — Production and next schemas.
- `db/gorm.go`, `db/map.go` — DB wrappers.

Publication and Revocation
- `crl/updater/updater.go`, `crl/storer/storer.go` — CRL pipeline.

Supporting Services
- `nonce/nonce.go` — Nonce service.
- `ratelimits/limit.go`, `ratelimits/transaction.go` — Rate-limit catalog and transaction plumbing.
- `salesforce/exporter.go`, `salesforce/pardot.go` — Salesforce integration.
- `sfe/sfe.go`, `sfe/overrides.go`, `sfe/overridesimporter.go`, `sfe/zendesk/zendesk.go` — SFE and Zendesk-driven overrides.
- `observer/probers/ccadb/ccadb.go`, `observer/probers/tls/tls.go` — Non-trivial probers.

Cmd shell and operator tools
- `cmd/shell.go`, `cmd/config.go` — Shared startup and typed config.
- `cmd/boulder-wfe2/main.go`, `cmd/boulder-ra/main.go`, `cmd/boulder-va/main.go`, `cmd/boulder-ca/main.go`, `cmd/sfe/main.go`, `cmd/crl-updater/main.go`, `cmd/ceremony/main.go`, `cmd/cert-checker/main.go`, `cmd/bad-key-revoker/main.go`, `cmd/admin/cert.go`, `cmd/admin/key.go`, `cmd/ceremony/cert.go` — Service wiring and administrative orchestration.

Shared libraries
- `core/objects.go`, `core/util.go`, `identifier/identifier.go`, `log/log.go`, `log/validator/validator.go`, `pkcs11helpers/helpers.go`.
