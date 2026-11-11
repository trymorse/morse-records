# morse-records
Public pointers to Morse door boot records in Sigstore's log, with an archive of each entry

## Files

- `prod.jsonl` and `dev.jsonl` hold one pointer a line, `{"log": "log2025-1.rekor.sigstore.dev", "index": 194119183, "spki_sha256": "…", "boot": "…"}`. `log` is the Rekor v2 shard, `index` the entry, `spki_sha256` the SHA-256 of the door key's SubjectPublicKeyInfo, and `boot` the 16-hex HPKE key id that the record's Nitro document names. Lines are only appended.
- `leaves/<log>/<index>.json` holds each pointer's entry as a Rekor v2 `TransparencyLogEntry`: the canonical body, its inclusion proof, and the cosigned checkpoint the proof is against. It verifies with the log's key from Sigstore's TUF repository and the witnesses' keys, so it needs no trust in its writer. It keeps the entry readable after a shard stops serving its tiles.

A pointer is a hint. [`check-door.py`](https://github.com/trymorse/morse-enclave) verifies the entry it names, and a wrong pointer costs only a failed verification.

## Workflows

`pointers.yml` runs every 10 minutes. It reads `/v1/attestation` for prod (`api.trymorse.ai`) and dev (`vault.watermark.ac`), and takes the receipts from each instance's `door` object. For each receipt not yet listed, it reads the entry from the shard's tiles and checks that it carries a door boot record of that environment for the door's key. It then appends the pointer, archives the leaf, and pushes to `main`. An environment that does not answer is skipped with a warning. The dev endpoint is behind Cloudflare Access and does not answer GitHub's runners. The workflow signs nothing, reads no repository variable or secret, and pins every action by commit SHA. Ruleset "history is permanent" refuses force-pushes and deletions on every branch, so `main` only grows.

`check.yml` runs on each pull request. It runs the lint copied from `trymorse/morse-pins`, and it runs `pointers.yml`'s steps against recorded attestations, checking each archived leaf with `check-door.py`.
