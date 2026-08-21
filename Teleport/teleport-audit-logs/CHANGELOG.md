# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.1.0] - 2026-07-16

### Added

- Coverage for the 8 event types seen in a live intake extract that previously normalized through the common stage only, without any `event.category`, `event.type` or `event.outcome`:
  - SSH session lifecycle: `session.start`, `session.join`, `session.leave`, `session.end`, `session.data` (category `session`; `session.data` also `network`)
  - Identity lifecycle: `user.create`, `lock.created`, `reset_password_token.create` (category `iam`)
- `event.outcome` for every mapped event type, set inside that event's own stage by comparing the **full** event code, per the [audit events reference](https://goteleport.com/docs/reference/audit-events/) — e.g. `T2000I` for `session.start`, `T1002I` for `user.create`. Only four event types have a documented failure code (`user.login` T1001I/T1001W, `db.session.start` TDB00I/TDB00W, `db.session.query` TDB02I/TDB02W, `exec` T3002I/T3002E); the rest set `success` on their single documented code. `ssm.run` is the exception: its code `TDS00I` is emitted whether the command succeeded or not, so its outcome comes from the `status` field.
- `user.effective.name` from `login` — the OS account a Teleport identity assumed on a node (`user.name` remains the Teleport identity)
- `user.target.name` / `user.target.roles` on `user.create`, `lock.created` and `reset_password_token.create` — the user acted upon, previously dropped entirely. The engine also folds it into `related.user`, making the target pivotable.
- `destination.ip` / `destination.port` from `addr.local` (the Teleport node side), via a new `parsed_local` grok step
- `source.bytes` / `destination.bytes` from `session.data` `rx` / `tx`
- `event.start` / `event.end` on `session.end` from `session_start` / `session_stop`
- `network.protocol` from `proto` on session start/end
- New custom fields: `teleport.server.*`, `teleport.session.*`, `teleport.lock.name`, `teleport.token.ttl`, `teleport.namespace`, `teleport.private_key_policy`, `teleport.user_kind`, `teleport.user_cluster_name`
- Smart descriptions for each new event type
- `Process monitoring` data source for the SSH session lifecycle
- Test cases for the 8 new event types (18 fixtures total)

### Fixed

- Removed a stale comment describing the `CreatedAt` storage-envelope field, which no longer exists after 2.0.0

## [2.0.0] - 2026-07-16

### Changed

- **BREAKING**: the parser now expects a Teleport audit event as the raw `message` — the flat object previously nested under `FieldsMap`. The earlier samples were exported from Teleport storage (Athena/DynamoDB) and carried a storage envelope (`CreatedAt`, `CreatedAtDate`, `EventIndex`, `EventNamespace`, `EventType`, `Expires`, `SessionID`, `FieldsMap`) that a live intake never receives. All field references dropped the `FieldsMap.` segment.
- `teleport.session_id` now reads `sid` (the event's own session ID) instead of the envelope's `SessionID`
- Database protocol moved from `teleport.db.protocol` to the native ECS `network.protocol`
- Identity provider groups moved from `teleport.user_groups` to the native ECS `user.roles`
- `teleport.auth.connector_id` renamed to `teleport.auth.connector`
- `exec` outcome now derives from the Teleport event code (`T3002I` success / `T3002E` failure) rather than `exitCode`, per the [Teleport audit events reference](https://goteleport.com/docs/reference/audit-events/#exec)

### Added

- `observer.name` from the Teleport cluster name

### Fixed

- `teleport.session_id` is no longer populated on events that have no session (`user.login`, `cert.create`, `role.created`, `role.deleted`, `ssm.run`). The storage envelope supplied a synthetic `SessionID` for these, giving them a session they never had.
- Removed the `user.id` mapping from `user.login`. It was fed by `uid`, the audit event's own identifier, which is event-scoped — the same user gets a different `uid` on every login, so it never identified a user. The value remains available as `teleport.uid`.
- Removed a dead `source.ip` mapping in `user.login` that read `addr.remote` through dotted access (the key is the literal string `addr.remote`, so it never resolved). `source.ip` is set correctly in the common stage from the grok-parsed remote address.

## [1.0.0] - 2026-06-22

### Added

- Initial release of the Teleport parser
- JSON parsing of Teleport audit events from the `message` field (payload under `FieldsMap`)
- ECS mapping: `observer`, `event`, `user`, `source`, `destination`, `cloud`, `host`, `orchestrator`, `container`, `process`, `error`, `user_agent`, `related`
- Category-specific routing: `user.login` (authentication), `db.session.start` / `db.session.query` (database), `ssm.run` and `exec` (process), `role.created` / `role.deleted` (iam), `cert.create` (authentication)
- Filter-free common stage so unseen Teleport event types still normalize their common fields
- Custom fields under the `teleport.*` namespace
- Smart descriptions for event summarization in Sekoia
- Test cases for the 8 supported event types, including a database access-denied failure and an exec edge case without an exit code

[2.1.0]: https://github.com/SEKOIA-IO/intake-formats/releases/tag/v2.1.0
[2.0.0]: https://github.com/SEKOIA-IO/intake-formats/releases/tag/v2.0.0
[1.0.0]: https://github.com/SEKOIA-IO/intake-formats/releases/tag/v1.0.0
