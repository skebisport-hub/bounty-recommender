# bounty recommender

an AI agent that takes a wallet address and returns ranked bounty recommendations from the live
[AI Bounty Board](https://bounty.owockibot.xyz) (owockibot, Base network, x402/USDC payments).

no backend, no build step, no npm install — it's one HTML file that calls the live API directly
from the browser. open `index.html` locally or host it on GitHub Pages and it works.

**live demo:** `https://<your-username>.github.io/<repo-name>/` (see deploy steps below)

## what it does

1. fetches `GET /bounties` from `https://bounty.owockibot.xyz` — every bounty ever posted, open and closed
2. builds a profile for the wallet you enter: which of those bounties it previously claimed/completed, what tags those were, and its average historical reward
3. pulls the wallet's general on-chain activity on Base from the public Blockscout API (tx count) as a rough experience/readiness signal
4. scores every currently open bounty against that profile and returns them ranked highest to lowest

if the board has zero open bounties at the moment you check (it does sometimes — it's a small live
board), the demo falls back to ranking *all* posted bounties instead, so you can still see the
scoring logic work end to end. that's flagged clearly in the UI, not hidden.

## the scoring model (100 points, in `scoreBounty()`)

| signal | weight | what it measures |
|---|---|---|
| tag match | 0–50 | overlap (Jaccard) between the open bounty's tags and the tags of bounties this wallet has completed before |
| reward fit | 0–20 | how close the bounty's reward is to the wallet's historical average payout |
| chain activity | 0–20 | `log2(tx_count)`-scaled — a wallet with more Base activity scores higher as a proxy for reliability/experience |
| freshness / cold start | 0–10 | for wallets with no completion history, bounties are ranked by reward size as a sane default until real history exists |

for a wallet with no history on the board *and* no Base activity, the ranking degrades gracefully
to "biggest open bounties first" rather than erroring out — try a random `0x000...000` address to
see that path.

## on the live API's schema

owockibot's `/bounties` response format isn't publicly documented beyond the endpoint list on their
homepage (`GET /bounties`, `GET /stats`, `POST /bounties/:id/claim`, `POST /bounties/:id/submit`).
the code doesn't assume one exact field name for "who completed this bounty" — it checks a list of
likely names (`claimed_by`, `claimant`, `winner`, `winner_address`, `submitter`, `wallet`, etc.) in
`CLAIMANT_FIELDS` near the top of `index.html`. if the live payload uses something else, that's the
one line to edit — no other logic needs to change.

use the "raw API calls made" expandable at the bottom of the page to see exactly what the board
returned for any given run, which makes it fast to spot a field-name mismatch.

## deploy it (GitHub web UI only, no terminal needed)

1. create a new repo on github.com → **New repository** → name it (e.g. `bounty-recommender`) → Public → Create
2. **Add file → Upload files** → upload `index.html` and this `README.md` → Commit
3. **Settings → Pages** → under "Build and deployment", Source: **Deploy from a branch** → Branch: `main`, folder: `/ (root)` → Save
4. after ~1 minute, your live link appears at `https://<your-username>.github.io/bounty-recommender/`

to edit anything later (the scoring weights, the field-name list, styling): open `index.html` in the
GitHub web editor (pencil icon), change it, commit to `main` — Pages redeploys automatically.

## known limitations

- **CORS**: if `bounty.owockibot.xyz` or `base.blockscout.com` ever add stricter CORS policies, the
  browser fetch will fail — the UI surfaces that as an error rather than hanging silently.
- **schema assumptions**: see the section above. this was built by reading the board's public
  homepage and third-party MCP-server docs that wrap the same API, not an official API reference,
  since owockibot doesn't publish one.
- **wallet history is board-scoped**: "past completions" only counts bounties completed *on this
  specific board*. it doesn't reach into other bounty platforms.
- Blockscout's address endpoint is used for chain activity only — no wallet write access, no
  transaction signing, nothing that touches funds. this tool only reads and ranks.
