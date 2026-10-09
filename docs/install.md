# Installing

Both extensions install into PingAuthorize like any Server SDK extension: download the bundle
for your major version, check it, install it with `manage-extension`, register it with
`dsconfig`, and serve it on a connection handler.

## 1. Download the right bundle

Each [release](../../../releases) has one bundle per PingAuthorize major version:

| Your server | Bundle | Servlet API |
| ----------- | ------ | ----------- |
| PingAuthorize 10.x | `*-paz10.zip` | `javax.servlet` |
| PingAuthorize 11.x | `*-paz11.zip` | `jakarta.servlet` |

A bundle built for the other major is refused at install with "unrecognized or unsupported
extension type".

## 2. Check it

Download `SHA256SUMS` and `SHA256SUMS.asc` from the same release, and the signing key from this
repository:

```sh
gpg --import release-signing-key.asc
gpg --verify SHA256SUMS.asc SHA256SUMS          # "Good signature" from the key below
sha256sum --check --ignore-missing SHA256SUMS   # macOS: shasum -a 256 --check --ignore-missing SHA256SUMS
```

The key's fingerprint is `9746 5826 75C7 A5D0 AC41  61B8 F082 1D4B 157C A3DC`.

## 3. Install it

```sh
<server>/bin/manage-extension --install authzen-adapter-<version>-paz11.zip
```

**Building container images on PingAuthorize 11?** Install the bundle into the image at build
time, as the [quick start](../quickstart/Dockerfile) does. Delivering it through a server
profile's `server-sdk-extensions/` directory stops setup on 11.1 with "Server Already
Configured". After installing as root, `chown -R 9031:0 /opt/server`: the image's start-up runs
as 9031 and fails on a file it cannot read.

## 4. Register it

Each bundle carries a `dsconfig` batch in its `config/` directory, with every optional argument
shown as a commented example. Set the arguments for your environment, then:

```sh
<server>/bin/dsconfig --no-prompt --batch-file config/register.dsconfig
```

The [AuthZEN Adapter](authzen-adapter.md#configuration) and [SPIFFE mTLS](spiffe-mtls.md#configuration)
pages describe every argument.

The adapter will not start without a way to tell callers apart: an `api-key` of at least 32
characters, or `accepted-spiffe-id` on an mTLS handler. If a setting is unsafe or inconsistent,
setup stops with a message naming the argument, rather than starting open.

## 5. Serve it

A servlet extension answers only on the connection handlers that list it:

```sh
<server>/bin/dsconfig set-connection-handler-prop \
  --handler-name "HTTPS Connection Handler" \
  --add "http-servlet-extension:AuthZEN Adapter"
```

A handler reads its servlet list when it starts, so disable and re-enable it (or restart the
server) after changing the list. Serving the adapter on more than one handler is fine: it is
still one adapter, with one set of counters.

## Upgrading the AuthZEN Adapter from 2.0

A deployment that does not use SSF needs no configuration change. Two things it may notice: a
governance-engine response whose decision fields disagree or have the wrong type fails an
evaluation with `502` and gives a resource search no resources, and responses over `max-pdp-response-bytes` (4 MiB) are refused.

The 2.0 SSF Transmitter was a preview, and its Receivers need updating for 2.1:

| 2.0 | 2.1 | What to do |
| --- | --- | ---------- |
| HS256 with `ssf-shared-secret` | `ssf-signing-key-file` (ES256, ES384, RS256), published at `/ssf/jwks`; `ssf-shared-secret` still accepted | Move to a key file, so Receivers verify without a secret |
| Event type `https://schemas.idpartners.com.au/ssf/authzen-decision`, flat payload | `https://schemas.idpartners.com.au/secevent/authzen/event-type/decision`, with `sub_id` and `txn` on the event | Request the new event type and read the [new payload](authzen-adapter.md#decision-events-ssf) |
| A fixed list of request-context keys in `attrs` | None unless named in `ssf-event-context-attribute` | List the keys your Receivers need |
| One stream; a second create replaced the first | Up to `ssf-max-streams`; past it a create is `409` | - |
| Reading a stream returned its `authorization_header` | Never read back | - |
| `DELETE` of an unknown stream was `204` | `404` | - |
| `ssf-issuer` could be any string | An `https` URL with no query or fragment | Fix it if it was not |

Then update as below.

## Upgrading the AuthZEN Adapter from 1.x

2.0.0 changes defaults that let a 1.x deployment run open. A 1.x registration that relied on
them fails at setup with a message naming the argument.

| 1.x | 2.0.0 | What to do |
| --- | ----- | ---------- |
| No `api-key` meant any caller | Refuses to start | Set `api-key` (32+ characters) or `accepted-spiffe-id`, or `allow-unauthenticated=true` for a test bench |
| Any `api-key` length | 32 characters minimum | Generate a longer key and give it to your PEPs |
| `trust-any-server-cert` defaulted to `true` | Defaults to `false`; `true` only for a loopback `pdp-url` | Keep `true` for `https://localhost:1443`; give a remote engine a `pdp-trust-store` |
| `X-Forwarded-*` always honoured | Ignored by default | Behind a proxy that sets them, set `trust-forwarded-headers=true` |
| `/ssf/stream` behind the `api-key` | Behind `ssf-management-key` | Set it (32+ characters, not the api-key) and give it to the Receiver |
| Any SSF `endpoint_url` | `https` and public, unless listed | List an internal or `http` Receiver in `ssf-receiver-allow` |
| Any `ssf-shared-secret` | 32 bytes minimum | Generate a longer secret |
| A bad `timeout-millis` fell back to 12000 | Refuses to start | Fix the value |
| A search's `page` was ignored | Honoured | PEPs that send `page.limit` now get pages and must follow `next_token` |

Also new, with defaults that suit a 1.x deployment: `max-request-bytes`, `max-batch-size`,
`max-search-results` and `request-deadline-millis`. A decision whose `authorised` or
`authorized` is not a JSON boolean is now a `502`, not a permit.

Set any arguments the table calls for first, then:

- **A server installed on a host:** `manage-extension --update <new bundle>`. It stops the
  server, replaces the extension and starts the server again. Your registration is kept.
- **A container image:** rebuild the image with the new bundle installed, as in step 3, and
  redeploy. Don't run `manage-extension --update` inside a running container: stopping the
  server stops the container, the update is cut off, and the old version comes back.

An upgrade from 1.2.0 to 2.0.0 was tested both ways on PingAuthorize 11.1.
