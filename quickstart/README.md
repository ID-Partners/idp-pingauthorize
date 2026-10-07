# Quick start

Run the AuthZEN Adapter on PingAuthorize 11 on your own machine in about five minutes, and ask
it for decisions with `curl`.

The image is the public PingAuthorize 11.1 image with the released adapter installed. The
Dockerfile downloads the adapter bundle and checks it against its published checksum. The
server decides with a small example policy, the one from the
[AuthZEN certification scenario](https://openid.github.io/authzen/authorization-api-1_0-certification-scenario.html):
users `alice` and `bob`, and records they can read, write and delete.

## You need

- Docker with Compose
- Ping Identity DevOps credentials. The server uses them to fetch an evaluation licence at
  start-up. Sign up at <https://devops.pingidentity.com>.

## Run it

```sh
cd quickstart
cp .env.example .env
# fill in .env: your DevOps user and key, and an API key of at least 32 characters
#   (openssl rand -hex 24 makes one)
docker compose up --build
```

The server is ready when the log says `The PingAuthorize Server ... has started successfully`,
usually within two minutes. The adapter answers on <http://localhost:1080>.

## Ask it something

```sh
set -a; . ./.env; set +a
auth=(-H "Authorization: Bearer $AUTHZEN_API_KEY" -H "Content-Type: application/json")
```

**What does it serve?** The metadata document needs no key.

```sh
curl -s http://localhost:1080/.well-known/authzen-configuration
```

**Can alice read record-1?**

```sh
curl -s "${auth[@]}" http://localhost:1080/access/v1/evaluation -d '{
  "subject":  {"type": "user", "id": "alice"},
  "action":   {"name": "read"},
  "resource": {"type": "record", "id": "record-1"}
}'
# {"decision":true}
```

**Can bob write it?**

```sh
curl -s "${auth[@]}" http://localhost:1080/access/v1/evaluation -d '{
  "subject":  {"type": "user", "id": "bob"},
  "action":   {"name": "write"},
  "resource": {"type": "record", "id": "record-1"}
}'
# {"decision":false}
```

**Several questions at once.** Each entry overrides the shared subject and resource where it
gives its own.

```sh
curl -s "${auth[@]}" http://localhost:1080/access/v1/evaluations -d '{
  "subject":  {"type": "user", "id": "alice"},
  "resource": {"type": "record", "id": "record-1"},
  "evaluations": [{"action": {"name": "read"}}, {"action": {"name": "delete"}}]
}'
# {"evaluations":[{"decision":true},{"decision":false}]}
```

**Who can read record-1?**

```sh
curl -s "${auth[@]}" http://localhost:1080/access/v1/search/subject -d '{
  "subject":  {"type": "user"},
  "action":   {"name": "read"},
  "resource": {"type": "record", "id": "record-1"}
}'
# {"results":[{"type":"user","id":"alice"},{"type":"user","id":"bob"}]}
```

Without the key, every decision endpoint answers `401`.

## What's in here

| File | What it does |
| ---- | ------------ |
| `Dockerfile` | The public PingAuthorize 11.1 image, plus the released adapter, checked by SHA-256 and installed with `manage-extension`. |
| `pd.profile/dsconfig/10-embedded-pdp.dsconfig` | Has the server decide with the policy baked into the image (embedded PDP mode). |
| `pd.profile/dsconfig/20-authzen-adapter.dsconfig` | Registers the adapter and serves it, alone, on the HTTP handler at port 1080. |
| `policy/authzen-example.deploymentpackage` | The example policy. |
| `compose.yml` | Runs it, published on `127.0.0.1` only. |

## From here to production

This is set up for trying it on your own machine. Before you put it in front of anything real:

- **Use your own policy.** Export a deployment package from your Policy Editor, or point the
  server at your Policy Editor in external PDP mode.
- **Use TLS.** Serve the adapter on an HTTPS connection handler, or behind a proxy that
  terminates TLS. If that proxy sets `X-Forwarded-*` headers, set `trust-forwarded-headers=true`.
- **Keep the API key secret,** and rotate it like any other credential. Or admit callers by
  workload identity instead, with the SPIFFE mTLS extension.
- **Use a licence for your environment,** not an evaluation licence.

The [docs](../docs/) cover installing on an existing server, every configuration argument, and
monitoring.
