# SENTRY phase 8 report — S-01…S-09

Date: 2026-09-12 (Asia/Tehran)<br>
Repository: `lqbw9yw8/sni-spoof-new--alpha-3`<br>
Branch: `arena/01a091d3-sni-spoof-new-alpha-3`<br>
Current implementation commits: `969d22d` (`Harden resolver WFP and relay rotation paths`), `37e2b3b` (validated relay parsing/status truth), `ae7d4ea` (rotation-validation regression test), and generated snapshot `42d9f52`<br>
Earlier SENTRY commits retained in history: `7ce50e1`, `a455dcb`, `9c0a0e6`

This is the acceptance record for the remaining SENTRY findings and the
follow-up hardening performed in this checkout. The report distinguishes
source/static evidence from executed tests. A documentation change alone is
never counted as a fix, and no Rust or Windows result is inferred from the
JavaScript suite.

## 1. Finding matrix

| ID | Minimal implementation now in the tree | Exploit/failure path closed | Evidence (`file:line`) | Verification in this checkout | Rollback / recovery | Status |
|---|---|---|---|---|---|---|
| S-01 CI/build verification | CI retains fmt, all-target build/test, clippy, audit/deny and documentation-generation jobs. The generated matrix now reports 42 modules, 448 declared tests and 0 statically dead functions; module docs use `UNTESTED`/`PARTIAL` where the current Rust run is absent. | A static table or a historical count can no longer be presented as a current Rust build result. CI is the required execution boundary. | `.github/workflows/ci.yml:35-91`; `tools/gen_status.py:1-31,193-220`; `TEST_MATRIX.md:4-23` | `python3 tools/gen_status.py --check` passed. `cargo`, `rustc`, `rustup` and `rustfmt` are absent locally; Rust build/test/fmt/clippy/audit/deny are `NOT TESTED`. | Revert `ae7d4ea`, `42d9f52`, `37e2b3b`, and `969d22d` for this follow-up, or revert the complete SENTRY series in reverse dependency order. Never replace generated counts by hand. | `[UNVERIFIED]` runtime / workflow implementation present |
| S-02 trusted DNS rebinding/redirect | `trusted_dns` is rejected fail-closed because a signed WFP redirect callout is not shipped. DoH uses the certificate-bearing hostname `cloudflare-dns.com`; the default resolver pins the reviewed `1.1.1.1`/`1.0.0.1` addresses while preserving the hostname for TLS/SNI. Custom endpoint addresses are resolved once and filtered before `ureq` connects; redirects are disabled. | A legacy IP can no longer look like a working redirect while the user-mode guard only blocks port 53. DoH cannot silently follow a second endpoint or use a resolver result that is loopback/metadata/link-local/etc. | `src/config.rs:654-665,816-818`; `src/doh.rs:17-30,398-459,466-490`; `src/netguard.rs:43-66,192-246`; `src/dns_guard.rs:1-12` | Source-level regression tests and static checks are present, but cargo is unavailable. DoH TLS/network and Windows/BFE execution are `[UNVERIFIED]`. | Keep `trusted_dns` absent. Re-enable redirect only in a future change that ships a signed callout, installer/signature verification, and Windows tests. Revert `ae7d4ea`, `42d9f52`, `37e2b3b`, and `969d22d` to return to the preceding source state. | Implemented fail-closed; redirect/TLS/WFP runtime `[UNVERIFIED]` |
| S-03 driver integrity/pinning | WinDivert DLL/SYS hashes are computed from locked handles; exactly two ordered pins are required; the DLL is loaded by executable-directory absolute path and its loaded module path is compared; `DriverPin` keeps file/module references alive for the backend lifetime. | DLL search-path planting, DLL/SYS swap, post-check replacement and presence-only loading are rejected before capture starts. | `src/engine.rs:470-592,610-666`; `src/integrity.rs:42-86`; `src/main.rs:500-509`; `src/engine_stub.rs:51-55` | No Windows compile, loader, or TOCTOU test is possible locally. `build-windows`/`e2e-windows` evidence is still `[UNVERIFIED]`. | A pin mismatch safely stops startup. Revert only as a coordinated release rollback; do not restore presence-only loading. | Implemented; Windows runtime `[UNVERIFIED]` |
| S-04 fuzz execution | GitHub Actions runs all three fuzz targets with bounded libFuzzer time, non-zero crash status, and corpus/crash/stdout artifacts uploaded even on failure. | A target is not treated as covered merely because its file exists; a crash fails the job and retains the reproducer. | `.github/workflows/fuzz.yml:22-67`; `fuzz/Cargo.toml:1-20`; `fuzz/fuzz_targets/` | Workflow/source was inspected. cargo-fuzz was not available locally; an actual fuzz campaign is `[UNVERIFIED]`. | Fix and retain any reproducer before rerunning; do not delete a failing corpus. | Implemented; CI execution `[UNVERIFIED]` |
| S-05 release signing | Release workflow requires both signing secrets, signs with SHA-256/timestamp, runs `signtool verify /pa /all`, checks `Get-AuthenticodeSignature.Status == Valid`, and refuses staging/publishing unless `SIGNED`. | An unsigned executable cannot pass the release job or be described as signed. | `.github/workflows/release.yml:66-135` | No Windows runner or signing certificate is available locally. Release execution is `[UNVERIFIED]`. | Revoke/replace a bad certificate and rerun the tag; a failed verification publishes nothing. Never bypass the mandatory gate. | Implemented; release run `[UNVERIFIED]` |
| S-06 WinDivert relay E2E | The Windows workflow captures real traffic with `pktmon`, drives a TLS client through the relay to `1.1.1.1:443`, converts ETL to pcapng, and mechanically checks fake SNI first, real SNI later, and repeated pure ACK after fake. The relay's approved `rotate_ips` selection is limited to new relay connections and never rewrites an established transparent flow. | Log-only claims and synthetic tests cannot satisfy the wire acceptance check; wrong ordering, missing duplicate ACK, or an unsafe destination-rotation path fails or remains unverified. | `.github/workflows/e2e.yml:57-153`; `scripts/assert-e2e-pcap.py:63-250`; `src/relay.rs:65-105,280-330,640-665`; `src/main.rs:232-248`; `src/pipeline.rs:519-628` | `python3 -m py_compile scripts/assert-e2e-pcap.py` passed. No Windows runner/WinDivert capture was available; wire result is `[UNVERIFIED]`. | Stop-file plus bounded process cleanup. Revert `ae7d4ea`, `42d9f52`, `37e2b3b`, and `969d22d` to remove the source-level rotation follow-up; preserve the pcap assertions and rerun them on Windows. | Implemented; Windows wire evidence `[UNVERIFIED]` |
| S-07 token leakage | Generated web tokens are emitted only to an interactive stderr. GUI/service/redirected startup with an empty token exits fail-closed; configured tokens are not logged. | `2>log`, service-manager capture, and GUI-child inheritance cannot receive a generated bearer token. | `src/main.rs:879-907`; `src/native_gui.rs:698-703`; `src/webui.rs:208-230,823-854` | `cd uitest && npm test` passed 375/0. Cargo and native token-path runtime are `[UNVERIFIED]`. | Configure a 16+ printable token or disable the web UI; do not redirect a generated token. | Implemented; native runtime `[UNVERIFIED]` |
| S-08 bounded logging/metrics | Rotating JSON-lines observability has a 4 MiB active cap and five backups, with atomic packet, hold, fail-open, injection, relay-fail-closed and capture-error counters. Metrics feed `/api/status` and checkpoint once per second. | The GUI's 500-entry memory cap is no longer the only audit surface; disk growth and mutation effectiveness have bounded machine-readable behavior. | `src/observability.rs:1-316`; `src/fail_open.rs:35-49`; `src/engine.rs:370-395`; `src/main.rs:818-829,1017-1044`; `src/webui.rs:65-176,787-815` | Node/docs checks passed. Rust unit tests and a live rotation test are `[UNVERIFIED]` because cargo is absent. | Delete/rotate `logs/dpi_guard-events.jsonl*` only after preserving evidence. On a read-only directory capture continues and emits a warning. | Implemented; Rust runtime `[UNVERIFIED]` |
| S-09 phase-8 report and status truth | This report records each finding, exact source evidence, executed commands, unavailable environments, rollback and the current status labels. `mobile_gateway` remains `[PARTIAL]`: discovery/reporting only; it does not open a LAN listener, forward traffic or install NAT. The module map no longer uses `DONE` for Rust code whose current tests were not executed. | Prevents historical 431/427/369/344 counts, Linux/JS tests, or a source tag from being converted into a current Windows/Rust `DONE` claim. | `SENTRY_REPORT.md:1-130`; `STATUS.md:1-61,138-160`; `README.md:543-580`; `src/mobile_gateway.rs:1-14`; `tools/status.json` | This report is generated from the current checkout. The UI suite, matrix check, docs lint, Python compile and diff check passed; Rust/Windows/fuzz/signing remain `[UNVERIFIED]`. | Revert `ae7d4ea`, `42d9f52`, `37e2b3b`, and `969d22d` for the current follow-up; retain this report only with a replacement that carries the same evidence and explicit limitations. | Implemented as evidence record; runtime claims remain explicit |

