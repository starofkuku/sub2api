# Agent Identity Format Analysis

## Scope

This document analyzes the OpenAI/Codex Agent Identity import and runtime
authentication implementation in Sub2API as of upstream version `v0.1.163`.

Agent Identity is not a normal OAuth access-token flow. It imports a long-lived
Ed25519 private key plus an agent runtime identity. The service dynamically
signs upstream requests and maintains an upstream task identifier rather than
storing an OAuth `access_token` or `refresh_token`.

This document intentionally uses placeholders only. Do not paste real
credentials, private keys, tokens, or complete `auth.json` files into issue
trackers, chat logs, or source control.

## User Interface Entry Point

The UI entry is:

1. Open the admin console.
2. Open **Account Management**.
3. Select **Create Account**.
4. Select platform **OpenAI** and set normal account options such as name,
   proxy, concurrency, priority, model mapping, and groups.
5. Continue to the authorization step.
6. Select **Agent Identity auth.json** as the input method.
7. Paste the Agent Identity JSON and submit.

The specialized input is part of the Codex credential import flow. It is not
part of **Account Management -> Import Data**, which is a Sub2API backup/data
restore feature.

Frontend implementation:

- `frontend/src/components/account/CreateAccountModal.vue`
- `frontend/src/components/account/OAuthAuthorizationFlow.vue`

The frontend submits to:

```text
POST /api/v1/admin/accounts/import/codex-session
```

The shared endpoint supports conventional Codex OAuth session imports and
Agent Identity imports. The selected input method determines the client-side
validation and the expected format.

## Accepted Input Format

The recommended format is a JSON object with an Agent Identity marker and a
nested identity object:

```json
{
  "auth_mode": "agentIdentity",
  "agent_identity": {
    "agent_runtime_id": "<runtime-id>",
    "agent_private_key": "<base64-pkcs8-ed25519-private-key>",
    "task_id": "<optional-task-id>",
    "account_id": "<chatgpt-account-id>",
    "chatgpt_user_id": "<chatgpt-user-id>",
    "email": "optional@example.invalid",
    "plan_type": "optional-plan",
    "chatgpt_account_is_fedramp": false
  }
}
```

The importer accepts both snake_case and camelCase names:

| Snake case | Camel case |
| --- | --- |
| `auth_mode` | `authMode` |
| `agent_identity` | `agentIdentity` |
| `agent_runtime_id` | `agentRuntimeId` |
| `agent_private_key` | `agentPrivateKey` |
| `task_id` | `taskId` |
| `account_id` | `accountId` |
| `chatgpt_user_id` | `chatgptUserId` |
| `plan_type` | `planType` |
| `chatgpt_account_is_fedramp` | `chatgptAccountIsFedramp` |

A flat form is also accepted when `auth_mode` or `authMode` is `agentIdentity`:

```json
{
  "authMode": "agentIdentity",
  "agentRuntimeId": "<runtime-id>",
  "agentPrivateKey": "<base64-pkcs8-ed25519-private-key>",
  "accountId": "<chatgpt-account-id>",
  "chatgptUserId": "<chatgpt-user-id>"
}
```

The presence of an `agent_identity` or `agentIdentity` object is sufficient to
select the Agent Identity parsing branch. The explicit `auth_mode` marker is
still recommended because it makes the artifact unambiguous.

### Required Fields

All of the following are required:

- `agent_runtime_id`
- `agent_private_key`
- `account_id`
- `chatgpt_user_id`

### Optional Fields

- `task_id`: existing upstream task identifier. If omitted, the first runtime
  operation registers a new task.
- `email`: account display and metadata only.
- `plan_type`: account metadata only.
- `chatgpt_account_is_fedramp`: enables the corresponding upstream header.

### Private Key Encoding

`agent_private_key` must be:

1. Standard Base64 encoded.
2. The DER bytes of a PKCS#8 private key.
3. An Ed25519 private key.

PEM text, a public key, another private-key algorithm, or non-Base64 content
is rejected. The importer validates the format without returning or logging
the key material.

## Parsing and Import Pipeline

### 1. Frontend Structural Validation

When **Agent Identity auth.json** is selected, the frontend accepts content
only when it parses as JSON and each submitted value looks like Agent Identity
input. It checks for either:

- `auth_mode` / `authMode` equal to `agentIdentity`, or
- an `agent_identity` / `agentIdentity` object.

The UI is designed for one JSON object, but the backend parser can process:

- a JSON object;
- a JSON array;
- multiple JSON values in a stream; and
- multiple nonempty lines, where each line is a JSON object.

For UI use, an object or an array is the clearest format. The generic endpoint
can be used for controlled batch imports, but batch credentials should be
handled with the same protections as private keys.

### 2. Backend Format Validation

The backend parses the request body into entries, then normalizes every entry.
An Agent Identity entry is selected when it contains the marker or nested
identity object described above.

The backend then:

