# Security

## Reporting a vulnerability

Report it privately through GitHub: the **Security** tab of this repository, then **Report a
vulnerability**. Please don't open a public issue.

Tell us which extension and version, which PingAuthorize version, and how to reproduce it. We'll
confirm we have it, keep you informed while we work on it, and credit you in the release notes
if you'd like.

## Supported versions

Security fixes go into the latest release of each extension:

| Extension | Supported |
| --------- | --------- |
| AuthZEN Adapter | 2.0.x |
| SPIFFE mTLS (preview) | the latest 0.x |

## Checking what you install

Every release's bundles are listed in `SHA256SUMS`, signed with the key in
[release-signing-key.asc](release-signing-key.asc)
(`9746 5826 75C7 A5D0 AC41  61B8 F082 1D4B 157C A3DC`). [Installing](docs/install.md#2-check-it)
shows how to check them.

From SPIFFE mTLS 0.1.1 and AuthZEN Adapter 2.0.1, each release also carries a software
bill of materials (CycloneDX JSON) for each bundle, listing the libraries shaded inside its
dependencies as well. Each is checked against the OSV vulnerability database before release. Any
published advisory against something a release ships, and why it does not affect it, is listed in
that release's `ADVISORIES.txt`.

## Libraries PingAuthorize provides

The AuthZEN Adapter bundles no libraries. It uses what the running PingAuthorize provides,
Jackson and the servlet API among them, so advisories against those are PingAuthorize's, and are
fixed by patching PingAuthorize. PingAuthorize 10.2.0.1, for example, ships jackson-databind
2.16.2, which has published advisories; we have checked each and none is reachable through the
adapter, which parses requests into plain JSON trees and its own simple types. Keep PingAuthorize
on a supported, patched release.
