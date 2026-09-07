# Elektron Net Registry - Integration Guide for Mempool/Explorer Software

- **Version:** 1.0
- **Date:** September 6, 2026
- **Audience:** developers of any Elektron Net block explorer / mempool visualizer wanting to attribute self-reported pool blocks, not only `elektron-net-mempool`
- **Reference implementation:** [`elektron-net-mempool`](https://github.com/kutlusoy/elektron-net-mempool) - `backend/src/tasks/pool-registry-updater.ts`, `backend/src/api/pool-registry.routes.ts`, `backend/src/api/pool-registry-parser.ts`
- **See also:** [`integration-pools.md`](./integration-pools.md) (the other side of this protocol), `elektron-net-mempool`'s own `doc-elektron/guideline-pool-registry-reporting.md`, [`../README.md`](../README.md) (the registry file format)

- Never use the em dash character in this document or its follow-up code comments; use a hyphen and spaces instead, as done throughout.

---

## 1. What This Solves

A block explorer cannot reliably show which pool found a block unless the pool is in a curated address registry, or proves its identity some other way. This protocol lets a pool report a found block to your explorer directly, and lets you verify that claim yourself, off-chain, with no wallet, signature, or shared secret - just an HTTP callback to the URL the pool itself registered. Never attribute a block from a report alone; the verification step in Section 4 is what makes this safe to trust.

## 2. Step 1: Read `pools.txt`

Fetch `<REGISTRY_URL>/pools.txt` (default `REGISTRY_URL` is `https://raw.githubusercontent.com/kutlusoy/elektron-net-registry/main`; make this configurable, never hardcode it as the only option). One entry per line:

```
"Type", "Name", "URL";
```

`Type` is `"PPLNS"` or `"SOLO"`, not currently used by the reporting/verification flow itself (kept for possible future filtered listings). Parsing note: only the quoted substrings on each line matter - extract every `"..."` token and ignore whatever separates them (comma, semicolon, or nothing); this way your parser keeps working even if a fork of the file changes the separator style. Skip any line that does not yield exactly three non-empty quoted fields rather than failing the whole file.

- **Poll interval:** every 15 minutes is what the reference implementation uses. The file is tiny, so a plain refetch on every poll is enough - no conditional-fetch/diffing logic needed.
- **Local cache:** write the last successfully fetched file to local disk (e.g. next to your existing cache directory), and read that local copy first on startup, before attempting any network request, so you have a usable pool list immediately even if the registry host is unreachable at boot. Never let a failed or empty-parsing fetch overwrite an already-good in-memory map or the on-disk cache.
- Keep the parsed result as a map keyed by `Name`, since that is what an incoming report will reference (Section 3).

## 3. Step 2: Receive Block Reports

Expose an endpoint pools can report to:

```
POST /api/v1/pool-registry/report
Content-Type: application/json

{
  "name": "<the reporting pool's exact Name field>",
  "blockHash": "<64-character hex block hash>"
}
```

Validate the payload (non-empty `name`, `blockHash` matching `^[a-f0-9]{64}$`, case-insensitive) and reject anything else with `400`. Look up `name` in **your own locally-synced copy** of `pools.txt` (Section 2) to get that pool's registered URL. If the name is not in your synced list, respond `404` - do not fall back to any URL the caller might have also included in the request body; only ever use the registry's own mapping. This is the whole point: the URL you call back to in Section 4 must be one your own trusted source (the registry) says belongs to that name, never one an untrusted caller supplies about itself.

## 4. Step 3: Verify Before Attributing (Mandatory)

Call back to the pool's own registered URL:

```
GET <registered_pool_url>/pool/identity/confirm?blockHash=<same hash>

200 OK
{
  "confirmed": true | false,
  "name": "...",
  "url": "..."
}
```

- Recommended timeout: 5 seconds. Treat a timeout, connection error, non-2xx response, or `confirmed !== true` all the same way: **do not attribute the block.** There is no partial trust here - either the pool's own server confirms it, or you do nothing.
- Only on `confirmed === true` should you proceed to attribute the block to that pool in whatever data model your software uses.
- Skipping this step (trusting the report directly) reintroduces exactly the spoofing problem this design exists to avoid - anyone could then claim any block under any registered pool's name.

## 5. Step 4: Attribute the Block

How you record the attribution is entirely up to your own software's data model; this protocol does not prescribe one. For reference, `elektron-net-mempool` resolves or creates a row for the pool (keyed by name, reusing its self-reported-pool mechanism so existing ranking/hashrate code picks it up automatically) and updates that specific block's pool reference by hash. The registry protocol's job ends at Section 4 - a confirmed report is a fact you can act on however fits your system.

## 6. Step 5 (Optional): Register Yourself

If you want pools to discover and report to your instance, fork [`elektron-net-registry`](https://github.com/kutlusoy/elektron-net-registry) and add your own line to `mempools.txt`:

```
"Your Explorer Name", "https://your-explorer.example";
```

Open a pull request. Not required to consume the registry (Sections 2-5 work regardless), only to be reachable by pools that discover mempool instances the same way.

## 7. Checklist

- [ ] Fetch and periodically refresh `pools.txt`, with a local on-disk cache read first on startup
- [ ] Implement `POST /api/v1/pool-registry/report`, validating the payload and resolving the URL only from your own synced registry map
- [ ] Call back to that URL's `/pool/identity/confirm` before ever attributing anything, treating any non-`true` outcome as "do not attribute"
- [ ] Wired confirmed reports into your own pool-attribution data model
- [ ] Verified end to end against a real (or test) pool implementation