1. Extracts required and optional fields with snake_case/camelCase aliases.
2. Rejects missing required values.
3. Parses and verifies the private key as PKCS#8 Ed25519.
4. Creates normalized credentials and account metadata.
5. Applies the first-step UI account configuration: proxy, groups,
   concurrency, priority, rate multiplier, load factor, model mapping, and
   other non-protected extras.

The import validates syntax and cryptographic key type. It does not prove that
the upstream account is usable at import time. Run an account connection test
after importing.

### 3. Persisted Account Representation

The imported account is created as:

```text
platform = openai
type     = oauth
```

The type is `oauth` for compatibility with the OpenAI/Codex dispatch path, but
its credentials are not OAuth credentials. The normalized credential document
is conceptually:

```json
{
  "auth_mode": "agentIdentity",
  "agent_runtime_id": "<runtime-id>",
  "agent_private_key": "<base64-pkcs8-ed25519-private-key>",
  "task_id": "<optional-task-id>",
  "chatgpt_account_id": "<account-id>",
  "chatgpt_user_id": "<user-id>",
  "chatgpt_account_is_fedramp": false,
  "email": "optional@example.invalid",
  "plan_type": "optional-plan"
}
```

The importer deliberately does not write `access_token` or `refresh_token`.
The normal OAuth expiry policy is skipped for this mode: the account is not
auto-paused based on a bearer-token expiry because it has no bearer-token
lifetime to track.

The importer attaches metadata similar to:

```json
{
  "import_source": "codex_session",
  "imported_at": "<RFC3339 timestamp>"
}
```

### 4. Protected Import Fields

The create-account page can attach non-sensitive credential extras such as
model mappings. It cannot override identity-sensitive fields through those
extras. The protected set includes:

- tokens and token expiry fields;
- `auth_mode`;
- `chatgpt_account_id` and `chatgpt_user_id`;
- `agent_runtime_id`;
- `agent_private_key`; and
- `task_id`.

This prevents a generic UI configuration field from silently replacing the
identity material extracted from the submitted `auth.json`.

## Deduplication and Update Semantics

The UI sends `update_existing: true` for this import flow.

Agent Identity uses `chatgpt_account_id` as its primary account identity key.
The importer also protects against cross-user collisions:

- Same ChatGPT account and same user: import updates the existing account.
  New runtime ID, private key, and task ID replace the old values.
- Same ChatGPT account but different user IDs: entries are kept separate.
- Same user in different ChatGPT Team/Account workspaces: entries are kept
  separate.
- Repeated matching entries in one batch are skipped as duplicates.

Runtime IDs are intentionally not the durable deduplication key because a
runtime may change for the same underlying account.

## Runtime Authentication Flow

### Authentication Mode Detection

An account is treated as Agent Identity only when all of the following apply:

```text
platform = openai
type = oauth
credentials.auth_mode equals agentIdentity, case-insensitively
```

Normal OpenAI OAuth, Codex PAT, and API-key accounts retain Bearer-token
authentication.

### Task Registration

An Agent Identity account requires a task ID. Before signing an upstream
request, Sub2API checks the stored `task_id`.

- If a task ID exists, it is used.
- If it is absent, Sub2API registers a task with the Agent Identity
  authentication service.
- The account proxy is used for task registration when the account has one.
- The registration request has a 30-second total timeout and a 15-second
  response-header timeout.

The registration request is equivalent to:

```text
POST https://auth.openai.com/api/accounts/v1/agent/{agent_runtime_id}/task/register
Content-Type: application/json
Accept: application/json
```

The request body contains:

```json
{
  "timestamp": "<RFC3339 UTC timestamp>",
  "signature": "<Base64 Ed25519 signature>"
}
```

The signature input is:

```text
agent_runtime_id:timestamp
```

The registration response may contain either:

```json
{ "task_id": "..." }
```

or camelCase `taskId`. It can also return `encrypted_task_id` or
`encryptedTaskId`. For encrypted task IDs, the implementation derives an
X25519 key pair from the Ed25519 private-key seed and decrypts the sealed-box
payload. The resulting task ID is persisted to the account credentials.

### Per-Request Assertion

Every upstream request builds a fresh `Authorization` value:

```text
Authorization: AgentAssertion <base64url-json-envelope>
```

The decoded envelope is:

```json
{
  "agent_runtime_id": "<runtime-id>",
  "task_id": "<task-id>",
  "timestamp": "<RFC3339 UTC timestamp>",
  "signature": "<Base64 Ed25519 signature>"
}
```

The assertion signature input is:

```text
agent_runtime_id:task_id:timestamp
```

The private key is never placed in the upstream authorization header.

Sub2API also applies normal ChatGPT account headers for OpenAI OAuth-path
accounts:

- `chatgpt-account-id` when `chatgpt_account_id` is available;
- `x-openai-fedramp: true` when the FedRAMP flag is enabled.

Session isolation, sticky-session behavior, and prompt-cache handling remain
on the normal OpenAI OAuth path.

## Task Failure Recovery

