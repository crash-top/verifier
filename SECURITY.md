# Security policy

## Reporting a problem

Write to us through <https://help.crashtop.com>. Please do not open a public issue for a problem
that could affect players; a public report is fine for a documentation mistake, a broken link or a
failing test.

What is worth reporting here:

- A round that the verifier says is fine but that does not replay from its seed.
- A `chains.js` entry that carries anything beyond a public commitment.
- A path in the page that could reach the network, store data or evaluate remote input.
- Any text, file or commit in this repository that looks like a credential, a seed or player data.

## What this repository must never contain

`chains.js` is served to every visitor, so it may hold public commitments only: the chain id, the
anchor hash (`terminalHash`), the salt, the salt block height, the chain length, the commit time and
the first round. No server seed, no per-round record, no player data, no credential, no environment
file. `tools/check-public.mjs` enforces the field list on every push, and a manual run checks a local
file:

```
node tools/check-public.mjs
```

## What the verifier cannot do

It is a static page. It reads no account, holds no balance and cannot move funds. Its only job is to
replay a published hash chain and print what it found, so a read-only audit of the game is possible
without any access to the server.