## 2. Follow-up hardening included in `969d22d`, `37e2b3b`, and `ae7d4ea`

These changes are source-level fixes, not a claim that the corresponding
platform has been tested:

1. **DoH TLS identity and resolver TOCTOU:** the default changed from the IP
   literal `https://1.1.1.1/dns-query` to
   `https://cloudflare-dns.com/dns-query`. The resolver pins the reviewed
   Cloudflare IPv4 addresses, so ordinary rustls hostname verification and
   SNI apply without a plaintext bootstrap lookup. Custom endpoint results
   are filtered as the exact `SocketAddr`s handed to ureq. HTTP redirects are
   disabled.
2. **WFP IPv6 condition type:** the IPv6 loopback permit now uses the official
   `FWP_V6_ADDR_MASK` (`0x101`) and `FWP_V6_ADDR_AND_MASK`-compatible storage
   with a `/128` prefix, rather than treating IPv6 as a byte-array condition.
   This is based on the Windows SDK layout, but still needs Windows/BFE
   execution.
3. **Relay `rotate_ips`:** validated config values flow through `RelayId` to
   `RelayTarget`; each newly accepted relay connection chooses the next
   operator-approved IP using `connection::rotate_ip`. The client cannot
   choose a destination or port, and transparent established flows are not
   rewritten.
