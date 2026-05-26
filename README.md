# fleet_test — Fleet PKI mailbox + secrets test fixture

본 repo는 moum-ops Fleet PKI sprint (ADR-012)의 *mailbox 추상화* 를
지원하는 GitOps 디렉토리 표준을 제공한다. moumd 데몬과 moum-ops 앱이
이 repo를 *shared medium* 으로 사용해 enrollment / renewal / revocation
을 비동기 교환한다 (HTTP `/enroll` RPC 는 의도적으로 채택 안 됨).

## 디렉토리 구조

```
_fleet/                              fleet-wide 자료 (모든 데몬 read-only)
├── ca-chain.pem                     Fleet CA chain (Phase G operator wizard 가 생성, D9-1)
├── secrets.enc.yaml                 fleet-wide secrets (SOPS encrypted, multi-recipient)
├── software.yaml                    3-tier cascade fleet baseline
└── playbooks/                       Ansible playbooks

_bootstrap/                          신규 데몬 부트스트랩 자료 (V3-β identity bundle)
├── <short_id>-identity.enc.yaml     daemon age key wrapped to common-pub (SOPS)
└── <short_id>-ticket.enc.yaml       HMAC-signed bundle ticket (SOPS)

enrollments/<short_id>/              데몬 enrollment mailbox 슬롯
├── pending.csr                      daemon writes (fleet_cert::generate_enrollment_csr)
├── meta.json                        daemon writes (EnrollmentMeta + SPIFFE URI)
└── signed.pem                       operator writes (moum-ops app via SigningService)

renewals/<short_id>/                 데몬 cert 갱신 mailbox 슬롯 (50% TTL)
├── current.pem                      daemon writes
├── new.csr                          daemon writes
├── meta.json                        daemon writes
└── renewed.pem                      operator writes

revocations/<short_id>.json          데몬 폐기 슬롯 (operator writes)

<short_id>/                          per-daemon 영구 자료
├── daemon.yaml                      daemon config
├── membership.yaml                  fleet 멤버십 (age_public_key 등)
└── secrets.enc.yaml                 daemon-specific secrets (SOPS, single-recipient)

.sops.yaml                           SOPS creation_rules (multi + single recipient)
index.jsonl                          데몬 인벤토리 (one-line-per-daemon)
```

## SPIFFE URI 형식 (ADR-012 D1+D2+D3)

- **daemon**: `spiffe://moum-ops/fleet/daemon/<short_id>`
- **operator**: `spiffe://moum-ops/operator/<user_id>`
- **trust domain**: `spiffe://moum-ops/` (단일)

## 워크플로 (ADR-012 §3)

### 1. 신규 데몬 부트스트랩 (Phase B V3-β)

```
operator (moum-ops app)
  ├─ forge_daemon_identity(short_id, common_priv)
  │     → _bootstrap/<short_id>-identity.enc.yaml (SOPS, recipient=common-pub)
  ├─ forge bundle_ticket (HMAC + fleet HMAC key)
  │     → _bootstrap/<short_id>-ticket.enc.yaml
  └─ git commit + push

daemon
  ├─ git pull
  └─ ingest_bootstrap_bundle(config_dir, fleet_repo, short_id, common_priv)
        → {config_dir}/age/keys.txt
```

### 2. Enrollment (Phase A)

```
daemon
  ├─ generate_enrollment_csr(config_dir, spiffe_id, fleet_ca_chain)
  │     → {config_dir}/fleet/{daemon.key (0600), daemon.csr}
  ├─ submit_csr_to_mailbox()
  │     → enrollments/<short_id>/{pending.csr, meta.json}
  ├─ git commit + push
  └─ watch_for_cert_and_install (30s polling, 1h timeout)

operator (moum-ops app)
  ├─ SigningService::unlock(passphrase)  ← 8h session + 30min idle lock
  ├─ scan_pending() → 발견된 enrollments
  ├─ LocalCaIssuer::sign_daemon_leaf(intermediate, request)
  ├─ deliver_cert()
  │     → enrollments/<short_id>/signed.pem
  └─ git commit + push

daemon
  └─ signed.pem 도착 감지 → install_cert → {config_dir}/fleet/daemon.crt
```

### 3. Renewal (Phase E, 50% TTL)

Enrollment와 동일 흐름, `renewals/<short_id>/` 슬롯 사용 (mtls
`FilesystemMailbox::scan_pending` 이 enrollments + renewals 둘 다 처리).

### 4. Revocation (Phase E)

```
operator → revocations/<short_id>.json commit + push
daemon → is_revoked(fleet_repo, short_id) 체크 → 다음 reconcile 시 폐기 인지
runtime mTLS handshake → verifier::ChainValidator 가 자동 차단
```

## 권위 참조

- [ADR-012: Fleet PKI Runtime mTLS + Capability-Based Authz](https://github.com/aileaf-co/moum-ops/blob/main/docs/architecture/adr/ADR-012-fleet-pki-runtime-mtls.md)
- [FLEET_PKI_MAILBOX_DESIGN.md](https://github.com/aileaf-co/moum-ops/blob/main/docs/architecture/FLEET_PKI_MAILBOX_DESIGN.md) — *(mailbox 패턴의 디자인 권위, "HTTP /enroll endpoint rejected" 명시)*
- [SPIFFE_MTLS_INTEGRATION_PLAN.md](https://github.com/aileaf-co/moum-ops/blob/main/docs/plans/SPIFFE_MTLS_INTEGRATION_PLAN.md) — sprint plan + Decision Log
- [moum-ops PR #46](https://github.com/aileaf-co/moum-ops/pull/46) — Fleet PKI daemon-side wiring

## 보안 정책

**Repo 에 절대 들어가지 않아야 할 자료** (`.gitignore` 강제):
- 데몬 private key (`daemon.key`, `*.priv`, `*.age`)
- Master Key 평문 또는 wrapped (`.fallback-dek`, `master.key.enc`, `master.key.passphrase`)
- 운영자 cert private key
- BIP39 recovery phrase (사용자 오프라인 백업만)
- 일반 시스템 부산물 (`.DS_Store`, `*.tmp`, `*.bak`)

**Repo 가 보유해도 안전한 자료**:
- 데몬 public age key (`membership.yaml` 안)
- Fleet CA chain (`_fleet/ca-chain.pem` — trust anchor)
- 서명된 cert (`signed.pem`, `renewed.pem` — leaf, 공개)
- CSR (`pending.csr`, `new.csr` — public)
- SOPS-encrypted bundles (`*-identity.enc.yaml`, `*-ticket.enc.yaml`, `secrets.enc.yaml`)
- Revocation list (`revocations/<id>.json` — 공개)
