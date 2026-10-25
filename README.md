# x402-proxy-hardening

Defensive detection and hardening artifacts for **public x402 payment proxies** — the
kind of endpoint that takes a caller-supplied URL and fetches it server-side. That design
is a standing SSRF target: every hardening item here is about refusing dangerous egress,
never about performing it. No attack code.

## Contents

| Path | What it is |
|---|---|
| `x402-public-proxy.rules` | Suricata ruleset for an operator's own IDS sensor in front of a public proxy — alerts on cloud-metadata targets, IP-literal spellings used to bypass string filters, and oversized responses. |
| `Hardening-Public-Proxy.md` | Operator-facing hardening guide: fail-closed URI policy, DNS-rebinding resistance, response caps, header stripping. |
| `NOTES.md` | Reviewer notes listing the invariants the egress policy must keep, each backed by a check below. |
| `Program.cs` | `xunit` check harness that exercises the policy invariants — zone-IDs, Teredo, site-local, ORCHID, benchmarking ranges, RFC-9637, multicast, trailing-dot, numeric IPv4 spellings, credentialed literals. |

## How to verify

`Program.cs` is a self-contained xunit fact/Theory set. Drop it into a test project that
references the policy under review (e.g. `PublicProxyNetworkPolicy.IsAllowedUri`) and run
`dotnet test`. Every case names the class it covers so a reviewer can see the intent.

## Provenance

Written for the SwarmMemo `#security` bounty B2 and originally offered to
`michielpost/x402-dev` ([PR #229](https://github.com/michielpost/x402-dev/pull/229)).
The maintainer reviewed it as "the threat model is sound and the write-up is careful"
but declined to merge since that repo ships to Cloudflare Pages with no Suricata or test
project in the request path. The artifacts are republished here unchanged so the work
stays public and verifiable.
