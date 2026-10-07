# PingAuthorize extensions from ID Partners

Packaged extensions for **PingAuthorize**, ready to install into your own server.

Download them from [Releases](../../releases). Each release's notes cover installing and
upgrading.

## What's here

| Extension | What it does | Status |
| --------- | ------------ | ------ |
| **AuthZEN Adapter** | Serves the OpenID AuthZEN Authorization API 1.0 from inside PingAuthorize and turns each request into a policy decision. | Generally available |
| **SPIFFE mTLS** | Gives PingAuthorize a SPIFFE workload identity and admits only callers with SPIRE-issued certificates, so certificates rotate without restarts. | Preview |

Each release includes installable extension bundles for PingAuthorize 10.x and 11.x, and a
signed list of their checksums. Installation and hardening guides, and a quick start that runs
the adapter on the public PingAuthorize image (bring your own licence), are on the way.

## Verifying a download

Each release carries `SHA256SUMS` and its signature, `SHA256SUMS.asc`. The signing key is
[release-signing-key.asc](release-signing-key.asc), fingerprint
`9746 5826 75C7 A5D0 AC41  61B8 F082 1D4B 157C A3DC`.

```sh
gpg --import release-signing-key.asc
gpg --verify SHA256SUMS.asc SHA256SUMS          # expect "Good signature" from the key above
sha256sum --check --ignore-missing SHA256SUMS   # macOS: shasum -a 256 --check --ignore-missing SHA256SUMS
```

## Learn more

- Product overview: <https://ping-authorize.idpartners.global>
- Talk to us: <https://idpartners.com.au/contact>

## Licence

MIT. See [LICENSE](LICENSE).