A generic 401 does not automatically cause task rotation. Recovery is limited
to a 401 whose body identifies an invalid task, including markers such as:

- `invalid_task_id`
- `task_not_found`
- `task_expired`
- equivalent human-readable task-invalid messages

When one of those failures occurs:

1. The currently stored task ID is treated as the expected stale task.
2. A new task is registered.
3. The new task ID is written to the database.
4. Existing WebSocket connections for the account are invalidated.
5. The current request is retried once with a new assertion.

Recovery is intentionally limited to one retry per request to prevent a
permanent invalid-task condition from becoming an unbounded loop.

The same behavior is wired into:

- OpenAI Responses forwarding;
- OpenAI Chat Completions forwarding;
- Anthropic Messages to OpenAI compatibility forwarding;
- Codex model manifest loading;
- account connection tests;
- quota operations; and
- Responses WebSocket connection setup.

## Scheduling and Model Behavior

Agent Identity accounts participate in the standard OpenAI OAuth scheduling
and model mapping path. They must still be:

- active;
- schedulable;
- in the target group when the API key is group-bound;
- compatible with the requested model; and
- outside rate-limit, overload, expiry, and temporary-unschedulable windows.

The authentication mode does not bypass group selection, account concurrency,
model restrictions, or billing. It only replaces the upstream authentication
mechanism.

## Security Analysis

### Protections Present in the Application

The implementation includes several protections:

- `agent_private_key` is classified as a sensitive credential key.
- Account API responses redact the private key and expose only a presence
  indicator.
- The Agent Identity import request body is omitted from audit-log capture,
  because the entire payload is credential material.
- Upstream error text is defensively redacted for private key, runtime ID,
  task ID, tokens, and `AgentAssertion` values before it can reach logs,
  operations events, or returned errors.
- Private-key parsing is strict: Base64, PKCS#8, and Ed25519 are all required.
- Per-account in-process locks and a database re-read reduce duplicate task
  registration during concurrent requests in one process.

### Storage Risk

The `accounts.credentials` column is PostgreSQL JSONB. The code path stores
credentials as JSONB and no field-level encryption layer is present in the
analyzed implementation.

That means the private key must be treated as recoverable credential material
by anyone with direct database, database-backup, or raw account-export access.
The generic account data export also serializes account credentials, so exports
can contain Agent Identity private keys.

Recommended controls:

1. Restrict database access to the smallest operational set.
2. Encrypt database volumes and backups.
3. Encrypt exported account backups before storing or transferring them.
4. Do not commit exports, `auth.json`, SQL dumps, or screenshots containing
   credentials to Git.
5. Restrict administrator roles that can export accounts or modify credentials.
6. Rotate/revoke identity material if it has been exposed.
7. Ensure application logs, support tickets, monitoring payloads, and error
   aggregation do not retain raw import requests.

### Operational Risk

Agent Identity is dependent on upstream protocol behavior. The following can
make an imported account unusable without any parser error:

- the runtime ID or private key was revoked;
- the task registration endpoint is unreachable through the configured proxy;
- account entitlement or model access was revoked;
- the upstream protocol changes its task-registration or assertion contract;
- the input represents a different authentication mode despite looking similar
  to an Agent Identity JSON object.

For this reason, run a connection test after import and monitor 401 failures,
task-registration failures, and repeated task recoveries.

## Practical Verification Checklist

After importing an Agent Identity account:

1. Confirm the account shows as OpenAI / OAuth and has the expected groups.
2. Confirm the credentials status indicates an Agent Identity private key is
   configured; do not expect the key value to be displayed.
3. Run the account connection test with a permitted model.
4. Make a controlled request through an API key assigned to the target group.
5. Confirm the account is selected by scheduler and that the request succeeds.
6. Verify that account proxy settings allow the authentication-service and
   upstream traffic required by the account.
7. Check operations logs for task registration or task-invalid recovery events.
8. Verify database-backup and export handling before relying on this account in
   a production pool.

## Relevant Source Files

- `frontend/src/components/account/CreateAccountModal.vue`
- `frontend/src/components/account/OAuthAuthorizationFlow.vue`
- `backend/internal/server/routes/admin.go`
- `backend/internal/handler/admin/account_codex_import.go`
- `backend/internal/handler/admin/account_codex_agent_identity_import_test.go`
- `backend/internal/service/openai_agent_identity.go`
- `backend/internal/service/openai_agent_identity_test.go`
- `backend/internal/service/openai_agent_identity_compat_test.go`
- `backend/internal/service/openai_gateway_forward.go`
- `backend/internal/service/openai_gateway_chat_completions.go`
- `backend/internal/service/openai_gateway_messages.go`
- `backend/internal/service/openai_codex_models_service.go`
- `backend/internal/service/openai_ws_forwarder_v2.go`
- `backend/internal/handler/dto/credentials_redact.go`
- `backend/internal/repository/account_repo.go`
- `backend/ent/schema/account.go`
