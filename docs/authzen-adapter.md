# AuthZEN Adapter

Serves the [OpenID AuthZEN Authorization API 1.0](https://openid.github.io/authzen/) from inside
PingAuthorize. Each AuthZEN request becomes a governance-engine decision or query in the same
server, so there is no proxy to run between your PEPs and the PDP.

## Endpoints

| Method | Path | Purpose |
| ------ | ---- | ------- |
| `POST` | `/access/v1/evaluation` | One access evaluation |
| `POST` | `/access/v1/evaluations` | Several in one call |
| `POST` | `/access/v1/search/subject` | Which subjects may do this? |
| `POST` | `/access/v1/search/resource` | Which resources may this subject act on? |
| `POST` | `/access/v1/search/action` | What may this subject do to this resource? |
| `GET`  | `/.well-known/authzen-configuration` | AuthZEN metadata, public |
| `GET`  | `/.well-known/ssf-configuration` | SSF Transmitter metadata, public, when SSF is on |
| `POST` `GET` `DELETE` | `/ssf/stream` | SSF stream management, when SSF is on |

## Who may call it

The adapter will not start without a way to authenticate callers. Every endpoint except the two
`/.well-known/` documents checks, in this order:

1. **A listed workload.** On an mTLS handler, a caller whose SPIFFE ID is in
   `accepted-spiffe-id` is admitted without a key. See [SPIFFE mTLS](spiffe-mtls.md).
2. **The API key.** Otherwise, unless `spiffe-required=true`, the caller sends
   `Authorization: Bearer <api-key>`. The key is compared in constant time and must be at least
   32 characters.
3. **Nobody else.** A refusal is a `401` with `WWW-Authenticate: Bearer`.

`allow-unauthenticated=true` admits anyone who can reach the handler when no key and no SPIFFE
ID are set. It is for a test bench, and the server logs a severe warning when it is on.

SSF stream management has its own credential, `ssf-management-key`. Every PEP holds the API
key, and a PEP that could repoint the stream could read every decision the PDP makes.

## What it refuses before asking the policy

Whether a request is permitted is the policy's question. Whether it is an AuthZEN request at
all is the adapter's, and it answers that first, so a malformed request never reaches the
governance engine and never comes back looking like a decision.

| Rule | Answer |
| ---- | ------ |
| The `/access/v1/` endpoints take `POST` only | `405`, `Allow: POST` |
| `Content-Type` must be `application/json` | `400` |
| The body must be a JSON object | `400` |
| `subject`, `action` and `resource` must be objects; `type`, `id` and `name` strings; `properties` and `context` objects | `400` |
| An evaluation needs `subject.type`, `subject.id`, `action.name`, `resource.type` and `resource.id`; a search needs the same, less what it is looking for | `400`, naming the first one missing |
| The body is at most `max-request-bytes` | `413` |
| A batch carries at most `max-batch-size` entries | `400` |
| A search's `page.limit` is a non-negative integer, and its `page.token` one this PDP issued | `400` |

Members the adapter does not know are ignored. An `X-Request-ID` the PEP sends comes back on the
response, refusals and errors included.

## How a request reaches your policy

An evaluation becomes a governance-engine decision request. Each part of the AuthZEN tuple
travels as a JSON string attribute:

```json
{
  "domain":  "<domain-prefix>",
  "service": "<pdp-service>",
  "action":  "<pdp-action>",
  "attributes": {
    "subject":  "{\"type\":\"user\",\"id\":\"alice\"}",
    "action":   "{\"name\":\"read\"}",
    "resource": "{\"type\":\"record\",\"id\":\"record-1\"}",
    "context":  "{ ... }"
  }
}
```

Your Trust Framework reads them with request attributes named `subject`, `action`, `resource`
and `context`, and JSON paths into them. `attribute-prefix` sends them as `<prefix>.subject` and
so on instead; leave it unset unless your Trust Framework was built for a prefixed request.

**Flat attributes.** With `flat-tuple=true` the adapter also sends `agentId`, `actionName`,
`resourceType` and `resourceId`, for a policy that binds request attributes directly instead
of using JSON paths.

**Reading the decision.** The adapter accepts `{"authorised": bool}`, `{"authorized": bool}` or
`{"decision": "PERMIT"|"DENY"}` from the engine. Anything else is a `502`, never a permit. Two
policy statement codes are mapped into the response `context` for a PEP to act on:

| Statement code | Response `context` |
| -------------- | ------------------ |
| `step-up-required` | `step_up_required: true`, `step_up_scope` (payload `scope`), `reason` (payload `message`) |
| `identity-proofing-required` | `identity_proofing_required: true`, `identity_proofing_doctype` (payload `doctype`), `reason` (payload `message`) |

## Several evaluations in one call

`/access/v1/evaluations` honours `options.evaluations_semantic`: `execute_all` (the default),
`deny_on_first_deny` or `permit_on_first_permit`. A short-circuit ends the response at the entry
that tripped it. Any other value is a `400`.

Each entry overlays the request's top-level `subject`, `action`, `resource` and `context`, and an
entry's member replaces the default whole: their fields are not merged (AuthZEN 1.0 §7.1.1).
`batch-defaults=fill` relaxes that for an entry that lacks a `type`, `id` or action `name`, which
is then taken from the default. Leave it off unless a PEP depends on it.

An entry that cannot be evaluated fails on its own: the batch is still a `200`, the other
entries still run, and that entry is `{"decision": false}` with the reason under
`context.error`. With no `evaluations` array, the top-level tuple is evaluated once and
answered like a single evaluation.

## Search

A search is an evaluation with one member of the tuple left open. The adapter asks the
governance engine's query endpoint (`query-url`), which draws candidates from the **query
settings** of that attribute in your Trust Framework, puts each to the policy, and returns the
permitted ones. An attribute with no query settings is an engine error, reported as a `502`;
the fix is in the Trust Framework.

Subject and action search always work this way. Resource search does with
`resource-search=query`. Its default, `statement`, is for a policy whose candidates come from
elsewhere: one evaluation is run with the resource's `type` only, and a permit carrying a
statement with code `entitled_accounts` and payload `{"accounts": ["id-1", "id-2"]}` supplies the
answer. Anything else yields an empty result.

A result names an entity and nothing more: `type` and `id`, or an action's `name`. A search pages
when the request carries a `page` object, or when the result set is bigger than
`max-search-results`. A paged response carries `page.next_token` (empty on the last page),
`count` and `total`; repeat the request with the token for the next page.

## Decision events (SSF)

Setting `ssf-shared-secret` makes the adapter an
[OpenID Shared Signals Framework 1.0](https://openid.net/specs/openid-sharedsignals-framework-1_0.html)
Transmitter. It publishes one Security Event Token per decision to a Receiver you register.
Because every PEP asks through the adapter, this is one place that sees every decision.

```sh
curl -s https://<server>/ssf/stream \
  -H "Authorization: Bearer <ssf-management-key>" \
  -H 'Content-Type: application/json' \
  -d '{
    "aud": "receiver-app",
    "events_requested": ["https://schemas.idpartners.com.au/ssf/authzen-decision"],
    "delivery": {
      "method": "urn:ietf:rfc:8935",
      "endpoint_url": "https://receiver.example/events",
      "authorization_header": "Bearer <receiver-secret>"
    }
  }'
```

`GET /ssf/stream?stream_id=<id>` reads it back, and `DELETE` removes it. Each event is a Security
Event Token (RFC 8417) signed HS256 with the shared secret, carrying the subject, action,
resource, decision and any advice. Unless `ssf-receiver-allow` lists it, a Receiver's
`endpoint_url` must be `https` and resolve only to public addresses. That check is repeated
before each delivery.

Know its limits before you rely on it:
- It keeps one stream, in memory. The Receiver re-registers after a restart.
- Delivery is push (RFC 8935) only, without retries.
- Delivery runs on its own thread, so a slow Receiver never delays a decision. Events past a
  queue of 256 are dropped and counted.
- Signing is symmetric: a Receiver that can verify events can also forge them.

## Configuration

| Argument | Default | What it does |
| -------- | ------- | ------------ |
| `pdp-url` | `https://localhost:8443/governance-engine` | The governance engine's decision endpoint |
| `query-url` | `https://localhost:8443/governance-engine/query` | Its query endpoint, for search |
| `pdp-secret-header`, `pdp-secret` | `CLIENT-TOKEN`, unset | The shared secret the engine expects, if it expects one |
| `domain-prefix`, `pdp-service`, `pdp-action` | empty | The decision request's `domain`, `service` and `action` |
| `attribute-prefix` | empty | Prefix for the tuple's attribute names |
| `api-key` | unset | The bearer key callers send. At least 32 characters |
| `accepted-spiffe-id` | unset | Repeatable. SPIFFE IDs admitted on an mTLS handler without a key. The caller's ID reaches the engine as `caller` |
| `spiffe-required` | `false` | `true` admits listed SPIFFE IDs only, with no key fallback |
| `allow-unauthenticated` | `false` | `true` admits anyone when no key or SPIFFE ID is set. Test benches only |
| `trust-any-server-cert` | `false` | Trust the engine's certificate unchecked. Accepted only for a loopback `pdp-url` and `query-url`, for the server's own self-signed certificate |
| `pdp-trust-store` | unset | PEM file of the CAs a remote engine's certificate must chain to. Hostnames are still checked |
| `timeout-millis` | `12000` | Timeout for one engine call |
| `request-deadline-millis` | `30000` | Budget for one batch. Past it the batch fails with `504` |
| `max-request-bytes` | `1048576` | Largest body accepted |
| `max-batch-size` | `100` | Most entries in one batch |
| `max-search-results` | `1000` | Most results in one search response; more pages |
| `trust-forwarded-headers` | `false` | Build the metadata URLs from `X-Forwarded-*`. Only behind a proxy that sets them itself |
| `flat-tuple` | `false` | Also send flat attributes (see above) |
| `resource-search` | `statement` | `statement` or `query` (see Search) |
| `batch-defaults` | `replace` | `replace` or `fill` (see above) |
| `ssf-shared-secret` | unset | Turns SSF on. At least 32 bytes |
| `ssf-management-key` | unset | Bearer key for `/ssf/stream`. Required with SSF; 32+ characters, not the api-key |
| `ssf-issuer` | `https://localhost:1443` | The `iss` of the events |
| `ssf-receiver-allow` | unset | Repeatable. Receiver URLs, or prefixes, allowed besides public `https` ones |

Every numeric argument must be a positive integer. A bad value stops setup with a message
naming it.

## Operating it

**Monitor entry.** The adapter publishes its counters as one entry under `cn=monitor`, however
many handlers serve it. The server names the entry, so find it by the extension's name:

```sh
ldapsearch --baseDN cn=monitor "(ds-extension-monitor-name=AuthZEN Adapter)"
```

| Attribute | Counts |
| --------- | ------ |
| `requests` | every request served |
| `decisions-permit`, `decisions-deny` | engine decisions, one per evaluation |
| `batch-entry-errors` | batch entries that failed on their own, without reaching the engine |
| `responses-400-bad-request`, `-401-unauthorized`, `-413-too-large` | refusals |
| `responses-502-pdp-error`, `-504-pdp-timeout`, `-500-server-error` | failures |
| `pdp-calls`, `pdp-average-millis`, `pdp-max-millis` | engine calls and their latency |
| `ssf-enabled`, `ssf-events-dropped`, `ssf-deliveries-failed` | the SSF Transmitter |

A rising `responses-401-unauthorized` is someone without the key; a rising `pdp-max-millis` is
the engine slowing before it times out.

**Trace log.** Each request writes one line to the server's trace log with `endpoint`,
`method`, `status`, `caller` (`api-key`, `spiffe:<id>`, `unauthenticated` or `none`), `permits`,
`denies`, `elapsed_ms` and the PEP's `request_id`. It never includes the request body, the
subject or a credential. The decision itself, with its attributes, is in PingAuthorize's
decision log.

**Error log.** Start-up warnings for `allow-unauthenticated` and `trust-any-server-cert`, refused
SPIFFE peers, engine errors behind a `502` or `504`, unexpected failures, and SSF delivery
failures (at most one report a minute).

## Conformance

AuthZEN Adapter 2.0.0 on PingAuthorize 11.1 passes all 147 tests in the OpenID Foundation's
conformance suite plans for an AuthZEN PDP: single and batch evaluation, and subject, resource
and action search. The suite's AuthZEN plans are in alpha and are not yet part of OpenID
certification, so this is a test result, not a certification. One batch test checks a rule from
a draft of the certification scenario that the published version reversed; that run used
`batch-defaults=fill` for it.
