# Elektron Net Registry

A public, plain-text registry of Elektron Net mining pools and block explorer
(mempool) instances. Any pool or mempool instance can list itself here so the
rest of the network can find it, without touching consensus, without any
on-chain data, and without a wallet.

This replaces an earlier approach that embedded pool name/URL directly in the
coinbase transaction. That broke the network's per-block UTXO attestation (see
`doc-elektron/fix-report-pool-identity-utxo-attestation.md` in
[`elektron-net-pool`](https://github.com/kutlusoy/elektron-net-pool),
[`elektron-net-ppool`](https://github.com/kutlusoy/elektron-net-ppool), and
[`elektron-net-mempool`](https://github.com/kutlusoy/elektron-net-mempool) for
the full story) and was reverted. This registry is the free, wallet-independent
replacement, currently being planned across those same three repos plus
[`elektron-net-stack`](https://github.com/kutlusoy/elektron-net-stack) (see each
repo's own `doc-elektron/guideline-pool-registry-reporting.md`).

## Who can apply

Not every mempool instance or pool can list itself here. To keep the registry
useful and trustworthy, entries must meet the following criteria before
opening a pull request.

### Mempools

- Should have been in live operation for at least 1 month.
- Should support the [Elektron Net - elektron-net-mempool Pool Registry Reporting](https://github.com/kutlusoy/elektron-net-mempool/blob/main/doc-elektron/guideline-pool-registry-reporting.md) guideline.
- Can be the official [`elektron-net-mempool`](https://github.com/kutlusoy/elektron-net-mempool) repo, or other mempool software that is compatible with and adapted for Elektron Net.

### Solo Pools and PPLNS Pools

- The pool software should have a public URL and should have been in live operation for at least 1 month.
- Should have at least 5 continuously active miners on the pool.
- Should support the Elektron Net Pool or PPLNS Registry Reporting guideline, either [`elektron-net-pool`](https://github.com/kutlusoy/elektron-net-pool/blob/main/doc-elektron/guideline-pool-registry-reporting.md) or [`elektron-net-ppool`](https://github.com/kutlusoy/elektron-net-ppool/blob/main/doc-elektron/guideline-pool-registry-reporting.md).
- Can be the official Pool or PPLNS repo ([`elektron-net-pool`](https://github.com/kutlusoy/elektron-net-pool) or [`elektron-net-ppool`](https://github.com/kutlusoy/elektron-net-ppool)), or other pool software that is compatible with and adapted for Elektron Net.

## Files

- **`pools.txt`** - one line per pool, both PPLNS (`elektron-net-ppool`) and
  solo (`elektron-net-pool`) entries together, tagged by type.
- **`mempools.txt`** - one line per known block explorer / mempool instance.

`mempools.txt` uses a deliberately simple format, one entry per line, fields
comma-separated, each line ending in `;`:

```
"Name", "URL";
```

`pools.txt` uses the same format with one extra field, the pool type, always
first, either `"PPLNS"` or `"SOLO"`:

```
"Type", "Name", "URL";
```

Putting the type first keeps it in a fixed position regardless of what the
name/URL contain, so tools can filter or sort by type without parsing the
whole line. It exists for future use (e.g. separate PPLNS/solo listings); the
current reporting/verification design does not depend on it.

Example (`pools.txt`):

```
"SOLO", "Kutlusoy's Solo Pool", "https://solopool3.elektron-net.org";
"PPLNS", "Bob's PPLNS Pool", "https://pool.bobtheguy.com";
```

Example (`mempools.txt`):

```
"Bob's MemPool", "https://mempool.bobtheguy.com";
```

No tokens, no signatures, no other fields. Software consuming these files is
expected to verify a claim (e.g. "this block belongs to pool X") by calling
back to the URL listed here for that name, rather than trusting either file's
content or any report on its own. See `doc-elektron/integration-pools.md` and
`doc-elektron/integration-mempools.md` for the exact HTTP protocol third-party
pool and mempool-explorer software can implement to participate.

## Adding an entry

Fork this repository, add your line to the appropriate file, and open a pull
request. Both files simply grow over time; entries are not automatically
removed.

## Official Elektron Net links

- Website: https://elektron-net.org
- X (Twitter): @elektronnet and @kutlusoy
- Telegram: @elektronnet
