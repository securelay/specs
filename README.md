# Securelay

A secure and ephemeral http-relay for small data.

# Version
0.1

# Features

**RESTful Web API:** Need we say more?

**SDK:** Atleast a JavaScript SDK. Here's [one](https://github.com/securelay/api) (in development).

**Ephemeral:** POST(s) and GET(s) can be concurrent. If not, POSTed data persists for a preset maximum period of time, say 24 hrs. Hence, Securelay is *ephemeral*, if not storageless.

**Duality:**

Securelay works in the following ways:
1. [Aggregator](## 'Public POST | Private GET') mode (**many-to-one**):
   Many can POST (or publish) to a public path for only one to GET (or subscribe) at a private path. POSTed data is added to a private queue and persists until next GET or expiry, whichever is earlier. This may be useful for aggregating HTML form data from one's users. Aggregated data is retrieved (GET) as a JSON array.
2. [Key-Value Store](## 'Private POST | Public GET') mode (one-to-many and one-to-one):
   - **one-to-many**: Only one can POST (or pub) to a private path for many to GET (or sub) at a public path. POSTed data persists till expiry. Expiry may be refreshed with a PATCH request at the private path, with no body. See the [Security](#security) section below for a significant usecase of this mode. Also see **CDN and password-protection** below for more details.
   - **one-to-one:** If path is suffixed with any unique string: `<channel>`. POSTs to `https://securelay.tld/<private_path>/<channel>` is stored until next GET or bodyless POST at `https://securelay.tld/<public_path>/<channel>`, or till expiry. So, only a single member of the public can consume the data. Public POSTs to a channel (i.e. at `https://securelay.tld/<public_path>/<channel>`), stores its content in the private queue, with `channel=<channel>` as metadata. If the public POST has an empty body (`content-length: 0`), securelay consumes the value stored at the channel as content. The bodyless public POST fails, if no privately POSTed content is found at the channel.

**Metadata**: Securelay adds metadata to POSTed data. Metadata mandatorily includes a unique id (`id`), Unix-time in seconds of when the data was posted (`time`) and a Boolean recording whether webhook was used (`webhook`). Implementations may also include `geolocation` and `ip` of the POST request. For public POSTs to a channel, say `<channel>`, metatdata also includes `channel=<channel>.`

**CDN and password-protection**: The one-to-many key-value store mode described above operates in two ways:

1. CDN: Data posted at a private path that is truly meant to be public is served over a [CDN](https://developer.mozilla.org/en-US/docs/Glossary/CDN) download-link. Due to cache-refresh time of CDN, any changes in the data (update/deletion) should take some time to reflect in the download link. In this mode, any GET at the publc path simply redirects to the CDN link. Data served over CDN persists longer, say for a month.

2. Password: Data posted at a private path may be protected with a password, simply by using a query string `?password=<secret>`. This data can be retrieved only with a GET at the public path, provided the same query string (i.e. password) is used during download. This type of data is ephemeral (e.g. 1-day retention). However, every download might automatically extend the expiry by 1 more day to retain oft requested data.

**Webhooks:** Private GET requests can optionally send a webhook URL using [query parameter `hook`](## '`?hook=<percent-encoded-URL>`'). The webhook URL is cached by the Securelay server for a preset [TTL](## 'Time To Live'). Subsequent public POSTs will be delivered (POST) to the cached webhook. The webhook URL will be [decached](## 'deleted from cache') if:
1. Attempted delivery to the webhook during a public POST fails.
2. A private GET does not resend the webhook URL with query `hook`.

Note that since the webhook URL can be sent only with the private path, it is never exposed to the public. So, you can safely pass a webhook URL containing, say, secret credentials.

Public POSTs respond with whether a webhook was used or not as `webhook: <boolean>`. They do not, however, reveal the webhook URL. 

**Custom redirects:** All allowed POST requests support optional query strings of the form: `?ok=<URL1>&err=<URL2>`. If the request is successful, a [`303`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/303) redirect to `URL1` is sent, instead of the usual status code [`200`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/200). On any failure, on the other hand, a `303` redirect to `URL2` is issued. Among other benefits, this helps provide user-friendly response when the user POSTs using HTML form submissions. Note: `<URL>` above denotes the [percent-encoded](https://www.urlencoder.org/) `URL`.

**CORS:** Allowing CORS is a must. Otherwise, browsers would block client side calls to the API. So securelay server replies with the HEADER- `Access-Control-Allow-Origin: *`

**Futureproof:** The URL(s) of the API endpoint(s) may be found with a GET at https://securelay.github.io/api/endpoints.json or https://cdn.jsdelivr.net/gh/securelay/api/endpoints.json courtesy of [jsdelivr](https://www.jsdelivr.com/?docs=gh). So it acts as a sort of dynamic DNS. This requires storing the list of URLs in minified JSON format in an `endpoints.json` file in the https://github.com/securelay/api repository. The format is `{"<id>":["<url1>","<url2>", ...], ...}` where `<id>` is a hash of the server secret. Having the same secret essentially means that one endpoint recognizes keys generated by the other. This also means that the endpoints, say `url1` and `url2`, share the same database; i.e. a post at `url1` may be retrieved at `url2`. URLs with the same `<id>` are interchangeable, and may be used for load balancing. Also, if an endpoint changes only its domain name, its `<id>` should persist! This helps prevent domain lock-in thus keeping domain costs lower for the Securelay service provider. Every endpoint reports its `<id>` to a GET request at path `/id`. Also endpoints must expose the version of this spec and that of their implementations, in the `/properties` path.

**Piping**: POST or PUT (GET) at public paths are piped to GET (POST or PUT) at corresponding private paths provided the paths are suffixed with the extension `.pipe`. Data here is streamed live from the sender to the receiver and not stored, even ephemerally. Essentially, the Securelay server redirects ([`307`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/307)) to a http-relay service such as [piping-server](https://github.com/nwtgck/piping-server) or [httprelay.io](https://httprelay.io/), with a unique path. The redirect URLs are valid for a preset TTL of, say, 60 seconds. Private pipes wait for the corresponding public pipes to connect. The waiting period should not exceed the abovementioned TTL of the redirect URLs. Once a private pipe expires (becomes defunct), Securelay plumbs it with a bodyless pipe. Currently, the implementations, if any, depend on the availability of 3rd party http-relays (piping-servers) for this feature to work. If the 3rd party service allows for it, there may not be any limit on the size of the piped data.

**Web-push notifications for registered apps**: A developer using Securelay as backend may register her app (`<app>`) at https://github.com/securelay/apps. Upon receiving any POST at the public path *with query string `?app=<app>`*, Securelay sends a web-push using [OneSignal](https://onesignal.com) and the registered settings. However, if a webhook is found and POSTing to that webhook succeeds, then no web-push is sent. To target the user, Securelay uses the public key as [OneSignal external_id](https://documentation.onesignal.com/docs/users#external-id). Metadata regarding the post is sent as `additionalData`. For security, the posted message is included as `additionalData.data` only if the developer has set `webPush.data` to `true` in her [`<app>.json`](https://github.com/securelay/apps) file. The frontend can simply use the [OneSignal SDK](https://documentation.onesignal.com/docs/web-sdk-reference) to register the user for the web-push. It, however, needs a OneSignal `appId` to  initiate the SDK. This is available from Securelay with a GET at path: `/properties`. For illustration, see the [Formonit](https://formonit.github.io) project that uses Securelay as backend.

# Security
Security is brought about by the use of dual paths, one private and the other public, for posting (POST) and retrieving (GET) data. Public path is derivable from private path but not the other way round. Compare this with other http-relay services like [piping-server](https://github.com/nwtgck/piping-server), [http-relay](https://httprelay.io) or [pipeto.me](https://pipeto.me) which use the same path for both GET and POST.

Note that Securelay doesn't require any API key for protecting data. The private path itself serves as one. Data can be overwritten only by the owner of the private path.

Another part of security is in ensuring that no two users can have the same private path, even by accident. Securelay achieves this by accepting only those paths that were uniquely generated and signed by the Securelay server itself!

Although Securelay can read the user data in principle, users can easily make the relay end-to-end-encrypted with the following strategy. One can simply POST his cryptographic public key at his private path for his users to consume at the corresponding public path. His users would then POST at the public path their data encrypted with that public key, for him to decrypt upon a subsequent GET at the private path!

Another security concern is whether to include the publicly POSTed message in the webpush. This decision Securelay leaves to the app developer. The message is included as `additionalData` in the OneSignal webpush only if the developer has set `webPush.data` to `true` in her [`<app>.json`](https://github.com/securelay/apps) file.

# Implementation details
A GET to `https://securelay.tld/keys` returns a new private-public key-pair as JSON: `{"private":"<private_key>","public":"<public_key>"}`.

`private_key` gives the private path as `/private/<private_key>`.

`public_key` gives the public path as `/public/<public_key>`.

<strong>A key, private or public, is generated as</strong> 
```
<typeCode> + sign(<random> + <type>) + <random>
```
where

   - `<type>` denotes the type of the key, i.e. 'private' or 'public'.

   - `<typeCode>` is a 6-bit (Base64-url) digit coding for a type. The map is: 
```
{'A': 'private', 'B': 'public'}
```

   - `+` denotes concatenation.

   - `<random>` denotes a random string (discussed below).

   - `sign(arg)` function is implemented as `substring(hmac(arg, <secret>))` where `<secret>` is some random string known only to the Securelay server. For the sake of [futureproofing](#features), `<secret>` may be related to the database used. Secret, therefore, may be chosen as some hash of the database credentials.

`<random>`, in case of private key, is sampled randomly.

`<random>`, in case of public key, is derived from that of the private key as `substring(hash(<random of private_key>))`.

MD5 is used for both `hash` and `hmac` above.

Substrings are used above to keep the key length short.

So, given any key, it is trivial to determine its type from its first letter. Also, checking its validity simply requires verifying its signature. A GET at `https://securelay.tld/keys/<key>` returns information about the `<key>` in JSON format, including any other key that may be derived from it.

Public paths are prefixed with `/public` and private paths with `/private` mainly for the sake of readability of user code.

# Limits
To mitigate abuse, the API might impose the following restrictions. Endpoint-specific limits should be available as JSON at path: `/limits`.

- Accepts and validates only these `Content-Type`s.
   - application/x-www-form-urlencoded
   - application/json
   - text/plain

- Accepts POSTs only if they have `Content-Length` less than a size-limit (`default`: 10 kiB). This may not apply for POST/PUT at path `/pipe/...`.

- Retains POSTed data until (retrieved or) expiry (`default` TTL: 24 hrs. For CDN: 30 days).

- Retains upto a certain number (`default`: 50) of Publicly POSTed messages. After reaching this limit, oldest messages are deleted, as necessary, in order to make space for the new ones.

- Upto a certain number (`default`: 50) of one-to-one fields are allowed. Beyond this limit, private POSTs creating new fields are denied until an old field is retrieved (using public GET) to make space.

- Rate-limits requests. After a certain number of 429 responses 403 bans may be imposed. [404s may also be rate limited](https://github.com/fastify/fastify-rate-limit?tab=readme-ov-file#preventing-guessing-of-urls-through-404s).

- Blocks offending IPs.

- Redirect URLs for `/pipe/` requests are valid for a certain TTL (`default`: 60 seconds).

# Use cases
- Form backend (in [aggregator](#features) mode).
- Comments store (in [aggregator](#features) mode).
- Likes/page-views/votes/clicks aggregator/counter (in [aggregator](#features) mode).
- Chats (in [one-to-one & aggregator](#features) mode).
- PubSub (in [one-to-many](#features) or key-value-store & [aggregator](#features) mode).
- Dynamic Key Value Store; values against a public key can be updated and even deleted (using the private path). For custom keys, [one-to-one](#features) mode may be used.
- Dynamic DNS (in [key-value-store](#features) mode). This is also useful for publishing dynamic relay IPs (such as provided by [ngrok](https://ngrok.com/) and other [port-forwarders](https://gist.github.com/SomajitDey/efd8f449a349bcd918c120f37e67ac00)) that expose nodes behind NAT.
- Single click URL shortener (in [one-to-one](#features) mode).
- Configuration sharing between microservices.
- WebRTC signaling server or peer-to-peer (p2p) connector.
- Request bin / http-bin (in [aggregator](#features) mode).
- Secure p2p tunneling (using the [piping feature](#features)).
- Hosting a rate-limited public webhook-server behind NAT (see [example](#api)).

# Model implementations
See repositories starting with `api-` in the [securelay](https://github.com/securelay) GitHub organization.
These implementations are not necessarily complete. [This](https://github.com/securelay/api-serverless-redis-vercel) serverless implementation is the most actively maintained. See [this](https://formonit.github.io) project for an example use case of Securelay as backend.

# Future directions
- Support email-TOTP based authorization. The URL-encoded email-id along with `?auth=<token>` query string or `Authorization: <token>` header may serve as private key. Any given email must generate one and only one public key.

- Support configuring the max number of concurrent streams. This may be done using a query parameter (`?max`) at the private path.

# API
The following documents the API by using `curl` and the original Securelay server: https://securelay.vercel.app as example. POSTs in the following examples have `Content-Type: application/x-www-form-urlencoded`.

**Note:** [Here](https://github.com/securelay/api) is a **JavaScript SDK** to access the API.

### Static assets
Each endpoint serves a few static assets as follows:
- The endpoint ID: https://securelay.vercel.app/id
- An about page: https://securelay.vercel.app/about
- A contact page: https://securelay.vercel.app/contact
- A JSON enlisting the properties or limits of the endpoint:
   - https://securelay.vercel.app/properties
   - https://securelay.vercel.app/limits

### Generate new key-pair
```bash
curl https://securelay.vercel.app/keys
```
Returns: `{"private":"A3zTryeMxkq","public":"Bw_1uSAakuZ"}`

### Check key type
```bash
curl https://securelay.vercel.app/keys/Bw_1uSAakuZ
```
Returns: `{"type":"public","public":"Bw_1uSAakuZ"}`

```bash
curl https://securelay.vercel.app/keys/A3zTryeMxkq
```
Returns: `{"type":"private","public":"Bw_1uSAakuZ"}`

### Many to One Relay
**POST at public path**:
```bash
curl -d 'data=This+is+data1' https://securelay.vercel.app/public/Bw_1uSAakuZ;
curl -d 'data=This+is+data2' https://securelay.vercel.app/public/Bw_1uSAakuZ;
```
Returns:
> `{"message":"Done","error":"Ok","statusCode":200,"webhook":false}`

Note:

The `webhook` value in the response is a Boolean stating whether the posted data was transferred via a webhook. If false, posted data remains available for GET at private path.

**GET at private path**:
```bash
curl https://securelay.vercel.app/private/A3zTryeMxkq
```
Returns:
> `[{"id":"AYqNL","time":1736920876,"data":{"data":"This is data1"}},{"id":"7q_n3","time":1736920880,"data":{"data":"This is data2"}}]`

Note:

The response is a JSON array. Each element is a JSON containing an `id` to uniquely identify the message, `time` containing the Unix time the message was posted, and `data` containing the message.

### One to Many Relay
**POST at private path**:
```bash
curl -d 'msg=This+is+a+public+notice' https://securelay.vercel.app/private/A3zTryeMxkq
```
Returns: `{"message":"Done","error":"Ok","statusCode":200, "cdn":"https://cdn.jsdelivr.net/gh/securelay/jsonbin@main/alz2h/Bw_1uSAakuZ.json"}`

However, providing a password bypasses CDN:
```bash
curl -d 'msg=This+is+a+secret+notice' https://securelay.vercel.app/private/A3zTryeMxkq?password=secret
```
Returns: `{"message":"Done","error":"Ok","statusCode":200}`

**GET at public path**:
```bash
curl https://securelay.vercel.app/public/Bw_1uSAakuZ
```
Redirects ([301](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/301)) to the CDN link: https://cdn.jsdelivr.net/gh/securelay/jsonbin@main/alz2h/Bw_1uSAakuZ.json.

However, providing a password as:
```bash
curl https://securelay.vercel.app/public/Bw_1uSAakuZ?password=secret
```
Returns: `{"id":"yW40d","time":1736921771,"data":{"msg":"This is a secret notice"}}`

Refresh expiry with PATCH at private path:
```bash
curl -X PATCH https://securelay.vercel.app/private/A3zTryeMxkq
```
Returns: `{"message":"Done","error":"Ok","statusCode":200, "cdn":"https://cdn.jsdelivr.net/gh/securelay/jsonbin@main/alz2h/Bw_1uSAakuZ.json"}` even if there is no data!

To refresh password protected data simply add the query `?password` (no need to pass the actual value of the password as the private key takes care of the authentication):
```bash
curl -X PATCH https://securelay.vercel.app/private/A3zTryeMxkq?password
```
Returns: `{"message":"Done","error":"Ok","statusCode":200}`

Note: Retrieving password protected data autorefreshes it.

DELETE at private path (unpublishes from CDN):
```bash
curl -X DELETE https://securelay.vercel.app/private/A3zTryeMxkq
```
Returns: [`204 No Content`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/204)

To delete the password protected data simply add the query `?password`.

### Stats
GET at private path with query string `?stats` gives number of public POSTs waiting to be retrieved (consumed), which have not expired yet. It also gives the remaining TTL (in seconds) for those data as well as the data last published with POST at the private path. TTL value of 0 would mean data has either been consumed or has expired.
```bash
curl https://securelay.vercel.app/private/A3zTryeMxkq?stats
```
Returns: `{"count":2,"ttl":86395}`

### One to One Relay
**POST at private path with some custom field (any random string)**:
```bash
curl -d 'msg=This+is+a+private+notice' https://securelay.vercel.app/private/A3zTryeMxkq/anyRandString

curl -d 'msg=This+is+another+private+notice' https://securelay.vercel.app/private/A3zTryeMxkq/anotherRandString
```
Returns: `{"message":"Done","error":"Ok","statusCode":200}`

**Check TTL (in seconds) of one-to-one data** (value 0 would mean data has either been consumed or has expired):
```bash
curl https://securelay.vercel.app/private/A3zTryeMxkq/anyRandString
```
Returns: `{"ttl":86397}`

**GET at public path with some custom field**:
```bash
curl https://securelay.vercel.app/public/Bw_1uSAakuZ/anyRandString
```
Returns: `{"id":"OL0UR","time":1736921895,"data":{"msg":"This is a private notice"}}`

```bash
curl https://securelay.vercel.app/public/Bw_1uSAakuZ/anotherRandString
```
Returns: `{"id":"aChYU","time":1736921895,"data":{"msg":"This is another private notice"}}`

### Get endpoint's ID
```bash
curl https://securelay.vercel.app/id
```
Returns: `alz2h`

### Webhooks
Open two terminals: A and B.

A. At terminal A, set up a webhook with URL https://ppng.io/A3zTryeMxkq as follows:
```bash
while curl -f https://ppng.io/A3zTryeMxkq; do :; done
```
B. At terminal B:
```bash
# Let Securelay know about your webhook URL
curl https://securelay.vercel.app/private/A3zTryeMxkq?hook=https%3A%2F%2Fppng.io%2FA3zTryeMxkq

# Make a public POST
curl -d 'data=This+is+data' https://securelay.vercel.app/public/Bw_1uSAakuZ
```
Terminal A should output: `{"id":"lwjHI","time":1736921992,"data":{"data":"This is data"}}`

### Custom redirects
Applicable for all allowed POST requests. Example:
```bash
curl -i -d 'data=This+is+data' 'https://securelay.vercel.app/public/Bw_1uSAakuZ?ok=https%3A%2F%2Fexample.com&err=https%3A%2F%2Fgithub.com%2F404.html'
```

### Get OneSignal appId for the endpoint
```bash
curl https://securelay.vercel.app/id?app=formonit
```
Returns: `<ONESIGNAL_APP_ID>`.

### Piping
Note: This feature is experimental and depends on the availability of 3rd party service(s).

Open two terminals A and B. In terminal A, GET from private pipe `/private/A3zTryeMxkq.pipe`, while at terminal B, POST (or PUT) to public pipe `/public/Bw_1uSAakuZ.pipe`. POSTed data is transferred as stream (i.e. piped) without being stored in the server.

Terminal A:
```bash
curl -L -i https://securelay.vercel.app/private/A3zTryeMxkq.pipe
```
Terminal B:
```bash
curl -L -i -d 'data=hello+world' https://securelay.vercel.app/public/Bw_1uSAakuZ.pipe
```

Note: `-L` option is used above to allow `curl` to follow the redirects.

Note: The private pipe must precede (i.e. wait for) the public pipe. Otherwise, public pipe would result in a 404 error. However, one can optionally redirect the public pipe to a web-page instead of the 404, by using the query string `?fail=<URLencoded-URL-of-landing-page>` while creating the private pipe.

Similarly for POST/PUT at private path and GET at public.

One can, for example, run a webhook server, even behind NAT, as:
```bash
while true; do timeout 60 curl -L https://securelay.vercel.app/private/A3zTryeMxkq.pipe; echo; done
```
with its public endpoint URL being the corresponding public path: `https://securelay.vercel.app/public/Bw_1uSAakuZ.pipe`.
