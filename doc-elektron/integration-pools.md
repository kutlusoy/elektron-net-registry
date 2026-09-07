# Elektron Net Registry - Integration Guide for Pool Software

- **Version:** 1.0
- **Date:** September 6, 2026
- **Audience:** developers of any Elektron Net pool/mining-pool software (solo or shared) wanting to participate in cross-explorer block attribution, not only `elektron-net-pool`/`elektron-net-ppool`
- **Reference implementation:** [`elektron-net-pool`](https://github.com/kutlusoy/elektron-net-pool) - `src/services/pool-registry.service.ts`, [`elektron-net-ppool`](https://github.com/kutlusoy/elektron-net-ppool) - `src/services/pool-registry.service.ts` (identical implementations)
- **See also:** [`integration-mempools.md`](./integration-mempools.md) (the other side of this protocol), each reference repo's own `doc-elektron/guideline-pool-registry-reporting.md`, [`../README.md`](../README.md) (the registry file format)

- Never use the em dash character in this document or its follow-up code comments; use a hyphen and spaces instead, as done throughout.

---

## 1. What This Solves

Block explorers cannot reliably show which pool found a block unless the pool is in a curated address registry, or unless the pool proves its identity somehow. Elektron Net's coinbase transaction cannot carry extra data for this (any additional output invalidates the network's per-block UTXO attestation - see the reference repos' own `fix-report-pool-identity-utxo-attestation.md`), so this has to happen off-chain.

This protocol lets a pool report a block it found directly to every known block explorer instance, and lets each explorer verify the claim itself before trusting it - no wallet, no signature, no token, no shared secret. Trust is established purely by whether your software's own web server, reached at the URL you registered, confirms the claim. This works identically whether your pool has a wallet (PPLNS-style) or not (solo, coinbase pays the finding miner directly).

## 2. Step 1: Read `mempools.txt`

Fetch `<REGISTRY_URL>/mempools.txt` (default `REGISTRY_URL` is `https://raw.githubusercontent.com/kutlusoy/elektron-net-registry/main`; make this configurable, never hardcode it as the only option). One entry per line:

```
"Name", "URL";
```

Parsing note: only the quoted substrings on each line matter. A conforming parser extracts every `"..."` token on a line and ignores whatever separates them, so it does not matter whether the file (or a fork of it) uses commas, semicolons, or nothing between fields. Skip any line that does not yield exactly two non-empty quoted fields rather than failing the whole file.

- **Poll interval:** every 15 minutes is what the reference implementation uses. The file is tiny (a handful of KB even with hundreds of entries), so there is no need for conditional-fetch/diffing logic - just refetch the whole file each time.
- **Local cache:** write the last successfully fetched file to local disk, and read that local copy first on startup, before attempting any network request. This way your pool has a usable list immediately even if the registry host is unreachable at boot, and keeps working through any later outage using whatever it last saw. Never let a failed fetch, or one that parses to zero entries (e.g. a redirect to an HTML error page), overwrite an already-good in-memory list or the on-disk cache - only replace either once a fetch actually parses successfully into at least one entry.

## 3. Step 2: Report a Found Block

The moment your software knows it found a block (e.g. right after `submitblock` succeeds), send a report to every mempool instance from your synced `mempools.txt`:

```
POST <mempool_url>/api/v1/pool-registry/report
Content-Type: application/json

{
  "name": "<your pool's exact Name field, as registered in pools.txt>",
  "blockHash": "<64-character lowercase hex block hash>"
}
```

- Send to every known instance in parallel; this is fire-and-forget. A mempool that does not answer, times out, or answers negatively simply never attributes that block to you - nothing else depends on the outcome, no retry is needed.
- Recommended timeout: 5 seconds per request.
- Expected responses (informational only, do not branch your own logic on the exact body): `200` with `{"attributed": true|false}` on a processed request, `400` on a malformed payload, `404` if the receiving mempool does not recognize your pool name from its own synced `pools.txt` yet (can happen briefly right after you first register, before that instance's next poll), `503` if the instance has its own database/indexing disabled.

## 4. Step 3: Implement the Confirmation Callback

**This is the security-critical part.** Every mempool instance that receives a report from you will call back to your own registered URL to verify it before trusting anything:

```
GET <your_pool_url>/api/pool/identity/confirm?blockHash=<64-character hex>

200 OK
{
  "confirmed": true | false,
  "name": "<your pool's name, or null>",
  "url": "<your pool's URL, or null>"
}
```

`confirmed` **must** be `true` only for a block hash your software genuinely, recently found and submitted itself - never for a hash it merely recognizes as valid, never as a default `true`, never based on trusting the caller. The entire trust model rests on this endpoint being answered honestly: nobody else can make your server say `true` for a block they claim, since they do not control your infrastructure, so a correct implementation is unforgeable by a third party.

- The `/api` path segment is required, not optional: mempool instances call back to exactly `<your_pool_url>/api/pool/identity/confirm`, since that is what the reference implementations serve at their registered dashboard domain. Serve this endpoint at that path, whatever your own internal routing otherwise looks like.
- Keep a short-lived, in-memory record of block hashes you have recently found (a TTL map or small ring buffer is enough; no database table needed). The reference implementation uses a 30-minute window, which comfortably outlasts any reasonable mempool polling/retry latency.
- This endpoint must be reachable without authentication - the mempool instance calling it has no credential to present, and does not need one; the security comes from the fact that only you can make your own server answer this way.
- Recommended timeout for the caller side is 5 seconds; implement accordingly (answer fast, do not block on slow I/O).

## 5. Step 4: Register Yourself

Fork [`elektron-net-registry`](https://github.com/kutlusoy/elektron-net-registry), add your line to `pools.txt`:

```
"PPLNS", "Your Pool Name", "https://your-pool.example";
```

or `"SOLO"` if you are a solo pool. Open a pull request. There is no automatic approval process; a human reviews additions the same way the existing, older `pools-v2.json` registry curation already works elsewhere in this project.

## 6. Checklist

- [ ] Fetch and periodically refresh `mempools.txt`, with a local on-disk cache read first on startup
- [ ] Report every found block to every known mempool instance, fire-and-forget, short timeout
- [ ] Implement `GET /api/pool/identity/confirm?blockHash=<hex>`, answering `true` only for blocks you genuinely, recently found
- [ ] Registered your own `pools.txt` entry
- [ ] Verified end to end: mine/submit a real (or regtest) block, confirm a mempool instance's dashboard shows your pool's name
