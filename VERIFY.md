# Verifying the Rakomi agent-action accountability anchor

Every day, Rakomi publishes a small JSON **manifest** plus its detached signature (and, when available,
an OpenTimestamps proof) under `anchors/{YYYY}/{date}.json`, `.sig`, `.ots` in this public repository.
Each manifest commits to a single 32-byte **Merkle root** over the *heads* of the per-agent
tamper-evident hash chains as they stood that day. This document is the step-by-step recipe to verify
that anchor **without trusting Rakomi** — every check uses bytes published here or timestamps from third
parties who are not Rakomi.

## What the manifest contains

```json
{
  "manifest_version": 1,
  "batch_date": "YYYY-MM-DD",
  "chain_head_root": "<64-hex sha-256>",
  "prev_chain_head_root": "<64-hex sha-256 | null>",
  "chain_count": "<integer-as-string>",
  "row_count": "<integer-as-string>",
  "leaf_schema_version": 1,
  "hash_alg": "sha-256"
}
```

Counts are strings on purpose (so a verifier in any language reproduces the exact bytes without
floating-point rounding).

## 1. Verify the detached signature

The `.sig` file is an RFC 7797 **detached** JWS (`b64:false`) over a domain-separated byte string — NOT
over the raw JSON. Reconstruct the signed bytes from the manifest fields, in this exact order:

```
agent-chain:v1:{batch_date}:{chain_head_root}:{prev_chain_head_root or empty}:{chain_count}:{row_count}
```

Then verify the detached JWS against the published public key, selected by the `kid` in the signature's
protected header. The public keys are published as a JWKS in **this same public repository** — fetch
`/.well-known/jwks.json` (raw: `https://codeberg.org/rakomidev/audit-anchor/raw/branch/main/.well-known/jwks.json`),
pick the key whose `kid` matches, and verify. Keys are retained long-term (old keys are never removed on
rotation), so a years-old anchor still verifies. Publishing the JWKS in the same independent repository
as the anchors means you never have to query a Rakomi-controlled endpoint to verify.

## 2. Recompute the root

The root is an RFC 6962 Merkle tree (leaf prefix `0x00`, internal-node prefix `0x01`, the odd node
promoted unchanged). Each leaf is `sha-256(0x00 || "{tenant}:{agent}:{head_version}:{head_hash}")`. The
published manifest is **root-only** — it deliberately does not list the per-chain head set (that would
leak tenant identifiers onto a permanent public surface). The full re-derivation from the head set is
available to a tenant for their own chains via the per-chain inclusion-proof export.

## 3. Verify the time witnesses (the part that does not depend on Rakomi)

- **OpenTimestamps** — verify `.ots` against any Bitcoin full node / the public OTS verifier. It proves
  the root existed no later than a given Bitcoin block — a timestamp from the Bitcoin network, not Rakomi.
- **Codeberg** — the commit that added these files carries Codeberg's own server-side commit time
  (Codeberg e.V. is a German non-profit). The commit history is independent of Rakomi's database.
- **Software Heritage** — the repository is archived by Software Heritage (France). The archive snapshot
  is an independent EU public-archive witness that these bytes existed.

## 4. Walk the continuity chain

`prev_chain_head_root` links each day to the previous day's root. Follow the chain backwards: a missing
day or a `prev` that does not match the prior published root is a gap you can flag.

## 5. Cross-witness reconciliation

For one `batch_date`, confirm the SAME `chain_head_root` appears across the manifest, the OpenTimestamps
proof, and (when present) any additional witness. A divergence between witnesses for the same date is
exactly the split-view condition this anchoring makes externally detectable.

---

This anchoring is **supplementary, independently-verifiable evidence** that the records existed at a
given time and have not been altered since. It is freely verifiable by anyone; it is not a paid
trust-service timestamp carrying a statutory legal presumption.
