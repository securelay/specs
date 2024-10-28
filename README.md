# Securelay

A secure and ephemeral http-relay for small data.

# Features

**Ephemeral:** POST(s) and GET(s) can be concurrent. If not, POSTed data persists for a preset maximum period of time, say 24 hrs. Hence, Securelay is *ephemeral*, if not storageless.

**Duality:**

Securelay works in the following ways:
1. [Aggregator](## 'Public POST | Private GET') mode (**many-to-one**):
   Many can POST (or publish) to a public path for only one to GET (or subscribe) at a private path. POSTed data persist until next GET or expiry, whichever is earlier. This may be useful for aggregating HTML form data from one's users. Aggregated data is retrieved (GET) as a JSON array.
2. [Key-Value Store](## 'Private POST | Public GET') mode (one-to-many and one-to-one):
   - **one-to-many**: Only one can POST (or pub) to a private path for many to GET (or sub) at a public path. POSTed data persists till expiry. Expiry may be refreshed with a PATCH request at the private path, with no body. See the [Security](#security) section below for a significant usecase of this mode.
   - **one-to-one:** If path is suffixed with a user-given unique id `<uid>`. POSTed data persists until next GET or expiry, whichever is earlier. That is to say, when one POSTs to `https://api.securelay.tld/<private_path>/<uid>`, there can be only one GET consumer at `https://api.securelay.tld/<public_path>/<uid>`, after which any more GET at that path would result in a 404 error. This is useful for sending a separate response to each POSTer.

**Webhooks:** Private GET requests can optionally send a webhook URL using [query parameter `hook`](## '`?hook=<percent-encoded-URL>`'). The webhook URL is cached by the Securelay server for a preset [TTL](## 'Time To Live'). Subsequent public POSTs will be delivered (POST) to the cached webhook. The webhook URL will be [decached](## 'deleted from cache') if:
1. Attempted delivery to the webhook during a public POST fails.
2. A private GET does not resend the webhook URL with query `hook`.

Note that since the webhook URL can be sent only with the private path, it is never exposed to the public. So, you can safely pass a webhook URL containing, say, secret credentials.

Public POSTs respond with whether a webhook was used or not as `webhook: <boolean>`. They do not, however, reveal the webhook URL. 

**Custom redirects:** All allowed POST requests support optional query strings of the form: `?ok=<URL1>&err=<URL2>`. If the request is successful, a [`303`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/303) redirect to `URL1` is sent, instead of the usual status code [`200`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200). On any failure, on the other hand, a `303` redirect to `URL2` is issued. Among other benefits, this helps provide user-friendly response when the user POSTs using HTML form submissions. Note: `<URL>` above denotes the [percent-encoded](https://www.urlencoder.org/) `URL`.

**CORS:** Allowing CORS is a must. Otherwise, browsers would block client side calls to the API. So securelay server replies with the HEADER- `Access-Control-Allow-Origin: *`

**Futureproof:** The URL(s) of the API endpoint(s) may be found with a GET at https://raw.githubusercontent.com/securelay/api/main/endpoints.json or https://cdn.jsdelivr.net/gh/securelay/api/endpoints.json courtesy of [jsdelivr](https://www.jsdelivr.com/?docs=gh). So it acts as a sort of dynamic DNS. This requires storing the list of URLs in minified JSON format in an `endpoints.json` file in the https://github.com/securelay/api repository. The format is `{"<id>":["<url1>","<url2>", ...], ...}` where `<id>` is a hash of the server secret. Having the same secret essentially means that one endpoint recognizes keys generated by the other. This also means that the endpoints, say `url1` and `url2`, share the same database; i.e. a post at `url1` may be retrieved at `url2`. URLs with the same `<id>` are interchangeable, and may be used for load balancing. Also, if an endpoint changes only its domain name, its `<id>` should persist! This helps prevent domain lock-in thus keeping domain costs lower for the Securelay service provider. Every endpoint reports its `<id>` to a GET request at path `/id`.

# Security
Security is brought about by the use of dual paths, one private and the other public, for posting (POST) and retrieving (GET) data. Public path is derivable from private path but not the other way round. Compare this with other http-relay services like [piping-server](https://github.com/nwtgck/piping-server), [http-relay](https://httprelay.io) or [pipeto.me](https://pipeto.me) which use the same path for both GET and POST.

Note that Securelay doesn't require any API key for protecting data. The private path itself serves as one. Data can be overwritten only by the owner of the private path.

Another part of security is in ensuring that no two users can have the same private path, even by accident. Securelay achieves this by accepting only those paths that were uniquely generated and signed by the Securelay server itself!

Although Securelay can read the user data in principle, users can easily make the relay end-to-end-encrypted with the following strategy. One can simply POST his cryptographic public key at his private path for his users to consume at the corresponding public path. His users would then POST at the public path their data encrypted with that public key, for him to decrypt upon a subsequent GET at the private path!

# Implementation details
A GET to `https://api.securelay.tld/keys` returns a new private-public key-pair as JSON: `{"private":"<private_key>","public":"<public_key>"}`.

`private_key` gives the private path as `/private/<private_key>`.

`public_key` gives the public path as `/public/<public_key>`.

A key, private or public, is generated as `sign(<random> + <type>) + <random>`. `<type>` denotes the type of the key, i.e. 'private' or 'public' and `+` denotes concatenation. `<random>` denotes a random string (discussed below). Note: `<random>` is a substring of the key.

The `sign(arg)` function is implemented as `substring(hmac(arg, <secret>))` where `<secret>` is some random string known only to the Securelay server. For the sake of [futureproofing](#features), `<secret>` may be related to the database used. Secret, therefore, may be chosen as some hash of the database credentials.

`<random>`, in case of private key, is `substring(hash(<UUID>))` where `<UUID>` is a [version 4 UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)).

`<random>`, in case of public key, is `substring(hash(<random of private_key>))`.

SHA256 or MD5 may be used for both `hash` and `hmac` above.

Substrings are used above only to keep the key length short.

Note that given any key, it is trivial to determine whether it is public or private just by validating its signature. A GET at `https://api.securelay.tld/keys/<key>` returns information about the `<key>` in JSON format. If the provided key is private, it's public key is also returned.

Public paths are prefixed with `/public` and private paths with `/private` mainly for the sake of readability of user code.

# Limits
To mitigate abuse, the API might impose the following restrictions.

- Accepts and validates only these `Content-Type`s.
   - application/x-www-form-urlencoded
   - application/json
   - text/plain

- Accepts POSTs only if they have `Content-Length` less than a size-limit (`default`: 10 kiB).

- Retains POSTed data until (retrieved or) expiry (`default TTL`: 24 hrs).

- Rate-limits requests. After a certain number of 429 responses 403 bans may be imposed. [404s may also be rate limited](https://github.com/fastify/fastify-rate-limit?tab=readme-ov-file#preventing-guessing-of-urls-through-404s).

- Blocks offending IPs.

# Use cases
- Forms
- Comments
- Likes/page-views/votes/clicks aggregator/counter
- Chats
- PubSub
- Dynamic Key Value Store
- Dynamic DNS
- Single click URL shortener
- Configuration sharing between microservices
- Request bin / http-bin

# Model Implementations
See repositories starting with `api-` in the [securelay](https://github.com/securelay) GitHub organization.
These implementations are not necessarily complete.

# Future directions
- Accept `?verify=<email>` query parameter when requesting the key-pair at `https://api.securelay.tld/keys`. This prompts Securelay to send an ephemeral nonce to the provided email. On presenting this nonce on a subsequent key-pair request with the query `?email=<email>&nonce=<nonce>&salt=<custom>` generates the desired key-pair. The `<random>` used to create the private key in this key-pair is : `hash(<email>+<custom>)`. So, basically it deterministically maps a *verified* email and a user-chosen salt to a private key. Even if the user loses his private key, he can easily retrieve it by verifying his email.

- Accept `Content-Type: multipart/form-data`.

# API
The following documents the API by using `curl` and the original Securelay server: https://securelay.vercel.app as example. POSTs in the following examples have `Content-Type: application/x-www-form-urlencoded`.

**Note:** [Here](https://github.com/securelay/api/blob/main/script.js) is a **JavaScript module** to access the private parts of the API.

### Generate new key-pair
```bash
curl https://securelay.vercel.app/keys
```
Returns: `{"private":"3zTryeMxkq","public":"w_1uSAakuZ"}`

### Check key type
```bash
curl https://securelay.vercel.app/keys/w_1uSAakuZ
```
Returns: `{"type":"public"}`

```bash
curl https://securelay.vercel.app/keys/3zTryeMxkq
```
Returns: `{"type":"private","public":"w_1uSAakuZ"}`

### Many to One Relay
POST at public path:
```bash
curl -d 'data=This+is+data1' https://securelay.vercel.app/public/w_1uSAakuZ;
curl -d 'data=This+is+data2' https://securelay.vercel.app/public/w_1uSAakuZ;
```
Returns: `{"message":"Done","error":"Ok","statusCode":200}`

GET at private path:
```bash
curl https://securelay.vercel.app/private/3zTryeMxkq
```
Returns: `[{"data":"This is data1"},{"data":"This is data2"}]`

### One to Many Relay
POST at private path:
```bash
curl -d 'msg=This+is+a+public+notice' https://securelay.vercel.app/private/3zTryeMxkq
```
Returns: `{"message":"Done","error":"Ok","statusCode":200}`

GET at public path:
```bash
curl https://securelay.vercel.app/public/w_1uSAakuZ
```
Returns: `{"msg":"This is a public notice"}`

Refresh expiry with PATCH at private path:
```bash
curl -X PATCH https://securelay.vercel.app/private/3zTryeMxkq
```
Returns: `{"message":"Done","error":"Ok","statusCode":200}` even if there is no data!

DELETE at private path:
```bash
curl -X DELETE https://securelay.vercel.app/private/3zTryeMxkq
```
Returns: [`204 No Content`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/204)

### One to One Relay
POST at private path with some custom field:
```bash
curl -d 'msg=This+is+a+private+notice' https://securelay.vercel.app/private/3zTryeMxkq/field
```
Returns: `{"message":"Done","error":"Ok","statusCode":200}`

Check if one-to-one data has been consumed:
```bash
curl https://securelay.vercel.app/private/3zTryeMxkq/field
```
Returns: `Not consumed yet.`

GET at public path with some custom field:
```bash
curl https://securelay.vercel.app/public/w_1uSAakuZ/field
```
Returns: `{"msg":"This is a private notice"}`

Check if one-to-one data has been consumed:
```bash
curl https://securelay.vercel.app/private/3zTryeMxkq/field
```
Returns: `Consumed.`

### Get endpoint's ID
```bash
curl https://securelay.vercel.app/id
```
Returns: `alz2h`

### Webhooks
Open two terminals: A and B.

A. At terminal A, set up a webhook with URL https://ppng.io/3zTryeMxkq as follows:
```bash
while curl -f https://ppng.io/3zTryeMxkq; do :; done
```
B. At terminal B:
```bash
# Let Securelay know about your webhook URL
curl https://securelay.vercel.app/private/3zTryeMxkq?hook=https%3A%2F%2Fppng.io%2F3zTryeMxkq

# Make a public POST
curl -d 'data=This+is+data' https://securelay.vercel.app/public/w_1uSAakuZ
```
Terminal A should output: `{"data":"This is data"}`

### Custom redirects
Applicable for all allowed POST requests. Example:
```bash
curl -i -d 'data=This+is+data' 'https://securelay.vercel.app/public/w_1uSAakuZ?ok=https%3A%2F%2Fexample.com&err=https%3A%2F%2Fgithub.com%2F404.html'
```
