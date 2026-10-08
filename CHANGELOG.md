# Changelog

Full notes, with downloads, are on each [release](../../releases).

## AuthZEN Adapter

### 2.0.1

- Pooled connections to the governance engine: steady throughput under load, where 2.0.0 fell
  away past five concurrent decisions. At low concurrency it adds under a millisecond.
- SPIFFE trust domains compare without case, as the SPIFFE ID specification says.
- A redirect from the governance engine is an error.
- A software bill of materials for each bundle.
- The SSF Transmitter is labelled a preview.

### 2.0.0

The first public release.

- Secure by default: the adapter refuses to start without an API key of at least 32
  characters, a listed SPIFFE ID, or an explicit `allow-unauthenticated=true`. The governance
  engine's certificate is checked unless it is on loopback, and `X-Forwarded-*` headers are
  ignored unless trusted.
- Limits on request size, batch size, search results and batch duration.
- Search results page, with `next_token`.
- SSF stream management has its own key, and Receivers must be public `https` addresses unless
  listed. SSF is a preview.
- One monitor entry with request, decision, refusal and latency counters, however many
  connection handlers serve the adapter; one trace line per request.
- Passes the 147 tests in the OpenID Foundation's AuthZEN PDP conformance plans (alpha, not
  certification).

Upgrading from a 1.x build: see [Installing](docs/install.md#upgrading-the-authzen-adapter-from-1x).

## SPIFFE mTLS

### 0.1.1 (preview)

- java-spiffe 0.8.17, clearing published advisories against the libraries 0.8.12 bundled.
- Trust domains compare without case, as the SPIFFE ID specification says; paths still exactly.
- A software bill of materials for each bundle. Known advisories in the Netty that java-spiffe
  shades, and why they don't apply, are in the release notes.

### 0.1.0 (preview)

The first public release: the server's TLS certificate from SPIRE, client SVIDs checked against
the trust bundle and optional allow-lists, rotation without restarts, start-up before SPIRE is
ready, and an SVID expiry gauge.
