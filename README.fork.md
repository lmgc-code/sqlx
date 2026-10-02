# sqlx (sentin8 fork) — OAUTHBEARER for PostgreSQL 18

Fork of [launchbadge/sqlx](https://github.com/launchbadge/sqlx) at tag
`v0.8.6` (`bab1b02`), branch `pgoauth-oauthbearer`, adding the
`OAUTHBEARER` SASL mechanism (RFC 7628) to `sqlx-postgres` so clients
can authenticate with a bearer token against PostgreSQL 18's native
`oauth` HBA method.

**This is a standalone repository** — the sentin8 platform consumes it
via a `[patch.crates-io]` path entry and never vendors it
(`design/sentin8-vertical-data-plane-contract.md` §VDC.3b).

## Why the fork exists

Upstream sqlx 0.8.6 implements only `SCRAM-SHA-256[-PLUS]` SASL
(`sqlx-postgres/src/connection/sasl.rs`); PostgreSQL 18's `oauth`
authentication method requires the client to speak `OAUTHBEARER`
(RFC 7628). No released Rust driver supports it as of this fork.

## What changed vs upstream v0.8.6

| File | Change |
|---|---|
| `sqlx-postgres/src/connection/sasl.rs` | OAUTHBEARER mechanism: offered by server + `oauth_token` present → token exchange (client-initial `n,,\x01auth=Bearer <token>\x01\x01`, empty-challenge client-final, `SaslFinal`/`Ok` handling; non-empty challenges complete the failure form and surface the JSON reason). SCRAM flow unchanged. |
| `sqlx-postgres/src/options/mod.rs` | `PgConnectOptions::oauth_token` field + `.oauth_token(token)` builder. When the mechanism is negotiated, `password` is unused. |
| `sqlx-postgres/src/message/sasl.rs` | `SaslInitialResponse` gains `mechanism: Option<&'static str>` (`None` = legacy SCRAM derivation). |
| `sqlx-postgres/src/message/authentication.rs` | `AuthenticationSaslContinue` decoder guards short/empty segments — the OAUTHBEARER success challenge is empty, error challenges are JSON (neither is SCRAM `k=v` shaped; unguarded indexing would panic). |

## Semantics

- Mechanism choice follows the credential: server offers
  `OAUTHBEARER` and the options carry `oauth_token` → OAUTHBEARER;
  otherwise SCRAM (password) as before. A server may offer both.
- Tokens are per-connection credentials: the caller sets
  `oauth_token` from its token cache at pool-connect time; the pool
  re-authenticates per new connection, so a refreshed cache is
  sufficient near expiry (no mid-connection renewal — PostgreSQL
  authenticates once per connection).

## Maintenance

- Rebase discipline: keep this branch minimal and rebase onto new
  upstream tags; the patch set is deliberately small (4 files).
- Upstreaming is desirable (an `OAUTHBEARER` feature flag) — until
  then this fork is the platform's driver.

## License

Upstream sqlx is MIT OR Apache-2.0; this fork carries the same terms.