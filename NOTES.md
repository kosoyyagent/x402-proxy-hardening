# Public-proxy egress policy — reviewer notes (internal)

Not user-facing documentation. The proxy actions carry
`[ApiExplorerSettings(IgnoreApi = true)]` on purpose: the routes and their
request-body shape are an internal surface, and nothing here should move them
into public docs. This note is for reviewers and for whoever touches
`PublicProxyNetworkPolicy`, `ProxyController`, `ProxyResponseHeaderPolicy` or
`BoundedContentReader` next.

## Invariants the egress policy must keep

Each of these is exercised by `Program.cs` in this folder. When you change the
policy, keep every line true or explain the trade in the PR:

- **Fail closed.** `IsAllowedUri` whitelists scheme (http/https), rejects
  userinfo, restricts ports to 80/443/8080/8443, then sends the host through
  `IsAllowedHost`.
- **Host spellings all funnel to one classifier.** Reserved names
  (`localhost`, `.localhost`, `.local`, `.internal`, `.invalid`, `.test`) are
  refused after trimming brackets and a trailing dot, and any host containing
  `%` is refused — that is what kills zone-ID tricks like `fe80::1%eth0`.
- **IP literals are classified, not string-matched.** `IPAddress.TryParse`
  feeds `IsPublicAddress`, so decimal/octal/hex IPv4 spellings the URI parser
  canonicalizes (`2130706433`, `0x7f000001`, `0177.0.0.1`) and IPv4-mapped
  IPv6 (`::ffff:x.x.x.x`) land on the same rules as the plain dotted form.
  The private/reserved set includes: loopback, `0.0.0.0/8`, RFC-1918,
  CGNAT `100.64.0.0/10`, link-local `169.254.0.0/16` (cloud metadata),
  `192.0.0.0/24`, `192.0.2.0/24`, `192.88.99.0/24`, benchmark `198.18.0.0/15`,
  `198.51.100.0/24`, `203.0.113.0/24`, multicast+reserved, NAT64
  `64:ff9b::/96`, 6to4 `2002::/16`, Teredo `2001::/32`, benchmarking
  `2001:2::/48`, ORCHID `2001:10::/28`, docs `2001:db8::/32` and
  `3fff::/20` (RFC 9637), site-local `fec0::/10`, ULA `fc00::/7`, multicast,
  and anything outside global unicast `2000::/3`.
- **One DNS answer taints the set.** `AreAllPublicAddresses` refuses a
  hostname if any returned A/AAAA is non-public — no "try the good one" path.
- **The socket uses the inspected addresses.** `ConnectCallback` resolves
  once, validates the set, and dials those IPs. There is no second resolver
  pass, so pre-flight-check-then-connect TOCTOU rebinding does not exist.
- **No ambient network state.** `UseProxy = false` (an environment proxy
  would forward private URLs from a public proxy), `UseCookies = false`,
  `AllowAutoRedirect = false` — a 3xx goes back to the caller unfetched.
- **Bounded read.** `BoundedContentReader` enforces the ceiling while
  streaming, not just on `Content-Length`.
- **Response scrubbing.** `ProxyResponseHeaderPolicy` drops hop-by-hop,
  cookie and CORS headers so a target cannot poison the x402dev.com origin.

## What an outside reader cannot tell from the code

- **Rate limiting.** `proxy-3sec` is the only throttle; the checks do not
  cover burst behavior. If the route ever becomes public-facing, per-caller
  limits deserve a second look — an allowlisted open fetcher is also an
  amplification/scanning tool.
- **DNS rebinding at TTL 0.** The policy validates the resolved set at
  connect time. A hostname that is public at connect and re-points afterward
  only matters if the connection is reused past re-resolution;
  `PooledConnectionLifetime` bounds that window to two minutes.
- **Operator-side monitoring.** If you ever do add IDS signatures for this
  traffic, the classes above are the alert set, every one already refused —
  alert on them as "probed and refused", not as incidents. Threshold and
  suppress them (e.g., a per-source cap): a single scan otherwise produces a
  page per probe. Signatures must anchor to the request-target field, not to
  any private-range string anywhere in the body.

## Why this is a note and not docs

A public page naming the routes would invite the traffic the design
currently avoids. The policy stands on code and on these checks; keep it
that way.
