# CrashTop round verifier

A single static page that checks a CrashTop round against the published hash chain and replays the
results in the browser. No database, no login, no analytics, no third-party script.

- **Verifier:** <https://crash-top.github.io/verifier/>
- **Game:** <https://crashtop.com>
- **Help and guides:** <https://help.crashtop.com>

The page is open source precisely so that anyone can read what it does and run it themselves: copy
this repository, open `index.html`, and compare the numbers with the ones the game showed.

## What it proves

Every round is committed before betting closes and revealed after the bust:

1. **Commitment** — `SHA-256(seed)` equals the hash the game published for that round.
2. **Replay** — the result the game showed is exactly what that seed produces.
3. **Chain walk** — hashing the round hash backwards reaches the chain anchor that was published
   before the chain's first round.
4. **Position** — the hash sits at the round number it claims, in the chain it claims.

Then it prints the selected round and the rounds before it, each with its hash and replayed result.

## How to check a round

1. Open **Round info** in the game and copy the round hash and the round number.
2. Paste both into the form, or press **Verify hash** in the game — with this verifier set as the
   game's verifier address, that link opens the page already filled in.
3. Press **Verify** and compare the replayed rows with the results the game showed.

A green **Verified** verdict means every check above passed. A red verdict lists the check that
failed and what was expected instead; nothing is hidden and nothing is corrected silently.

### Example link

The values below come from a small illustrative chain, not from a live round, so the link opens a
complete, working report you can look at before you try a real one:

<https://crash-top.github.io/verifier/#round=10523&hash=7f67379dca657c6abf6210ada0d41ef70ece5c0ffdde07141e56aa13abb96643&seed=195602707f3fb0fbe99b39b6f5687c14764f7930b93aa9df4c1cfa43f2724d34&salt=2ecdbc05dfc2be7fe3fa23bd8ef09bdbd36af90b7e0fb3a3a47a22c059524d51&terminal=9586accd110b15f80aeb260139a666fec1487a5674310bc5bc253029353d75a8&result=1.66>

Everything after `#` stays in the browser. The fragment is never sent to the server, so the round
hash, the seed and the salt never appear in any access log.

## URL parameters

The game builds these; you can also write them yourself.

| Parameter  | Required | Meaning                                                                       |
| ---------- | -------- | ----------------------------------------------------------------------------- |
| `hash`     | yes      | The 64-character hash published for the round.                                |
| `round`    | yes      | The round number. Labels the rows and checks the chain position.              |
| `seed`     | no       | The server seed revealed after the round. Adds the round's own result.        |
| `chain`    | no       | Chain id from `chains.js`. Omit it and the page detects the chain by walking. |
| `rows`     | no       | How many rounds to replay. Default 10, maximum 50.                            |
| `salt`     | no       | Chain salt, for a chain that is not in `chains.js` yet.                       |
| `terminal` | no       | Chain anchor, for a chain that is not in `chains.js` yet.                     |
| `result`   | no       | The result the game showed, so the page can cross-check the replay.           |

Without the seed the table starts at `round - 1`, because the published hash of round `n` is the
seed of round `n - 1`.

## The math

The chain is built downwards from a random top seed:

```
seed[j-1] = SHA-256(seed[j])            terminal = SHA-256(seed[0])
hash[j]   = SHA-256(seed[j]) = seed[j-1]
X         = int(HMAC-SHA256(salt, seed[j])[0..52 bits], 16) / 2^52
result[j] = max(1, floor(99 / (1 - X)) / 100)
```

The HMAC key is the chain salt; the message is the round's server seed. Because `hash[j]` is
`seed[j-1]`, hashing a published hash backwards walks the chain, and hashing it exactly `index`
times reaches the anchor.

## What it does not prove

A static page cannot prove that a file was not rebuilt after the fact. What it proves is that the
rounds are links of one chain, that every result replays from its seed, and that the chain reaches
an anchor. The anchor is the trust root: compare it with the value the game published before the
chain started, and with the value in this repository's `chains.js`.

The salt is a Bitcoin block hash. The page links to a public block explorer so you can confirm the
block yourself; it never queries an explorer API.

## Privacy

- Nothing is uploaded. The only network request is the one that loaded the page.
- No storage, no cookies, no analytics, no fonts or scripts from another origin.
- A Content-Security-Policy blocks connections, frames and form submissions.
- The round data travels in the URL fragment and in the form only, so it is never part of a request
  the server can see.

## `chains.js`

`chains.js` holds public chain commitments only:

```js
self.CT_CHAINS = [
  {
    id: 206,
    terminalHash: "…",
    salt: "…",
    saltHeight: 974329,
    length: 5000,
    committedAt: "2026-11-03T01:52:06.285Z",
    firstRound: 30001,
  },
];
```

No seed, no per-round record, no player data. It is generated from the game's public endpoints —
`/api/fair/chain`, `/api/history?limit=1` and `/api/fair/round/:id` — which need no credential and
expose no unfinished round:

```
node tools/export.mjs
```

The tool merges by chain id, so an older chain keeps its first round and stays verifiable after it
leaves the live window. `https://crashtop.com` is the default origin; `--base` and `--limit` exist
only for a staging copy:

```
node tools/export.mjs --base https://crashtop.com --limit 20
```

`tools/check-public.mjs` refuses a `chains.js` that carries any field beyond the public commitment,
and the publish workflow runs it before the artifact is uploaded.

## Run it locally

```
node --test                     # the full suite
node tools/check-public.mjs     # the publish guard
```

Then open `index.html` in a browser. It is plain files: no build, no bundler, no dependency, no
package to install.

The suite cross-checks SHA-256 and HMAC-SHA256 against Node's crypto, builds a synthetic chain the
way the server does, replays rounds with and without a seed, and checks that `chains.js` refuses
seed-shaped fields.

## Repository layout

| File                          | Role                                                                     |
| ----------------------------- | ------------------------------------------------------------------------ |
| `index.html`                  | The page. CSP locks it to same-origin scripts, no network.               |
| `styles.css`                  | Presentation.                                                            |
| `verify.js`                   | SHA-256, HMAC-SHA256, crash-point math, chain replay. No dependencies.   |
| `worker.js`                   | Runs the replay off the render thread and reports progress.              |
| `app.js`                      | Form, shared link, rendering. Text nodes only.                           |
| `chains.js`                   | Generated public chain commitments.                                      |
| `tools/export.mjs`            | Refreshes `chains.js` from the game's public endpoints.                  |
| `tools/check-public.mjs`      | Refuses a `chains.js` that carries a field beyond the public commitment. |
| `.github/workflows/pages.yml` | Tests, checks `chains.js`, and publishes the site.                       |
| `test/verify.test.mjs`        | Correctness tests.                                                       |
| `test/chains.test.mjs`        | Tests for the publish guard.                                             |

## Official address

The verifier lives at <https://crash-top.github.io/verifier/> and the game links to it from
the round panel. Any other page that claims to verify CrashTop rounds is not ours: a copy can be
changed to always answer "verified". Check the address in the browser bar, or open the verifier from
the game instead of from a message.

## Reporting a problem

Security and correctness questions go through <https://help.crashtop.com>; `SECURITY.md` describes
what may and may not enter this repository.

## License

MIT — see `LICENSE`.