4. **Honest status labels:** source module comments and the current README
   module map use `UNTESTED`, `PARTIAL`, or `STUB` rather than claiming Rust
   `DONE` while cargo is absent. The generated matrix remains the source of
   counts: 42 modules, 448 declared tests, 0 dead functions, 82 settings
   fields.

## 3. Raw verification output from this checkout

```text
$ (cd uitest && npm test)
# exit 0
110 passed, 0 failed
56 passed, 0 failed
67 passed, 0 failed
60 passed, 0 failed
56 passed, 0 failed
26 passed, 0 failed
# total: 375 passed, 0 failed

$ python3 -m py_compile scripts/assert-e2e-pcap.py
# exit 0

$ python3 tools/gen_status.py --check
TEST_MATRIX.md up to date (42 modules, 448 tests declared, 0 dead fns)

$ python3 tools/lint_docs.py
 tools/lint_docs.py: 0 parity violations (7 checks, 82 Settings fields)

$ git diff --check
# exit 0

$ command -v cargo; command -v rustc; command -v rustup; command -v rustfmt
# no output: none of these executables is installed in this environment

$ cargo test --all-targets --all-features
# NOT TESTED: cargo is unavailable; no substitute result is claimed

$ cargo fmt --all -- --check
# NOT TESTED: rustfmt/cargo is unavailable

$ cargo clippy --all-targets -- -D warnings
# NOT TESTED: cargo/rustc is unavailable

$ cargo build --release --target x86_64-pc-windows-msvc
# NOT TESTED: cargo/rustc and Windows/WinDivert/BFE are unavailable
```

The 375-check UI suite is real JavaScript/jsdom execution. It does not
substitute for Rust compilation or the Windows, fuzz, release-signing and
pcap jobs. The historical 431/431 Rust result belongs to the earlier baseline
and is not a result for this checkout after the current Rust edits.

## 4. Required artifacts before changing statuses

1. Run `cargo test --all-targets --all-features`,
   `cargo clippy --all-targets -- -D warnings`, and
   `cargo fmt --all -- --check`; copy raw output into the verification record.
2. Attach Windows `e2e.pcapng`, backend stderr, driver-load/hash evidence and
   the line printed by `assert-e2e-pcap.py`.
3. Exercise WFP/BFE installation and cleanup for IPv4 and IPv6, including
   failure halfway through the transaction and process termination.
4. Attach the three fuzz target artifacts and the release
   `signtool verify`/`Get-AuthenticodeSignature` output.
5. Exercise DoH against the pinned hostname, a valid custom hostname, a
   forbidden resolver result, certificate failure and redirect response.

Until those artifacts exist, `STATUS.md`, `README.md`, `TEST_MATRIX.md` and
module comments must retain their `[UNVERIFIED]`/`UNTESTED` meanings.

To roll back the current source-level follow-up without rewriting history:

```text
git revert ae7d4ea 42d9f52 37e2b3b 969d22d
```

To roll back the complete SENTRY implementation series, review and revert
`ae7d4ea`, `42d9f52`, `37e2b3b`, `969d22d`, `9c0a0e6`, `a455dcb`, and `7ce50e1` in dependency-aware order; do
not revert the report while leaving an implementation claim without evidence.
