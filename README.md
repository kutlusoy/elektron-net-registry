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

## Files

- **`pools.txt`** - one line per pool, both PPLNS (`elektron-net-ppool`) and
  solo (`elektron-net-pool`) entries together.
- **`mempools.txt`** - one line per known block explorer / mempool instance.

Both files use the same, deliberately simple format, one entry per line:

```
"Name"; "URL";
```

Example (`pools.txt`):

```
"Kutlusoy's Solo Pool"; "https://solopool3.elektron-net.org";
"Bob's PPLNS Pool"; "https://pool.bobtheguy.com";
```

Example (`mempools.txt`):

```
"Bob's MemPool"; "https://mempool.bobtheguy.com";
```

No tokens, no signatures, no other fields. Software consuming these files is
expected to verify a claim (e.g. "this block belongs to pool X") by calling
back to the URL listed here for that name, rather than trusting either file's
content or any report on its own.

## Adding an entry

Fork this repository, add your line to the appropriate file, and open a pull
request. Both files simply grow over time; entries are not automatically
removed.

## Official Elektron Net links

- Website: https://elektron-net.org
- X (Twitter): @elektronnet and @kutlusoy
- Telegram: @elektronnet
