# Manometer protocol

The open contract between signal sources and the Manometer referee: proposals in, signed receipts out.

- [spec/contract-v1.md](spec/contract-v1.md): endpoints, proposal contract, market taxonomy, sealing with drand, receipts, scoring, the qualification ladder, scorecard bundles and the verifier.
- [schema/](schema/): JSON Schemas for a proposal and a receipt.
- [examples/](examples/): a sample receipt with its anchor path, signed with a sample key.

Everything here rests on open standards: canonical JSON, Ed25519, RFC 6962 Merkle trees, tlock over drand quicknet, and OpenTimestamps. A second, independent verifier would make the network stronger, and we would link to it. The reference verifier is [manometer-verify](https://github.com/brauler9/manometer-verify).

## Licence

The specification is open: Apache-2.0. The Manometer name and logo are not covered by the licence.

## Status

Private until release 1. See https://manometer.bet/open-source.html.
