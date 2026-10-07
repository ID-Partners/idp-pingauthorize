# PingAuthorize extensions from ID Partners

Packaged extensions for **PingAuthorize**, ready to install into your own server.

> **First release coming soon.** No packages have been published here yet. Watch this
> repository's releases to be notified.

## What will be here

| Extension | What it does | Status |
| --------- | ------------ | ------ |
| **AuthZEN Adapter** | Serves the OpenID AuthZEN Authorization API 1.0 from inside PingAuthorize and turns each request into a policy decision. | Generally available from the first release |
| **SPIFFE mTLS** | Gives PingAuthorize a SPIFFE workload identity and admits only callers with SPIRE-issued certificates, so certificates rotate without restarts. | Preview |

Each release will include:

- installable extension bundles for PingAuthorize 10.x and 11.x, with checksums
- configuration examples
- installation and hardening guides
- a quick start that runs the adapter on the public PingAuthorize image (bring your own licence)

## Learn more

- Product overview: <https://ping-authorize.idpartners.global>
- Talk to us: <https://idpartners.com.au/contact>

## Licence

MIT. See [LICENSE](LICENSE).
