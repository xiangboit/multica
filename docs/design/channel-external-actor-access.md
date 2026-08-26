# Auditable External Actors for Channel Integrations

- Status: Proposed
- Initial channel: Feishu/Lark
- Related work: [#6986](https://github.com/multica-ai/multica/pull/6986), [#5150](https://github.com/multica-ai/multica/pull/5150), [#5906](https://github.com/multica-ai/multica/pull/5906)
- Audience: Multica maintainers, security reviewers, channel-integration owners, and self-hosted operators

## 1. Summary

Multica channel integrations currently authorize people by binding a provider
identity to a Multica workspace member. That is the correct default for agents
that can run code and connected applications, but it prevents a common support
workflow: a workspace wants one narrowly-scoped agent to answer questions from
people in the same Feishu tenant without requiring every caller to create a
Multica account.

The first implementation in #6986 adds an installation-level opt-in for such
callers, constrains them to the installation's Feishu tenant, removes personal
connected applications, and keeps `originator_user_id` null. It proves the
product flow, but it deliberately does not persist the exact external caller.
The task therefore falls back to the installation owner for display and audit,
which is not sufficient for a durable, fail-closed authorization model.

This document proposes a channel-neutral extension with three new concepts:

1. **External actor**: a stable, privacy-preserving record for a provider user
   who is not a Multica member.
2. **Access grant**: an immutable snapshot of the administrator-approved policy
   that allowed an external actor to invoke one channel installation.
3. **Invocation context**: the actor, grant, authorization principal, accountable
   principal, source message, and capabilities carried through chat, media,
   tasks, and `/issue` creation.

The design reuses Human Attribution instead of inventing a second authorization
system. An external actor never becomes a Multica authorization principal:

- `originator_user_id` remains `NULL`;
- `accountable_user_id` is the workspace administrator who enabled the grant;
- the exact caller and grant are stored separately and immutably;
- user-scoped connected applications and MCP connections remain unavailable;
- identity or attribution failures reject the request before agent side effects.

The schema changes are additive. Existing member-only installations and legacy
rows continue to work. A new, versioned access-mode value is required to make
mixed-version deployments and binary rollback fail closed.

## 2. Background

### 2.1 Product scenario

A company may expose a Feishu-bound agent to sales, support, operations, or
other internal employees. Those users need consultation and structured issue
filing, but they do not need Multica workspace membership or access to the
installer's personal tools.

Requiring every caller to register in Multica creates operational friction and
turns workspace membership into an identity-provisioning mechanism. Allowing
all same-tenant callers without durable attribution creates the opposite
problem: the system can no longer answer who invoked an agent, under which
policy, or why the invocation was allowed.

The intended product boundary is therefore:

- registration is optional for the caller;
- enabling external access is explicit and administrator-controlled;
- the caller must be an exact human sender in the installation's tenant;
- capabilities are narrower than those of a bound workspace member;
- every accepted invocation has a stable actor and an immutable policy record;
- privacy-sensitive native identifiers have a bounded, configurable lifetime.

### 2.2 Current behavior

The existing channel path resolves a sender through `channel_user_binding`.
That table maps a provider identity to a real Multica user and is used by
membership checks, per-user capabilities, and cleanup. It is not a guest table.

The #6986 implementation adds `workspace_members` and `feishu_users` modes to
the Feishu installation config. In `feishu_users` mode an unbound, same-tenant
sender is represented as a `ResolvedIdentity` whose durable owner is the
installer and whose `External` flag prevents it from inheriting the installer's
personal connected applications. `TaskInitiatorUserID()` returns null for that
sender.

This safely blocks the most important privilege escalation, but leaves four
gaps:

1. two different external people are indistinguishable in durable records;
2. a task cannot name the exact access decision that admitted its messages;
3. strict Human Attribution workspaces reject or ambiguously display the task;
4. an older binary that understands `feishu_users` can re-enable the unaudited
   behavior after rollback.

### 2.3 Existing constraints that must remain true

- `originator_user_id` is an authorization input, not a display convenience.
- `accountable_user_id` is for visibility, audit, and cost attribution; it must
  never grant user-scoped capabilities.
- If an originator exists, the database requires accountable and originator to
  be the same user.
- `chat_session.creator_id` is a required Multica user and remains a technical
  session owner, not proof of the sender of every message.
- `issue.creator_type` has established product and permission semantics.
- Channel tables intentionally avoid foreign keys and cascades; relationships
  and cleanup are enforced by the application.
- Installed clients may connect to a newer backend, so API additions must be
  optional and parsed defensively.

## 3. Goals and non-goals

### 3.1 Goals

- Let an owner or workspace administrator enable external access for one
  channel installation without adding the callers as workspace members.
- Accept only exact human senders from the installation's tenant.
- Persist a stable external actor for every accepted external invocation.
- Record the exact access grant used by chat, media, task, and `/issue` flows.
- Preserve the existing Human Attribution authorization boundary.
- Support recoverable and pseudonymous identity policies without plaintext
  provider identifiers in normal database columns or logs.
- Make missing identity, invalid tenant, unavailable crypto, and attribution
  failures visible and fail closed.
- Provide a safe upgrade, mixed-version, and rollback path.
- Keep the data model channel-neutral so DingTalk, WeCom, Slack, Telegram, and
  future adapters can opt in later.

### 3.2 Non-goals

- Giving an external actor workspace membership or a Multica login.
- Giving an external actor the installer's personal MCP servers, connected
  applications, repository credentials, or environment variables.
- Solving cross-tenant or public internet bot access.
- Building a general external identity provider or customer IAM system.
- Retrofitting exact identities onto legacy events whose sender identifiers
  were never stored.
- Changing issue permission semantics by adding `external_actor` as an issue
  creator type.
- Automatically enabling guest access during migration.

## 4. Design principles

1. **Authorization and identity are separate.** Knowing who sent a message does
   not make that person a Multica principal.
2. **The allow decision is durable.** A later policy edit must not rewrite the
   reason an older invocation was accepted.
3. **The default is member-only.** Missing, malformed, or unknown policy values
   normalize to `workspace_members`.
4. **Sensitive identity is recoverable only by policy.** Correlation and
   reversal are separate data operations.
5. **Retries preserve context.** A retry, debounce recovery, or media completion
   cannot acquire broader capabilities than the original message.
6. **Rejections are observable.** The sender receives a non-sensitive failure
   reply, and operators receive structured audit and metrics.
7. **Old binaries must fail closed.** A rollback must not silently restore an
   earlier unaudited guest mode.

## 5. Terminology

| Term | Meaning |
| --- | --- |
| Member actor | A provider identity bound to a current Multica workspace member. |
| External actor | A provider identity accepted without a Multica user binding. |
| Installation | One channel application or bot bound to one Multica agent. |
| Access grant | An immutable administrator-approved policy snapshot for one installation. |
| Native subject | The provider's stable user identifier, such as Feishu `open_id`. |
| Subject lookup | A keyed HMAC used to find the same external actor without storing the native subject in plaintext. |
| Invocation context | The normalized actor, grant, attribution, capability, and source evidence used by downstream flows. |
| Identity reveal | An audited operation that decrypts a recoverable native subject for an authorized investigation. |

## 6. Proposed architecture

```text
provider event
    │
    ▼
adapter: verify event, decode exact sender kind and native subject
    │
    ▼
tenant check + active access-grant lookup
    │
    ├── bound member ───────► existing member authorization
    │
    └── unbound human ──────► resolve/upsert external actor
                                  │
                                  ▼
                         build InvocationContext
                                  │
                   ┌──────────────┼──────────────┐
                   ▼              ▼              ▼
              chat message      media         /issue
                   │              │              │
                   └──────────────┼──────────────┘
                                  ▼
                        task attribution + run
```

The channel engine owns the normalized orchestration. Provider adapters own
only provider verification, sender decoding, tenant extraction, media download,
and provider-specific replies.

### 6.1 Invocation context

The in-memory channel contract should replace the ambiguous `External bool`
with an explicit context similar to:

```go
type InvocationActorKind string

const (
	InvocationActorMember   InvocationActorKind = "member"
	InvocationActorExternal InvocationActorKind = "external_actor"
)

type InvocationContext struct {
	ActorKind             InvocationActorKind
	ActorID               pgtype.UUID
	AuthorizationUserID   pgtype.UUID
	AccountableUserID     pgtype.UUID
	AccessGrantID          pgtype.UUID
	SourceMessageID        string
	CapabilityProfile     CapabilityProfile
	DisableConnectedApps  bool
}
```

For a member, `ActorID`, `AuthorizationUserID`, and `AccountableUserID` all
identify the member and `AccessGrantID` is null. For an external actor,
`AuthorizationUserID` is null, `AccountableUserID` is the administrator who
enabled the grant, and `AccessGrantID` is required.

The context is computed once, before durable side effects, and is passed to
session, task, media, and issue services. Downstream code must not reconstruct
permissions from the current installation config.

## 7. Data model

The following SQL is illustrative. Final migrations must follow repository
rules: no foreign keys or cascades, a paired down migration, and every index in
its own single-statement migration using `CONCURRENTLY`.

### 7.1 `channel_access_grant`

```sql
CREATE TABLE channel_access_grant (
    id                    UUID PRIMARY KEY,
    workspace_id          UUID NOT NULL,
    installation_id       UUID NOT NULL,
    policy_version        INTEGER NOT NULL,
    access_mode           TEXT NOT NULL,
    identity_storage_mode TEXT NOT NULL,
    identity_retention_days INTEGER,
    capability_profile    JSONB NOT NULL DEFAULT '{}'::jsonb,
    agent_capability_revision BIGINT NOT NULL,
    enabled_by_user_id    UUID NOT NULL,
    enabled_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_by_user_id    UUID,
    revoked_at            TIMESTAMPTZ,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

An active grant has `revoked_at IS NULL`. There may be only one active grant
per installation; enforce that with a partial concurrent unique index. Editing
guest policy revokes the old row and creates a new row instead of mutating the
historical decision.

`capability_profile` is a versioned snapshot of the capability policy the
administrator chose to expose. The initial Feishu profile allows conversation,
file ingestion, and `/issue` creation; denies the caller any member-scoped
connected applications or personal MCP overlay; and records that execution
inherits the target Agent's configured repositories, workspace MCP
servers, environment, and filesystem mode. It does not create a second runtime
sandbox or promise read-only repository behavior when the Agent itself is
write-enabled. This is why only a workspace owner or administrator may enable
the grant. A future per-invocation sandbox may narrow those Agent capabilities,
but must be implemented and tested before the profile advertises that limit.
The snapshot stores policy decisions and versions only; it never copies
repository credentials, environment values, MCP secrets, or connected-app
tokens into the grant row.

`agent_capability_revision` makes that administrator approval stable. Any
exposure-relevant Agent change—repository/resource bindings, workspace MCP,
environment, runtime, or filesystem mode—advances the Agent revision. A grant
whose revision no longer matches is suspended for new external messages until
an owner/admin reviews and replaces it. An already-enqueued external task also
checks the captured revision at claim time and fails visibly rather than
executing with newly broadened current capabilities. Member invocations are not
blocked by this guest-specific revision gate.

The installation config remains the current-state projection:

```json
{
  "inbound_access_mode": "feishu_users_audited",
  "active_access_grant_id": "...",
  "guest_access_policy_version": 2
}
```

Runtime authorization reads and validates the referenced active grant. The
JSON value alone never admits an external actor.

### 7.2 `channel_external_actor`

```sql
CREATE TABLE channel_external_actor (
    id                    UUID PRIMARY KEY,
    workspace_id          UUID NOT NULL,
    installation_id       UUID NOT NULL,
    channel_type          TEXT NOT NULL,
    subject_lookup        BYTEA NOT NULL,
    lookup_key_version    SMALLINT NOT NULL,
    subject_ciphertext    BYTEA,
    identity_storage_mode TEXT NOT NULL,
    encryption_key_version SMALLINT,
    linked_user_id        UUID,
    first_seen_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    retention_until       TIMESTAMPTZ,
    subject_ciphertext_purged_at TIMESTAMPTZ,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`subject_lookup` is a keyed HMAC over a canonical, scope-bound value:

```text
channel_type || NUL || installation_id || NUL || native_subject
```

It supports equality lookup and deduplication but cannot be reversed without
the secret key. A concurrent unique index on `(installation_id,
lookup_key_version, subject_lookup)` prevents duplicates for one lookup-key
generation. Resolution computes candidate HMACs for every retained lookup key,
then lazily rewrites a matched actor to the current lookup-key version in the
same transaction. An old lookup key cannot be retired until recoverable actors
have been reindexed and the operator accepts that any dormant
`pseudonymous_only` actors not seen during the rotation window will no longer
correlate. Encryption-key rotation is independent from lookup-key rotation.

`subject_ciphertext` is optional. It is present only for
`encrypted_recoverable` mode. `linked_user_id` may be set if the person later
binds a Multica account; historical message actor IDs remain unchanged, while
new events can use normal member authorization.

The initial implementation does not persist display name, email, phone,
`union_id`, avatar, or message body in this table. Those fields are not needed
for authorization and increase privacy risk.

### 7.3 Existing table extensions

Additions to historical message/task/issue rows are nullable so old rows and
writers remain valid. The Agent revision is a metadata-only constant-default
counter whose value `1` represents the pre-feature capability generation.

#### `agent`

```text
external_access_capability_revision BIGINT NOT NULL DEFAULT 1
```

An authoritative database trigger increments this revision when
exposure-relevant columns on `agent` change, so an adjacent old binary cannot
bypass the gate during a rolling deployment. Related tables such as Agent ↔
workspace-MCP bindings bump the Agent revision in the same transaction through
their write service or a narrow relation trigger. Every create/update/delete
entry point is covered by a migration test. The constant default keeps old
Agents compatible; the grant snapshots the value that an administrator
reviewed.

#### `chat_message`

```text
sender_type             TEXT NULL  -- member | external_actor
sender_id               UUID NULL
channel_access_grant_id UUID NULL
```

The message is the immutable evidence of who supplied each piece of channel
input. A batched task may include messages from more than one actor; recording
the actor only on the task would lose that distinction.

#### `channel_chat_context_generation`

```text
channel_access_grant_id UUID NULL
```

An external policy change closes or flushes the current debounce generation.
The next accepted message creates a generation under the new grant. A change
between member authorization and external-grant authorization is the same kind
of boundary: it flushes or advances the generation instead of combining both
into one task with ambiguous Human Attribution. Multiple external actors may
share a generation only when they use the same grant and approved Agent
capability revision.
This prevents one recovered or debounced batch from mixing authorization
semantics or policy snapshots.

#### `agent_task_queue`

```text
channel_access_grant_id UUID NULL
external_access_capability_revision BIGINT NULL
```

External tasks also use existing trigger evidence with
`trigger_evidence_kind = 'channel_message'` and the exact source chat message
as `trigger_evidence_ref_id`. The task-level grant and capability revision are
the claim/retry gate and audit anchor; message-level actor records remain the
source for multi-sender input. A null task revision is the legacy/member case,
not permission to bypass the external gate.

#### `issue`

```text
origin_message_id UUID NULL
```

An issue created by an external `/issue` command keeps the existing semantics:

- `creator_type = 'agent'`;
- `creator_id` is the receiving agent;
- `origin_type` identifies the channel;
- `origin_id` identifies the chat session;
- `origin_message_id` identifies the exact command, actor, and grant.

This avoids expanding `creator_type`, which would ripple through issue
permissions, UI assumptions, and existing client enums.

### 7.4 Why `channel_user_binding` is not reused

`channel_user_binding.multica_user_id` is required and means a current bound
workspace member. Membership validation, capability resolution, and cleanup
depend on that meaning. Making the user nullable or inserting the installer
for every external actor would turn a security boundary into an overloaded
identity table and would make ordinary membership queries unsafe.

External actors therefore need a separate table and an explicit actor kind.

### 7.5 Why `activity_log` is not the grant source of truth

`activity_log` should record enable, revoke, reveal, ciphertext purge, and
retention-extension events. It is useful for audit and already supports fail-closed
reveal behavior elsewhere in Multica. It is not a runtime policy table: finding
the current grant by replaying loosely structured activity payloads would be
slow, fragile, and difficult to enforce atomically.

## 8. Identity privacy, crypto, and retention

### 8.1 Storage modes

| Mode | Stored data | Correlate | Reveal native subject | Intended use |
| --- | --- | --- | --- | --- |
| `encrypted_recoverable` | HMAC + versioned ciphertext | Yes | Authorized, audited | Recommended default |
| `pseudonymous_only` | HMAC only | Yes | No | Privacy-maximizing deployments |

Guest access cannot use a `none` mode because it would remove durable actor
traceability. Plaintext storage is not supported.

The HMAC is still pseudonymous personal data and must be protected by the same
workspace and operator controls as other audit records.

### 8.2 Key separation

The existing secretbox helper provides AES-256-GCM with a random nonce, but it
does not provide searchable identity, associated-data binding, or production
key rotation. Channel identity should use a small dedicated crypto service:

- an operator-provided versioned identity keyring;
- HKDF-derived, separately labelled encryption and HMAC subkeys;
- AES-256-GCM ciphertext;
- associated data containing actor ID, workspace ID, installation ID, channel
  type, and key version;
- ciphertext prefixed with a format and encryption-key version;
- versioned lookup keys retained long enough to resolve and lazily reindex old
  HMACs;
- no plaintext native subject in logs, errors, metrics, or activity payloads.

For deployment compatibility, absence of an identity key disables audited guest
access only. It must not prevent server startup or member-only Feishu bots from
working.

### 8.3 Retention model

The design separates durable audit identity from reversible provider identity:

- actor UUID, subject HMAC, message link, grant link, and task/issue evidence
  follow the workspace's task and audit retention policy;
- encrypted native subject follows a rolling retention period from
  `last_seen_at`, configurable by an administrator;
- an open issue, incident investigation, or legal hold may extend
  `retention_until` with an audited reason;
- expiry removes ciphertext and encryption-key-version data and sets
  `subject_ciphertext_purged_at`; it does not rewrite historical messages or
  delete the actor UUID/HMAC.

After ciphertext expires, Multica can still prove that two historical events
came from the same pseudonymous actor, but cannot turn that actor back into a
Feishu `open_id`. This is pseudonymization, not full anonymization. If the
person sends another message, a retained lookup key resolves the same actor and
recoverable ciphertext may be stored again under the current policy. This
behavior must be explicit in the UI and operator documentation; “180 days” is
a reversal window, not the lifetime of the audit event.

### 8.4 Identity reveal

Reveal is a separate administrator action, not part of ordinary task display.
It requires:

1. workspace owner or admin permission;
2. a non-empty investigation reason;
3. an unexpired recoverable ciphertext;
4. successful insertion of an immutable `external_actor.identity_revealed`
   activity record before plaintext is returned.

If audit insertion or decryption fails, no plaintext is returned. The API must
not include the subject in realtime events or generic task payloads.

## 9. Authorization and attribution

### 9.1 Enable and revoke permissions

Only the workspace owner or an administrator may create, replace, or revoke an
external access grant. Owning the target agent or installing its bot is not
sufficient.

An enable operation validates the identity-key configuration, creates the new
grant, updates the installation projection, and records the activity event in
one application transaction. A revoke operation closes the grant, resets the
installation to `workspace_members`, advances the channel context generation,
and records the revoker.

An exposure-relevant Agent capability change advances its capability revision.
The grant row remains immutable for audit, but external admission becomes
suspended because the active grant no longer matches the Agent. A current
owner/admin must replace the grant to approve the new capability revision.

The enabler's owner/admin role is checked when the grant is created or replaced.
A later role change does not rewrite or silently revoke that historical
workspace decision; the immutable grant remains active until a current owner or
administrator revokes or replaces it. Workspace deletion and installation
revocation still close the grant. Deployments that require a continuously
active sponsor should add an explicit grant-expiry or re-approval policy rather
than deriving revocation implicitly from membership cleanup.

Revocation is a kill switch, not only a configuration edit. It prevents new
messages immediately and makes queued, deferred, retry, and rerun attempts under
that grant ineligible at claim/enqueue time. Revocation also requests
cancellation of dispatched/running tasks through the existing task-cancel path;
that cancellation is best-effort because a remote process may already be
executing. Historical tasks retain the revoked grant and approved Agent
capability revision for audit. Grant state and capability revision are checked
separately: neither a revoked grant nor a revision mismatch may fall back to a
newer/broader policy.

### 9.2 Sender requirements

An external message is accepted only when all of the following are true:

- the provider event and installation have been authenticated;
- the adapter reports the exact sender kind as a human user;
- sender kind is not bot, app, system, missing, or unknown;
- an effective installation tenant is available: normally the persisted
  installation tenant, or only for a legacy row with that field absent, the
  authenticated event-header tenant from the installation's WebSocket;
- sender tenant is present and exactly equals that effective installation
  tenant; when both persisted and event-header tenant exist they also agree;
- an active, version-supported access grant exists;
- the external actor can be resolved or persisted;
- the workspace attribution policy permits the resulting context.

Unknown values never fall back to “human”.

### 9.3 Human Attribution mapping

| Field | Bound member | External actor |
| --- | --- | --- |
| Authorization user | Member | `NULL` |
| `originator_user_id` | Member | `NULL` |
| `accountable_user_id` | Member | Grant-enabling admin |
| `originator_source` | `direct_human` | `external_grant` |
| Trigger evidence | Chat/session evidence | Exact `channel_message` |
| Exact actor | Member sender | `channel_external_actor` |
| Policy evidence | Existing membership | `channel_access_grant` |
| Personal connected apps | Member policy | Disabled |

This preserves the rule that accountable identity does not confer permissions.
The admin is accountable for exposing the agent under the recorded capability
profile; the external actor is the exact human who invoked it.

The attribution package should add an `ExternalGrant` constructor rather than
hand-building task fields. It returns null authorization user, the grant
enabler as accountable user, source `external_grant`, and the exact source
message as evidence. Delegated tasks then reuse the existing rule that copies a
parent's accountable human while keeping a null originator. `external_grant`
is a precise attribution source in strict workspaces when the exact external
actor, immutable active grant, exact source message, and approved Agent
capability revision are all present. `originator_user_id` remains null because
the caller has no Multica account; that intentional absence does not make the
invocation unattributed. The grant enabler is accountable but never becomes
the authorization user or caller.

This combination satisfies `attribution_fail_closed`. If any required actor,
grant, message evidence, tenant check, or capability revision is absent or
invalid, the invocation is rejected before enqueue. It must never enter
`SourceUnattributed`, use `owner_fallback`, or borrow the grant enabler's
personal permissions.

## 10. End-to-end processing

### 10.1 Normal chat

1. Verify and deduplicate the provider event.
2. Decode exact sender type, native subject, sender tenant, and installation
   tenant.
3. Resolve a bound member. If none exists, evaluate the active audited grant.
4. Resolve or upsert the external actor using the subject HMAC.
5. Build and preflight the invocation context, including strict attribution.
6. Ensure the chat session. For groups and external direct messages,
   `chat_session.creator_id` remains the installer as a technical owner.
7. Append chat message, actor, grant, and dedup completion atomically.
8. Append to the current context generation or start a new one.
9. Enqueue the task with the same grant, policy profile, and approved Agent
   capability revision.
10. Emit the classified outcome: visible guidance for actionable human failures
    and a terminal silent drop for unsafe/non-actionable sender classes.

### 10.2 Debounce, retries, and recovery

Actor and grant fields are persisted before enqueue. A recovered debounce batch
loads them from messages/context generation; it never calls current policy to
substitute a newer grant for an already accepted event. Claim still checks the
captured grant's revocation state and approved Agent capability revision.
Member and external messages use different generations, so one task has one
authorization model. An external batch always disables member-scoped connected
applications.

A batch may contain messages from multiple external actors admitted by the same
grant. The task links to that grant and source evidence, while individual
messages preserve each actor. The agent prompt may receive safe participant
labels such as “External user 1” without native IDs.

Changing or revoking a grant flushes or advances the context generation before
the new policy becomes active. Messages admitted under different grants cannot
share one debounce generation.

### 10.3 Files and media

Media follows the same preflight as text. Provider download credentials are
installation-scoped and may be used only to fetch the message's declared file.
The file is attached to the source chat message, whose actor and grant are
already durable.

The source message, media intent, deferred task, actor, and grant persist enough
context for reconciliation and crash fallback. The current remote-download
goroutine itself is best-effort and is not resumed after a process crash; the
persisted deadline eventually finalizes the placeholder and releases the task.
No recovery path re-resolves the sender through the latest installation mode.
Authorization or identity preflight failure still prevents all issue/task side
effects. Once a message and issue have been accepted, remote download, scan,
timeout, or attachment-binding failure follows Multica's existing
channel-media fallback: the text/placeholder remains durable, the media-pending
marker is finalized, and the deferred task becomes runnable rather than
remaining stuck forever. The failure is observable to operators, but the
initial implementation does not send a separate provider warning to the
sender. The eventual task response remains the only normal outbound result.
Changing this fallback to cancel the issue/task or add a second warning is a
separate product decision, not part of external identity.

### 10.4 `/issue`

The command is an explicit product side effect and uses the same preflight as
agent execution.

For text-only commands, issue creation, origin-message linkage, and any assigned
task enqueue must commit atomically. For media commands, issue creation and its
deferred task commit atomically after authorization preflight. Attachment
binding then finalizes the media marker and promotes the task; on terminal
media failure the existing placeholder fallback still promotes the task rather
than leaving it deferred forever. Provider message ID and origin message ID
keep all paths idempotent.

An accepted source chat message may remain as audit evidence if downstream
issue creation fails, but the issue and task must not be partially committed.
Retries use provider message ID and origin message ID as idempotency keys.

## 11. Failure semantics

The following conditions reject before task or issue side effects:

- missing, bot, app, system, or unknown sender type;
- missing or mismatched tenant;
- inactive, revoked, missing, or unsupported grant;
- missing identity key for recoverable mode;
- HMAC, encryption, actor persistence, or grant persistence failure;
- incomplete or invalid precise attribution evidence in a strict workspace;
- capability-profile parse or version failure.

Recognized human senders should receive replies for actionable classes without
leaking security details, for example:

- “This agent is available only to authorized members.”
- “External access is temporarily unavailable; ask a workspace administrator
  to check the bot's access policy.”
Bot/app/system senders, missing or unknown sender kinds, unverifiable or
cross-tenant senders, non-addressed group chatter, and duplicate events are
terminal silent drops.
Replying to those classes could reveal policy or create an automated reply
loop. Their outcome is visible only through bounded internal reason codes.

Internal logs and metrics carry a stable reason code, installation ID, grant
ID when available, and pseudonymous actor ID when persistence succeeded. They
must not carry the native subject or message body.

## 12. API and UI

The installation response adds optional, additive fields:

```json
{
  "inbound_access_mode": "workspace_members",
  "supports_audited_guest_access": true,
  "active_access_grant": {
    "id": "...",
    "policy_version": 2,
    "identity_storage_mode": "encrypted_recoverable",
    "identity_retention_days": 180,
    "enabled_by_user_id": "...",
    "enabled_at": "..."
  }
}
```

The UI shows the control only when the backend advertises support. Enabling it
uses a dedicated grant endpoint rather than a generic config patch, because it
requires administrator authorization, crypto readiness, policy validation, and
an immutable audit row.

The confirmation dialog explains:

- who may invoke the agent;
- which capabilities are denied;
- what identity data is stored and for how long;
- who is accountable for enabling access;
- how to revoke access.

Older clients ignore optional fields and continue to display member-only mode.
New clients connecting to an older backend hide the control when the capability
field or endpoint is absent.

## 13. Compatibility and migration

### 13.1 Compatibility rule

Database compatibility alone is insufficient. A mixed-version system is safe
only when an old process cannot interpret a new guest policy as permission to
run an unaudited external task.

The new mode must therefore use a value that older implementations do not
recognize, for example:

```text
feishu_users_audited
```

Official versions before #6986 require bindings and ignore this config. The
current #6986 branch normalizes unknown modes to `workspace_members`. Both fail
closed. Reusing the old `feishu_users` value would be unsafe because an old
Fork binary would accept guests without external-actor persistence.

New code admits an external actor only when all three signals agree:

1. mode is `feishu_users_audited`;
2. `guest_access_policy_version` is supported;
3. `active_access_grant_id` resolves to an active grant for the same workspace
   and installation.

### 13.2 Schema compatibility

The migration sequence is expand-first:

1. create new actor and grant tables;
2. add nullable columns to message, context generation, task, and issue tables,
   plus the constant-default Agent capability revision and its bump triggers;
3. create concurrent indexes in separate migrations;
4. deploy code that can read and write the new schema while guest access stays
   disabled;
5. let administrators explicitly create audited grants.

No existing column changes meaning. No existing `NOT NULL` constraint, enum
check, or creator-type check is widened. Legacy rows with null actor/grant
fields mean “recorded before audited channel actors”; they must never be shown
as precisely attributed.

Old binaries can read and write the expanded schema because historical event
columns are nullable, the Agent revision has a constant default, and new tables
are independent. Database triggers still bump revisions for relevant Agent
updates issued by an adjacent old writer. New binaries must tolerate legacy
null fields.

### 13.3 Existing #6986 installations

An installation currently configured as `feishu_users` has no durable external
actor history. Upgrade must not silently convert it into an audited grant.

The upgrade process resets or treats `feishu_users` as `workspace_members` and
asks an owner or administrator to review identity retention and enable a new
audited grant. Existing messages/tasks remain legacy-unattributed. Native
subjects cannot be backfilled because the old path did not store them.

This creates a brief, intentional loss of guest availability rather than a
false claim of audit continuity.

### 13.4 Mixed-version processes

Channel WebSocket ownership already uses a lease and should continue to permit
only one active hub for an installation. During rolling deployment:

- an old hub sees the unknown audited mode and rejects unbound callers;
- a new hub accepts only after resolving the v2 grant;
- lease takeover must complete before the new mode is enabled;
- workers use persisted invocation context and never infer permissions from
  their own binary's current defaults.

If a deployment cannot guarantee version-compatible workers for persisted
channel jobs, audited guest activation must wait until all workers run the new
version.

### 13.5 Client/server compatibility matrix

| Combination | Expected behavior | Impact/risk | Required mitigation |
| --- | --- | --- | --- |
| Old official server + old client | Member bindings only | None | No change |
| New server + old official client | Member flows work; the client has no guest control | Low | Optional response fields and member-only default |
| New server + old #6986 Fork client | Member flows work; the old toggle may display v2 as disabled and its legacy PATCH is rejected | Medium UX impact, fail-closed security | Require Web/Desktop upgrade to manage grants; return an actionable unsupported-mode error |
| Old server + new client | Member flows work; guest control hidden | Low | Capability detection; no optimistic enum PATCH |
| New schema + old official server | New tables/columns ignored; member flows work; Agent trigger still advances exposure revision | Low | Additive nullable event schema plus constant-default Agent revision |
| New schema + old #6986 Fork server | Audited mode is unknown and normalizes to member-only | Low, temporary guest outage | Use `feishu_users_audited`, never reuse `feishu_users` |
| New server + legacy `feishu_users` config | Guest access disabled pending admin review | Medium availability impact | Explicit re-authorization migration/UI notice |
| Mixed old/new hubs | Only one hub may own an installation; old hub rejects v2 guests | Medium if leases overlap | Lease fencing and activation after takeover |
| New server without identity key | Member bots work; audited guest mode unavailable | Low | Readiness status and actionable admin error |
| Binary rollback after audited activation | Old binary rejects v2 guests; member flows continue | Medium availability impact, no audit bypass | Leave expanded schema; disable/revoke grants before planned rollback |
| Schema down-migration after audited use | Actor/grant evidence would be destroyed | High data-loss risk | Do not down-migrate in operational rollback; backup/export first |

### 13.6 Rollback policy

Operational rollback means rolling back the application binary while leaving
the additive schema in place. Before a planned rollback, revoke active grants
and verify installations are member-only. An emergency rollback remains secure
because the old binary does not recognize `feishu_users_audited`, although
external callers temporarily lose service.

Running down migrations after actors or grants exist is a destructive data
operation, not a normal application rollback. It requires explicit backup,
retention approval, and audit export.

### 13.7 Impact by historical version family

#### Versions before channel generalization

These versions may not understand the current `channel_*` schema at all and are
outside direct rolling-upgrade compatibility. Operators must follow the normal
supported migration path to a channel-generalized release before enabling this
feature. The design does not add a new exception to that existing boundary.

#### Channel-generalized official versions without #6986

Impact is small. They ignore new config keys and additive tables and continue
to require member bindings. Rolling back to them disables guests safely.

#### Fork versions containing the first #6986 implementation

This is the most affected group. Existing `feishu_users` settings must be
disabled and re-authorized; otherwise the system cannot claim exact audit
continuity. The service interruption is limited to unbound callers. Bound
members, installations, chat sessions, files, and existing issues remain
unchanged.

#### New audited-guest versions

They understand both legacy null rows and v2 audited rows. They never interpret
legacy guest tasks as precise external attribution. Future policy versions must
repeat the same unknown-version fail-closed rule.

### 13.8 Expected impact summary

| Area | Impact on older deployments | Assessment |
| --- | --- | --- |
| Bound member chat and existing bots | No behavior or data migration is required | Low |
| Existing issues, tasks, sessions, and files | New links remain null and are treated as legacy | Low |
| Official releases that never enabled guests | Expanded schema is ignored on rollback | Low |
| #6986 Fork installations with `feishu_users` enabled | External users stop working until an admin explicitly enables the audited v2 grant | Medium, intentional |
| Database migration on large `chat_message` or task tables | Nullable column additions require brief metadata locks; index builds are concurrent | Medium operational planning |
| Old installed Web/Desktop clients | Continue member flows but cannot view or manage the new policy | Low |
| Rolling multi-process channel deployments | Old and new workers must not concurrently reinterpret one installation | Medium; lease/version fencing required |
| Planned application rollback | Guests become unavailable, but cannot bypass audit | Medium availability, low security risk |
| Schema down-migration after use | Removes audit evidence | High; unsupported as routine rollback |

Overall, the design has **low impact on upstream member-only users** and a
**deliberate medium migration impact on deployments already using the unaudited
Fork mode**. Most application changes are additive. The largest engineering
surface is not old-row conversion; it is carrying one immutable invocation
context consistently through chat batching, media, `/issue`, retries, and Human
Attribution.

## 14. Cleanup and lifecycle

Application-layer cleanup follows existing channel conventions:

- revoking an installation revokes its active grant and stops new access;
- deleting an installation schedules actor ciphertext cleanup according to
  retention policy rather than cascading immediately;
- workspace deletion explicitly removes dependent grant and actor data in the
  workspace deletion transaction or cleanup job;
- removing the enabling administrator does not rewrite the historical grant;
  a new administrator must re-authorize active access if policy requires it;
- member binding creates `linked_user_id` for the matching actor but does not
  mutate historical message attribution;
- unbinding a member affects only future events.

## 15. Observability and audit events

Recommended activity actions:

- `channel.external_access.enabled`
- `channel.external_access.revoked`
- `channel.external_access.policy_replaced`
- `channel.external_actor.identity_revealed`
- `channel.external_actor.subject_ciphertext_purged`
- `channel.external_actor.retention_extended`

Recommended reason-coded metrics:

- accepted external messages by channel type;
- rejected sender kind;
- rejected tenant mismatch;
- missing/unsupported grant;
- actor persistence or crypto failure;
- strict attribution rejection;
- external `/issue` success/failure;
- media completion/failure;
- ciphertext eligible for expiry and purge failures.

Metrics labels must not include native subjects, actor UUIDs, chat IDs, message
bodies, or filenames.

## 16. Test and regression strategy

The complete design has a medium-to-large regression surface because it changes
the shared channel identity contract and carries new evidence through chat,
tasks, issues, and media. It does not require an undirected full-product manual
regression. The affected surface is bounded and should be verified in six
incremental PRs, followed by the normal full repository suites.

The scenario IDs below identify behavior families, not necessarily one test
function each. The initial catalog contains 154 scenario families. Table-driven
tests should cover equivalent inputs in one canonical layer, so the number of
test functions should be materially lower even though parameterized case count
may be higher. The catalog is distributed across six implementation stages; it
is not a requirement to run every scenario manually for every PR.

### 16.1 Regression scope

| Area | Risk | Required regression |
| --- | --- | --- |
| Feishu identity and access | High | Sender type, tenant, binding, actor, grant, admin authorization |
| Shared channel engine | High | Normalized invocation context, batching, retries, recovery |
| Human Attribution | High | Precise accountability, fail-open/fail-closed, evidence persistence |
| `/issue` and media | High | Atomicity, idempotency, deferred run promotion, classified outcomes |
| Database and migration | Medium-high | Expand migration, legacy null rows, cleanup, rollback |
| Other channel adapters | Medium | Existing member behavior after shared interface changes |
| Settings API and UI | Medium | Capability detection, old client behavior, owner/admin controls |
| Agent runtimes and ordinary product UI | Low | Full suites only unless a shared task contract changes |

### 16.2 Incremental execution plan

| Stage | Implementation scope | Required gate before next stage |
| --- | --- | --- |
| T1 | External actor schema, crypto, retention, and cleanup | Migration, crypto, concurrency, reveal-audit, and cleanup tests pass |
| T2 | Access grant schema, owner/admin API, config projection, and UI | Authorization matrix, API schema, malformed response, and UI permission tests pass |
| T3 | Generic invocation context and message/task persistence | All channel engine suites and member-only adapter regressions pass |
| T4 | Feishu external text chat | Real same-tenant chat, rejection paths, batching, retry, and attribution evidence pass |
| T5 | External `/issue` and media | Text/media atomicity, failure injection, idempotency, and classified visible/silent outcomes pass |
| T6 | Legacy Fork migration and compatibility | Upgrade, mixed-version, old-client, rollback, and retained E2E results pass |

Every stage runs its focused tests while iterating. Before merging a stage, run
the relevant Go packages, `go vet`, sqlc generation/check, affected TypeScript
typecheck/lint/tests, and `git diff --check`. T3 and later also run the full Go
and frontend suites because the shared channel/task contracts have changed.

The catalog has three practical gates:

- **PR gate:** all scenarios owned by the current stage plus regressions for
  contracts that stage changes;
- **feature gate:** the complete functional catalog through COMPAT and all
  channel adapter regressions before the setting is generally available;
- **rollout gate:** retained real-provider E2E, migration timing, rollback drill,
  load/concurrency, fuzzing, and observability checks before broad deployment.

### 16.3 Shared fixtures and assertion rules

The test suite should provide reusable fixtures for:

- workspace owner, administrator, ordinary member, non-admin agent owner, and
  removed former member;
- bound member, unbound same-tenant human, cross-tenant human, bot, app/system,
  missing sender type, and unknown sender type;
- member-only installation, active audited grant, revoked grant, legacy
  `feishu_users`, unsupported policy version, and missing identity key;
- fail-open and `attribution_fail_closed` workspaces;
- direct chat, group mention, group non-mention, topic/thread, text `/issue`,
  and media `/issue`.

Each accepted external-input test must assert more than the visible response:

1. exact external actor persisted;
2. exact grant persisted on the source message and downstream run;
3. `originator_user_id` is null;
4. accountable user and attribution source match the approved policy;
5. connected applications and personal MCP overlays are absent;
6. source message/evidence resolves to the actor and grant;
7. provider native identity is absent from logs and ordinary API payloads.

Each rejected-input test must assert:

1. no runnable task or issue side effect;
2. no session/message persistence unless the design explicitly keeps a rejected
   audit event;
3. dedup state permits or blocks retry according to the documented failure
   class;
4. outcome visibility matches its class: terminal silent drop for bot/app,
   missing/unknown sender, unverifiable/cross-tenant, non-addressed group, and
   duplicate;
   binding/access guidance for an actionable human policy failure; visible
   temporary failure for identity/attribution infrastructure errors;
5. the internal reason code is observable without storing native identity.

### 16.4 Migration and schema scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| DB-001 | Apply the expansion migrations to a current database | New tables and nullable columns exist; existing rows are unchanged | Migration integration |
| DB-002 | Apply the migrations to a fixture from before #6986 | Member bindings, sessions, messages, issues, and tasks remain readable | Migration integration |
| DB-003 | Re-run every up migration | Idempotent migrations do not mutate existing evidence | Migration integration |
| DB-004 | Insert rows with an old-writer column set | Nullable actor/grant columns accept the write | DB-backed |
| DB-005 | Read legacy messages/tasks/issues with null actor/grant | They render as legacy/unattributed, never precise external attribution | Service/API |
| DB-006 | Create two active grants for one installation concurrently | Exactly one succeeds; no ambiguous active policy remains | DB concurrency |
| DB-007 | Create the same external actor concurrently | Exactly one actor is resolved for the scoped HMAC | DB concurrency |
| DB-008 | Use a grant or actor from another workspace/installation | Application transaction rejects the relationship | DB-backed service |
| DB-009 | Interrupt a concurrent index migration and retry | Invalid index is cleaned/rebuilt according to migration-runner rules | Migration integration |
| DB-010 | Run normal binary rollback with expanded schema present | Old binary starts and member writes continue | Compatibility E2E |
| DB-011 | Attempt destructive down migration with retained evidence | Operator warning/gate prevents routine evidence loss | Migration/operations |
| DB-012 | Delete a workspace with actors/grants present | Explicit application cleanup leaves no orphaned workspace data | DB-backed service |
| DB-013 | Adjacent old writer updates an exposure-relevant Agent field | Database trigger advances the capability revision, suspending stale grants | Migration + old-writer integration |

### 16.5 Access grant, API, and UI scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| GRANT-001 | Workspace owner enables audited guest access | Active immutable grant is created and projection updated atomically | Handler + DB |
| GRANT-002 | Workspace administrator enables access | Same success contract as owner | Handler + DB |
| GRANT-003 | Non-admin agent owner enables access | `403`; no grant, config, or audit mutation | Handler + DB |
| GRANT-004 | Ordinary member enables access | `403`; no mutation | Handler + DB |
| GRANT-005 | Admin targets another workspace's installation | Not found/forbidden without leaking installation existence | Handler + DB |
| GRANT-006 | Enable with missing identity key | Action fails visibly; member-only bot remains healthy | Handler/service |
| GRANT-007 | Enable with unsupported identity mode or retention | `400`; no partial grant or config update | Handler schema |
| GRANT-008 | Owner/admin replaces an active policy | Old grant is revoked and a new immutable grant becomes active in one transaction | DB-backed service |
| GRANT-009 | Owner/admin revokes access | New guests are rejected; queued/deferred tasks become ineligible; historical grant remains queryable | Handler + channel + task |
| GRANT-010 | Grant update and incoming message race | Message uses exactly the old or new grant, never a mixed projection | DB/channel concurrency |
| GRANT-011 | Old backend does not advertise capability | New UI hides the audited guest control and keeps the existing member-only management surface | View test |
| GRANT-012 | Non-admin views an enabled installation | Policy is readable as allowed, but control is disabled with explanation | View test |
| GRANT-013 | API returns missing/malformed optional grant fields | Client falls back safely to member-only management state | Core schema test |
| GRANT-014 | Realtime grant update reaches old and new clients | New client refreshes; old client does not crash | API/realtime test |
| GRANT-015 | Repository, workspace MCP, environment, runtime, or filesystem mode changes after enable | Capability revision mismatch suspends external access until owner/admin replaces the grant; member use continues | Service + handler + UI |
| GRANT-016 | Non-admin Agent owner attempts to replace or revoke guest policy | `403`; active grant and projection remain unchanged | Handler + DB |
| GRANT-017 | Non-admin Agent owner manages Bot binding/disconnection | Existing Agent-management permission still succeeds; only guest-policy management is restricted | Handler regression |
| GRANT-018 | Grant is revoked while tasks are queued, deferred, dispatched, and running | Queued/deferred work cannot claim, retries/reruns are denied, and dispatched/running work receives best-effort cancellation without deleting audit evidence | Task lifecycle integration |

### 16.6 External actor, privacy, and retention scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| ACTOR-001 | First recoverable-mode message from a human | Actor stores scoped HMAC and versioned ciphertext, not plaintext | DB + crypto |
| ACTOR-002 | Repeated message from the same installation subject | Existing actor is reused and `last_seen_at` advances | DB-backed service |
| ACTOR-003 | Same native subject on another installation | A distinct scoped actor is created | DB-backed service |
| ACTOR-004 | Pseudonymous-only mode | HMAC is stored and ciphertext remains null | DB + crypto |
| ACTOR-005 | Encrypt identical subjects twice | Ciphertexts differ because nonces are random; lookup HMAC is stable | Crypto unit |
| ACTOR-006 | Ciphertext or associated data is altered | Decryption fails closed | Crypto unit |
| ACTOR-007 | Current and previous encryption-key versions are configured | Existing ciphertext decrypts and new ciphertext uses the current version | Crypto unit |
| ACTOR-008 | Referenced key version is unavailable | Reveal fails without plaintext or silent key fallback | Crypto/service |
| ACTOR-009 | Admin reveals an unexpired identity with a reason | Audit event commits before plaintext is returned | Handler + DB |
| ACTOR-010 | Reveal audit insertion fails | Plaintext is not returned | Failure-injection DB test |
| ACTOR-011 | Non-admin requests reveal | `403`; no decryption or audit side effect | Handler |
| ACTOR-012 | Ciphertext reaches retention expiry | Ciphertext/key version are removed; actor UUID/HMAC/evidence remain | Cleanup job |
| ACTOR-013 | Open issue or legal hold extends retention | Cleanup skips ciphertext and records the extension reason | Cleanup + DB |
| ACTOR-014 | An actor whose ciphertext was purged sends another message | Retained HMAC resolves the same actor; recoverable mode writes fresh ciphertext only when the configured retention policy permits it, while pseudonymous-only mode keeps ciphertext null | DB-backed service |
| ACTOR-015 | External actor later binds a member account | Link is recorded; old events stay external and new events use member authorization | Integration |
| ACTOR-016 | Logs, metrics, API, and activity payloads are inspected | No native subject, ciphertext, message body, or secret is exposed | Security regression |
| ACTOR-017 | Lookup-HMAC key is rotated | Retained keys resolve old actors, matches are lazily reindexed, and an old key cannot be retired while unmatched pseudonymous actors still require correlation | Crypto + DB integration |

### 16.7 Sender authorization scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| AUTH-001 | Current bound workspace member sends a message | Existing member authorization and attribution remain unchanged | Channel integration |
| AUTH-002 | Binding exists, membership was removed, and an active audited grant exists | Stale member authority is ignored; after the full tenant/actor/grant/attribution preflight succeeds, the sender is admitted only as an external actor | Identity resolver + DB |
| AUTH-003 | Same-tenant unbound human with active grant | External invocation context is accepted | Identity resolver + DB |
| AUTH-004 | Same-tenant unbound human in member-only mode | Rejected before message/task side effects | Identity resolver |
| AUTH-005 | Cross-tenant human | Silently rejected regardless of active grant | Identity resolver |
| AUTH-006 | Persisted installation tenant is missing but authenticated event tenant matches sender tenant | The event-header tenant becomes the effective installation tenant; the sender is accepted after the same grant, actor, capability-revision, and attribution checks as AUTH-003 | Identity resolver |
| AUTH-007 | Sender tenant is missing | Silently rejected before actor/message/task persistence | Identity resolver |
| AUTH-008 | Exact sender type is `bot` | Silently rejected before actor persistence; no reply loop | Decoder/resolver |
| AUTH-009 | Sender type is app/system | Silently rejected before actor persistence; no reply loop | Decoder/resolver |
| AUTH-010 | Sender type is missing | Silently rejected before actor persistence | Decoder/resolver |
| AUTH-011 | Sender type is unknown | Silently rejected before actor persistence | Decoder/resolver |
| AUTH-012 | Grant is revoked concurrently with decode/preflight | If revoke commits first, preflight rejects; if message acceptance commits first, it persists under the old grant and the revoke gate makes downstream work ineligible. Neither interleaving substitutes a replacement grant | Channel concurrency |
| AUTH-013 | Policy version is unsupported | Rejected and reported to sender/admin | Resolver/service |
| AUTH-014 | Actor persistence or identity crypto fails | Rejected with no runnable side effect and a visible response | Failure injection |
| AUTH-015 | Persisted installation tenant conflicts with authenticated event tenant | Silently rejected even if sender matches one of them | Identity resolver |
| AUTH-016 | Persisted and authenticated event tenant are both missing | Silently rejected before actor/message/task persistence; sender tenant alone cannot establish installation ownership | Identity resolver |
| AUTH-017 | Exact human sender has an empty native subject/open ID | Silently rejected before HMAC lookup; empty identities never collapse into one actor | Decoder/resolver |
| AUTH-018 | Binding exists, membership was removed, and installation is member-only | Rejected before actor/message/task persistence with visible access guidance; the stale binding grants no authority | Identity resolver + replier |

### 16.8 Chat, batching, retry, and recovery scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| CHAT-001 | External direct-chat text message | Session technical creator is the installer; message/task link to the exact external actor, grant, and approved Agent capability revision | DB-backed channel |
| CHAT-002 | External group mention | Mention is removed correctly and task uses external context | Channel integration |
| CHAT-003 | Group non-mention when mentions are required | Ignored without task creation | Channel integration |
| CHAT-004 | Topic/thread message | Correct composite binding and source evidence are preserved | Channel integration |
| CHAT-005 | Duplicate provider message | One durable message and at most one task are created | Dedup integration |
| CHAT-006 | Same actor sends several messages in one debounce window | One context generation produces one debounced task containing the messages in durable order; every source message keeps actor/grant | Batcher + DB |
| CHAT-007 | Two external actors share one debounce window | Task input preserves both message actors and denies personal overlays | Batcher + DB |
| CHAT-008 | Bound member and external actor arrive in one debounce interval | Authorization boundary flushes/advances the generation; they do not share one task | Batcher + DB |
| CHAT-009 | Grant is replaced during debounce | Generation is flushed/advanced; one task never mixes grants | Batcher + DB |
| CHAT-010 | Process stops after message append before enqueue | Recovery creates at most one task from persisted context | Recovery integration |
| CHAT-011 | Process stops after enqueue before provider reply | Retry does not create a second task and can send the correct outcome | Recovery integration |
| CHAT-012 | Task retries after grant revocation | Retry is denied with a visible revoked-access outcome; it cannot use a replacement grant or broader current capabilities | Task retry |
| CHAT-013 | Archived/offline agent or runtime | Sender receives the established visible unavailable response | Router integration |
| CHAT-014 | Attribution/actor infrastructure failure | Sender receives a visible security-safe failure response | Router failure injection |
| CHAT-015 | Legacy member chat session continues after upgrade | Session history and generation-1 fallback remain valid | Upgrade integration |
| CHAT-016 | Agent response is streamed/patched to Feishu | Existing outbound card/message behavior remains unchanged | Channel E2E |
| CHAT-017 | External user sends bare `/new`, then normal text | No empty task is created; the next external task starts a fresh context under the same grant | Channel + recovery |
| CHAT-018 | A sender is bound/unbound or gains/loses workspace membership inside one debounce window | The authorization transition flushes/advances the generation; member-authorized and external-authorized messages never share a task | Batcher + identity integration |

### 16.9 Attribution and capability scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| ATTR-001 | Bound member invocation | Originator and accountable user both equal the member | Task service + DB |
| ATTR-002 | Strict workspace receives an external invocation with complete precise evidence | Originator/authorization user are null; accountable admin, source `external_grant`, exact channel-message evidence, actor, grant, and approved capability revision are persisted | Task service + DB |
| ATTR-003 | Strict workspace receives an external invocation with missing/invalid actor, grant, source message, tenant evidence, or capability revision | Rejected before task enqueue with a visible security-safe response; no owner fallback or admin permission borrowing occurs | Task/router |
| ATTR-004 | Fail-open workspace with complete external context | `ExternalGrant` attribution is used; no `unattributed` or `owner_fallback` transition | Task service |
| ATTR-005 | Missing external actor or grant reaches enqueue boundary | Enqueue refuses rather than degrading to owner fallback | Task service |
| ATTR-006 | Delegated task originates from an external channel task | Authorization remains null and restrictive capability context propagates | Task delegation |
| ATTR-007 | Agent mentions another agent from external-origin task | A2A permission does not treat accountable admin as originator | Task authorization |
| ATTR-008 | External task resolves connected applications | Personal Composio/connected-app overlay is empty | Task execution |
| ATTR-009 | External task resolves Agent/workspace MCP, environment, repository, and filesystem capabilities at the approved revision | It receives no member-scoped overlay; read/write behavior matches the admin-approved Agent revision | Task execution |
| ATTR-010 | Retry, rerun, or merge is requested while grant is active | External evidence, grant, and approved Agent capability revision are preserved; revoked grants are covered by CHAT-012/GRANT-018 | Task lifecycle |
| ATTR-011 | Agent capability revision changes after an external task is queued but before claim/retry | Task fails visibly instead of executing with the new capability set; no silent widening occurs | Task claim/retry |

### 16.10 `/issue` text scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| ISSUE-001 | Valid external text `/issue` | Issue creator is the receiving Agent; exact `origin_message_id`, actor, grant, and assigned task commit consistently | DB-backed E2E |
| ISSUE-002 | Command omits required title/content | No issue is created; sender receives usage guidance | Router/service |
| ISSUE-003 | Duplicate command event | At most one issue and assigned task exist | Idempotency integration |
| ISSUE-004 | Actor/grant preflight fails | No issue or task is created | Failure injection |
| ISSUE-005 | Issue insert fails | Already-accepted source message remains as failed-command evidence; no issue/task exists and no success reply is sent | Failure injection |
| ISSUE-006 | Assigned task enqueue fails before the external `/issue` transaction commits | Issue and task both roll back; source command remains failed-attempt evidence and no success reply is sent | Failure injection |
| ISSUE-007 | Fail-open workspace | Issue/task use `ExternalGrant` attribution: null originator/authorization user, accountable enabler, and exact actor/grant/message evidence | DB-backed service |
| ISSUE-008 | Fail-closed workspace with complete external evidence | The operation uses the precise `ExternalGrant` contract from ATTR-002; incomplete evidence rejects before issue/task creation as ATTR-003, with no owner fallback or partial success | DB-backed service |
| ISSUE-009 | External issue is loaded by old/new clients | Creator remains the agent and origin fields parse safely | API/core schema |
| ISSUE-010 | Workspace member uses `/issue` | Existing member creator/origin/task behavior remains unchanged | Regression E2E |
| ISSUE-011 | External `/issue` matches an active duplicate, with and without media | Existing issue is returned as the terminal result; no new task, issue, download, or attachment is created | Router/media integration |

### 16.11 Media scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| MEDIA-001 | External chat file with supported type/size | File binds to source message and run under the same actor/grant | Media integration |
| MEDIA-002 | External `/issue` with one file | Issue and deferred task finalize after the attachment is durable | DB-backed E2E |
| MEDIA-003 | External `/issue` with multiple files | All expected attachments resolve idempotently before promotion | Media integration |
| MEDIA-004 | File event is duplicated | One file/attachment effect is produced | Dedup integration |
| MEDIA-005 | Provider download is unauthorized/expired after issue acceptance | Placeholder is finalized and deferred task is promoted by existing fallback; no duplicate issue/attachment is created and failure is observable | Failure injection |
| MEDIA-006 | Download times out or process restarts | Remote download is not falsely resumed; persisted deadline finalizes placeholder and promotes exactly one task under the original actor/grant | Recovery integration |
| MEDIA-007 | File exceeds size or type policy after message acceptance | Binary is not stored/attached or exposed to the Agent; placeholder is finalized and exactly one text/placeholder task becomes runnable | Media guard |
| MEDIA-008 | Scan/storage step fails after issue acceptance | Unsafe/unavailable binary is not attached; placeholder fallback prevents the deferred task from remaining stuck | Failure injection |
| MEDIA-009 | Attachment bind fails after upload | No duplicate attachment is created; intent ledger/reconciler owns any unbound object, and the persisted deadline finalizes the placeholder and promotes exactly one task | Failure injection |
| MEDIA-010 | Grant is revoked during asynchronous media processing | A media/storage transaction that committed before revocation remains auditable and reconciled; after revocation the deferred task cannot promote/claim and new messages are rejected | Media/task integration |
| MEDIA-011 | Fail-open and fail-closed workspace variants with complete external evidence | Both accept under the same precise `ExternalGrant` contract; strict mode rejects any incomplete evidence before media/issue/task side effects | DB-backed matrix |
| MEDIA-012 | Bound member file flow | Existing channel media behavior is unchanged | Regression E2E |
| MEDIA-013 | Multiple-file message has partial download/upload failure | Successful files attach once, failed files remain placeholders, and one deferred task is promoted | Media integration |
| MEDIA-014 | Accepted media later reaches timeout/download/upload/bind fallback | No separate provider warning is sent; operators retain the failure reason and the eventual task response is the only normal outbound result | Media + outbound regression |

### 16.12 Compatibility and rollback scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| COMPAT-001 | New server reads an installation with no access-mode key | Normalizes to member-only | Store/unit |
| COMPAT-002 | New server reads legacy `feishu_users` | Disables guest access pending explicit re-authorization | Store/migration |
| COMPAT-003 | Old #6986 server reads `feishu_users_audited` | Unknown value normalizes to member-only | Old-version fixture/binary |
| COMPAT-004 | New server reads unsupported future policy version | Fails closed | Store/service |
| COMPAT-005 | New client parses old installation response | Safe defaults and hidden control | Core schema/view |
| COMPAT-006 | Old official client receives new optional response fields | Ordinary member UI remains functional | Supported-client fixture/E2E |
| COMPAT-007 | Old #6986 client receives v2 mode | It does not crash; legacy PATCH is rejected with actionable upgrade guidance | Compatibility E2E |
| COMPAT-008 | New UI connects to old server | Capability detection hides guest management | View/E2E |
| COMPAT-009 | New server starts without identity key | Server and member bots start; guest readiness is false | Startup integration |
| COMPAT-010 | Old writer inserts member chat/task rows into expanded schema | Write succeeds and new server reads them as legacy | DB integration |
| COMPAT-011 | Old hub owns lease while v2 config exists | Unbound sender is rejected; no unaudited run | Mixed-version E2E |
| COMPAT-012 | New hub takes lease after full deployment | Audited external flow begins without duplicate event consumption | Mixed-version E2E |
| COMPAT-013 | Planned rollback after revoking grants | Member flow remains operational | Docker rollback drill |
| COMPAT-014 | Emergency rollback with an active v2 grant | Guest flow becomes unavailable but cannot bypass audit | Docker rollback drill |

### 16.13 Other channel regression scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| CHANNEL-001 | Slack bound member chat and `/issue` | Existing identity, task, reply, and origin behavior pass | Adapter regression |
| CHANNEL-002 | DingTalk bound member chat and participant profile | Existing binding/profile semantics remain unchanged | Adapter regression |
| CHANNEL-003 | WeCom bound member text and media | Existing identity, media, and task behavior pass | Adapter regression |
| CHANNEL-004 | Telegram bound member chat and media | Existing identity, origin, and task behavior pass | Adapter regression |
| CHANNEL-005 | Adapter does not implement external-actor capability | Unbound user remains rejected by default | Engine contract |
| CHANNEL-006 | Shared batcher receives only member contexts | Output is byte/semantically equivalent to pre-feature behavior | Engine regression |

### 16.14 Cleanup, audit, and observability scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| OPS-001 | Enable, replace, and revoke grant | Immutable activity events name acting admin and grant without native subject | DB/service |
| OPS-002 | Accepted external message | Success metric increments with bounded non-identity labels | Metrics test |
| OPS-003 | Tenant/sender/grant/crypto rejection | Correct stable reason code is logged and counted | Router/metrics |
| OPS-004 | Identity reveal and ciphertext purge | Required audit events exist and contain no plaintext | DB/service |
| OPS-005 | Cleanup job retries after failure | Work is idempotent and evidence is not partially removed | Job integration |
| OPS-006 | Installation is revoked | Active grant closes atomically; recoverable ciphertext remains bounded by its recorded expiry/legal hold and pseudonymous actor/message/grant evidence remains available for audit | Service/DB |
| OPS-007 | Enabling administrator is demoted or leaves workspace | Historical accountability and active grant remain unchanged until a current owner/admin explicitly replaces or revokes it | Service/DB |
| OPS-008 | Operator inspects health/readiness | Guest-specific key/policy problems are actionable without exposing secrets | Handler/startup |

### 16.15 Non-functional scenarios

| ID | Scenario | Expected result | Layer |
| --- | --- | --- | --- |
| NFR-001 | Concurrent first messages from many external actors | No duplicate actors/grants and acceptable lock contention | Load/concurrency |
| NFR-002 | High-volume repeat messages from known actors | HMAC lookup uses the intended index and avoids sequential scans | Query plan/load |
| NFR-003 | Large chat/task tables receive nullable columns | Migration lock duration stays within the rollout budget | Staging migration |
| NFR-004 | Concurrent grant switch and message ingress | No deadlock; lock ordering matches channel context rules | DB concurrency |
| NFR-005 | Secrets and native identifiers are injected into errors | Redaction prevents them from reaching logs/provider replies | Security test |
| NFR-006 | Malformed ciphertext, JSON policy, and provider payloads are fuzzed | No panic, privilege widening, or plaintext leakage | Fuzz/unit |

### 16.16 Positive/negative semantic coverage review

This cross-check prevents a common test-plan failure: writing many rejection
tests without proving the intended positive path, or asserting “no side
effect” where the product deliberately preserves an accepted message, issue,
or media placeholder. “Negative” below means the inverse or exceptional
business path; it does not always mean total rollback.

| Business rule | Positive coverage | Inverse/negative coverage | Semantic verdict |
| --- | --- | --- | --- |
| Only workspace owner/admin may expose an Agent | GRANT-001, GRANT-002, GRANT-008, GRANT-009 | GRANT-003 through GRANT-005, GRANT-016 | Complete: Agent ownership alone is insufficient; GRANT-017 preserves ordinary Bot management |
| Exact same-tenant human requirement | AUTH-003, AUTH-006 | AUTH-005, AUTH-007 through AUTH-011, AUTH-015 through AUTH-017 | Complete: missing/unknown values never infer human, tenant, or identity |
| Bound-member priority without stale privilege | AUTH-001 | AUTH-002, AUTH-004, AUTH-018 | Complete: removed member qualifies only through a fresh external-grant decision and is rejected in member-only mode |
| Active immutable grant required | GRANT-001, AUTH-003 | GRANT-009, AUTH-012, AUTH-013 | Complete: current config alone cannot authorize a guest |
| Privacy-safe stable actor | ACTOR-001 through ACTOR-005 | ACTOR-006, ACTOR-008, AUTH-014 | Complete: correlation succeeds; tamper/key/persistence failure rejects |
| Lookup and encryption key lifecycle | ACTOR-007, ACTOR-017 | ACTOR-008 and old-key retirement guard in ACTOR-017 | Complete after adding lookup-key version to the model |
| Reversible identity retention is bounded | ACTOR-009, ACTOR-013, ACTOR-014 | ACTOR-010 through ACTOR-012 | Complete: ciphertext purge is not falsely described as anonymization |
| Member-scoped capabilities never transfer | ATTR-008, ATTR-009 | ATTR-005, ATTR-007 | Complete: Agent capabilities remain as admin approved; personal overlays remain absent |
| Agent capability changes cannot silently widen a grant | GRANT-001, ATTR-009 | DB-013, GRANT-015, ATTR-011 | Complete after adding trigger-backed capability revision: old/new writers suspend the grant and queued work fails rather than using an unapproved revision |
| Grant revocation behaves as a kill switch | GRANT-001, ATTR-010 | GRANT-009, GRANT-018, CHAT-012, MEDIA-010 | Complete: new/queued/retry work stops, running cancellation is best-effort, historical evidence remains |
| One task has one authorization model | CHAT-006, CHAT-007 | CHAT-008, CHAT-009, CHAT-018 | Complete after splitting member↔external, identity/membership, and grant-change boundaries |
| Retry/recovery cannot widen authority | CHAT-010 through CHAT-012, ATTR-010 | AUTH-012, CHAT-014 | Complete: accepted context is durable; failed preflight cannot fall back |
| Strict attribution recognizes precise external evidence | ATTR-002, ATTR-004 | ATTR-003, ATTR-005 | Complete: exact actor + immutable grant + source message + approved capability revision satisfies fail-closed attribution; incomplete evidence rejects without owner fallback |
| External text `/issue` is atomic after source-message acceptance | ISSUE-001, ISSUE-007, ISSUE-008 | ISSUE-004 through ISSUE-006 | Complete: failed command message may remain as evidence, but issue/task cannot partially succeed |
| Duplicate `/issue` is a terminal product result | ISSUE-003, ISSUE-011 | ISSUE-004 through ISSUE-006 | Complete: duplicate is not infrastructure failure and must not download duplicate media |
| Accepted media follows existing fallback | MEDIA-001 through MEDIA-003, MEDIA-013 | MEDIA-005 through MEDIA-009, MEDIA-014 | Complete after correction: post-acceptance media failure finalizes placeholder and unblocks task without an extra provider warning; remote jobs are not falsely described as resumable |
| Rejected or duplicate events are idempotent | CHAT-005, ISSUE-003, MEDIA-004 | Shared rejected-input assertions and COMPAT-011 | Complete: security rejection is terminal; retryable infrastructure failure releases the claim |
| New/old version boundary fails closed | COMPAT-001 through COMPAT-010 | COMPAT-011 through COMPAT-014 | Complete: availability may fall back, audit cannot be bypassed |
| Other channel member behavior remains unchanged | CHANNEL-001 through CHANNEL-004, CHANNEL-006 | CHANNEL-005 | Complete: adapters without external capability stay member-only |
| Failure is observable without identity leakage | OPS-001 through OPS-004, OPS-008 | ACTOR-016, NFR-005 | Complete: reason codes are visible internally, native identity is not |
| Unsafe/non-actionable senders do not trigger reply loops | Actionable human guidance in AUTH-004, AUTH-013, AUTH-014, CHAT-013, CHAT-014 | Silent drops in AUTH-005, AUTH-008 through AUTH-011, AUTH-015, AUTH-017, CHAT-003, CHAT-005 | Complete: visibility follows business and anti-loop semantics rather than “always reply” |

Post-acceptance media fallback is now decided: preserve and run the
text/placeholder path, retain the operator-visible failure reason, and do not
send a separate provider warning. A future cancellation or second-warning
policy requires a separate product decision and regression update.

### 16.17 Retained end-to-end evidence

Each implementation PR must attach or link test results. T4-T6 should retain
provider-side screenshots/message IDs and database assertions with sensitive
identifiers redacted.

| Scenario | Environment/commit | Expected | Result | Evidence |
| --- | --- | --- | --- | --- |
| Same-tenant external chat | | Accepted, actor/grant linked, no personal apps | | |
| Two guests under one grant in one batch | | Both actors retained; one grant and approved Agent revision | | |
| Member then guest in one debounce interval | | Authorization boundary produces separate task generations | | |
| Cross-tenant human | | Rejected before task | | |
| Bot, missing, and unknown sender | | Rejected before actor/task | | |
| External `/issue` text | | Issue/task/source atomically linked | | |
| External `/issue` media | | Attachment and deferred run finalized atomically | | |
| Fail-closed workspace | | Complete external evidence is accepted precisely; incomplete evidence rejects atomically | | |
| Legacy `feishu_users` upgrade | | Disabled until admin re-authorizes | | |
| Planned binary rollback | | Member flow works, guest flow fails closed | | |
| Slack/DingTalk/WeCom/Telegram smoke | | Existing member flows unchanged | | |

## 17. Rollout plan

1. **Decision:** strict attribution accepts complete precise external evidence;
   maintainers confirm the remaining identity storage modes, retention
   defaults, and generic-vs-Feishu scope.
2. **Schema expansion:** add tables, nullable columns, queries, and crypto
   readiness without exposing an enable control.
3. **Read path:** introduce invocation context and preserve existing member
   behavior; ship recovery and compatibility tests.
4. **Write path:** persist actors/grants for a disabled internal feature flag;
   validate chat, media, and `/issue` atomicity.
5. **Admin surface:** add dedicated enable/revoke APIs, UI explanation, and
   activity events.
6. **Fork migration:** disable legacy `feishu_users` and require explicit v2
   re-authorization.
7. **Feishu rollout:** enable for selected self-hosted installations, observe
   rejection and cleanup metrics, then widen availability.
8. **Other channels:** reuse the generic model only after each adapter can
   provide exact human sender type and tenant/account boundary evidence.

## 18. Alternatives considered

### Store raw `open_id` in `channel_user_binding`

Rejected. It overloads a workspace-member authorization table, stores identity
in plaintext, and makes existing membership queries ambiguous.

### Use the installer as the external caller

Rejected. It is useful only as technical ownership. It cannot distinguish
callers and risks inheriting user-scoped capabilities.

### Store only `external_actor_id` on the task

Rejected. A debounced task can contain messages from several people. Message
attribution plus a task policy/evidence anchor preserves the real input graph.

### Use `activity_log` as the active policy

Rejected. Runtime authorization needs an atomic, indexed source of truth, while
activity data is an append-only audit projection.

### Keep `feishu_users` for v2

Rejected. Old Fork binaries understand that value and would accept external
callers without v2 actor/grant persistence after rollback.

### Delete all identity after 180 days

Rejected. It would make historical task and issue attribution impossible. The
design instead expires reversible ciphertext while preserving pseudonymous
actor and immutable grant evidence.

### Remove the channel sandbox or give guests installer capabilities

Rejected. Registration friction does not justify transferring a member's
credentials or personal integrations to an unbound caller.

## 19. Open decisions

1. Should `encrypted_recoverable` be the default, or should deployments choose
   explicitly before enabling access?
2. What are the supported retention bounds and the default reversal window?
3. Should the initial capability profile be fixed in code or stored as a
   versioned, server-validated grant snapshot?
4. Should identity crypto derive from an existing installation secret or use a
   dedicated operator keyring? A dedicated versioned keyring is recommended.
5. Is the first implementation intentionally Feishu-only at the API/UI layer
   while using channel-neutral storage, or should the feature wait for a shared
   channel settings surface?

## 20. Recommended PR split

To keep review and rollback tractable:

1. **Design and compatibility contract** — this document and maintainer
   decisions.
2. **External actor storage and crypto** — schema, service, cleanup, reveal
   audit, and database tests.
3. **Access grants and admin API** — immutable policy, authorization, config
   projection, capability detection, and UI.
4. **Channel invocation context** — chat/task persistence, debounce/recovery,
   Human Attribution integration, and text E2E tests.
5. **`/issue` and media completion** — source linkage, transactional behavior,
   failure replies, and media E2E tests.
6. **Legacy Fork migration and rollout** — `feishu_users` shutdown,
   re-authorization notice, rollback drill, and operator documentation.

This split allows maintainers to accept the generic identity and grant model
without coupling it to every Feishu product surface in one review, while the
version fence prevents partially-deployed code from broadening access.
