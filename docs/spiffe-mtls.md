# SPIFFE mTLS (preview)

Gives PingAuthorize a [SPIFFE](https://spiffe.io) workload identity from SPIRE. The server
presents its own X.509-SVID on an HTTPS connection handler and admits only clients whose SVID
chains to the SPIRE trust bundle, optionally pinned to a list of SPIFFE IDs. SVIDs rotate in
place, with no restart. With the AuthZEN Adapter on the same handler, a PEP is identified by its
workload identity instead of a shared key.

This is a **preview** (0.x). Its configuration may change before 1.0.

## How it fits together

On PingAuthorize, mTLS is not something a servlet does. The connection handler terminates TLS
with the key manager and trust manager it is configured with. So this module is three
extensions and one handler:

| Piece | Type | What it does |
| ----- | ---- | ------------ |
| SPIFFE Key Manager Provider | Key Manager Provider | Serves the server's X.509-SVID from the Workload API, read on each handshake, so rotation needs no restart |
| SPIFFE Trust Manager Provider | Trust Manager Provider | Checks client chains against the trust bundle the Workload API keeps current, and the allow-lists |
| SPIFFE Who Am I | HTTP servlet | `GET /spiffe/whoami` answers with the caller's SPIFFE ID, to prove the path end to end |

The registration batch in the bundle creates all three and a dedicated handler on port 8444
with `ssl-client-auth-policy:required`. Your existing HTTPS handler and the server's
administrative APIs are not touched.

## What it needs

- **A SPIRE agent's Workload API socket, mounted into the PingAuthorize container.** On
  Kubernetes, the SPIFFE CSI driver does this:

  ```yaml
  volumeMounts:
    - name: spiffe-workload-api
      mountPath: /spiffe-workload-api
      readOnly: true
  volumes:
    - name: spiffe-workload-api
      csi:
        driver: csi.spiffe.io
        readOnly: true
  ```

- **A SPIRE registration entry for PingAuthorize,** and one for each PEP that will call it.
  The server process runs as UID 9031 in the official image, which a `unix:uid:9031` selector
  matches.
- **Linux on x86_64 or aarch64.**

## Configuration

| Argument | On | Default | What it does |
| -------- | -- | ------- | ------------ |
| `spiffe-socket` | both providers | `unix:///spiffe-workload-api/spire-agent.sock` | The Workload API endpoint |
| `workload-api-init-timeout-seconds` | both providers | `15` | How long start-up waits for the first SVID, 1 to 300. Past it the provider starts anyway and keeps trying |
| `accepted-trust-domain` | trust manager | unset | Repeatable. Only these trust domains get in |
| `accepted-spiffe-id` | trust manager | unset | Repeatable. Only these workloads get in |

With neither allow-list set, any workload whose SVID chains to the bundle is admitted. With
SPIRE federation, that means every federated trust domain, so set `accepted-trust-domain` to
your own. When both are set, a client must pass both. Changes to either list apply on the next
handshake.

A workload that isn't admitted fails the TLS handshake. It never reaches a servlet, so it never
sees a `401`.

**Starting before SPIRE is ready.** If no SVID arrives within the timeout, the provider logs a
warning, starts anyway and retries in the background: after a second, then doubling, at most a
minute apart. Handshakes fail closed until the first SVID arrives, and then the handler works,
with no restart.

## With the AuthZEN Adapter

Serve the adapter on the mTLS handler and list the PEPs it should admit:

```sh
dsconfig set-http-servlet-extension-prop --extension-name "AuthZEN Adapter" \
  --add "extension-argument:accepted-spiffe-id=spiffe://example.org/ns/peps/sa/payments-pep"
dsconfig set-connection-handler-prop --handler-name "SPIFFE mTLS Connection Handler" \
  --add "http-servlet-extension:AuthZEN Adapter"
```

A listed PEP needs no API key, and its SPIFFE ID reaches your policy as the `caller` attribute.
The identity comes from the verified certificate, never from the request, so a client cannot
claim to be a workload it is not. Add `spiffe-required=true` to refuse the API key on this
adapter altogether.

## Check it

From a workload with its own SVID:

```sh
spire-agent api fetch x509 -socketPath /spiffe-workload-api/spire-agent.sock -write /tmp/svid
curl --cert /tmp/svid/svid.0.pem --key /tmp/svid/svid.0.key --cacert /tmp/svid/bundle.0.pem \
     https://pingauthorize:8444/spiffe/whoami
# {"spiffe_id":"spiffe://example.org/ns/peps/sa/payments-pep","trust_domain":"example.org","path":"/ns/peps/sa/payments-pep"}
```

An SVID usually carries only a URI SAN, so curl's hostname check fails. Either register
PingAuthorize's SPIRE entry with a DNS name (`-dns pingauthorize`), or use a SPIFFE-aware
client, which checks the server's SPIFFE ID instead.

## Monitoring

PingAuthorize monitors the served SVID like any certificate it serves, under `cn=monitor`, with
`not-valid-after`, `expires-seconds` and `currently-valid`:

```sh
ldapsearch --baseDN cn=monitor "(&(objectClass=ds-x509-certificate-monitor-entry)(component-name=SPIFFE Key Manager Provider))"
```

The stock "Certificate Expiration (Days)" gauge is meant for certificates that live for months.
It would rate an hour-long SVID major from the moment it is issued, and raise the alarm again at
every rotation. The registration batch keeps SVIDs out of that gauge and adds "SPIFFE SVID
Expiration (Seconds)", which raises an alarm only when rotation stalls:

| Seconds left | Severity |
| ------------ | -------- |
| under 900 | warning |
| under 600 | minor |
| under 300 | major |
| under 60 | critical |

The thresholds suit SPIRE's default one-hour SVID, which the agent replaces at half-life. With a
shorter SVID lifetime, lower them with `dsconfig set-gauge-prop --gauge-name "SPIFFE SVID
Expiration (Seconds)"`.

The server's error log records each rotation, an expired SVID that hasn't been replaced, refused
workloads (at most once a minute, with a running total), and problems with the Workload API
connection.

## How it was tested

Every change is tested against a real SPIRE server and agent: the providers start before SPIRE
has an entry for them, the SVID rotates under live traffic, and the agent is stopped and
restarted. It is also tested inside a licensed PingAuthorize 11.1, alongside the AuthZEN Adapter.
That run checks the server presents its own SVID, admits only the listed workload, and refuses
another workload's SVID, and a client with no certificate, during the TLS handshake.
