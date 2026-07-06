# Boedelverdeling

Static web app for a family estate bidding process. GitHub is used as both host and datastore. No backend server.

## Architecture

Two repositories, split by trust boundary:

| Repo | Owner | Visibility | Purpose |
|---|---|---|---|
| `boedel-ui` (this repo) | `jandekort` | Public | Static site source, served via GitHub Pages. Contains app code and non-sensitive config (`items.json`, `heirs.json`, `settings.json`). No bid data. |
| `boedel-data` | dedicated second account | Private | Holds `bids.jsonl` only. Read/write exclusively via the GitHub Contents API, authenticated with one fine-grained Personal Access Token (PAT) shared within the family. |

Rationale: GitHub Pages Free serves only from public repos, and even Pro-tier private-repo Pages still publishes the site itself with no viewer auth. Splitting the repos keeps the served UI public (harmless, no data) while keeping actual bid data behind normal private-repo access control (free, no plan requirement).

`boedel-ui` has no knowledge of `boedel-data` beyond one hardcoded config value: `OWNER` in `index.html`. There is no repo dependency, submodule, or build-time link between them. `boedel-data` is addressed purely at runtime via the GitHub REST API.

## Stack

- Frontend: single HTML file (`index.html`), vanilla JS, no framework, no build step.
- Hosting: GitHub Pages, deploy from `main` / `/(root)`.
- Data layer: GitHub Contents API (`GET`/`PUT` on `boedel-data:bids.jsonl`), used as an append-only ledger.
- Auth: one shared fine-grained PAT from a dedicated second account that owns `boedel-data`. Its only job is keeping the bid data invisible to non-family members; it is not a per-person identity. Each family member pastes it once in the browser; it is stored in `localStorage`. No server-side secret handling, and family members do not need GitHub accounts.
- Concurrency control: optimistic locking via the Contents API's `sha` parameter (see `ghGetBids`/`ghPutBids`/`placeBid` in `index.html`).
- Identity: purely the self-declared name (dropdown from `heirs.json`; read live from the dropdown at bid time, deliberately never persisted, so every page load starts on the placeholder "Kies een naam" and nobody accidentally bids under someone else's name). Each bid must additionally be confirmed by re-typing the bidder's name (case- and whitespace-insensitive). Because everyone commits with the same shared token, all commits have the same author — the git history proves *when* each bid was appended, not *who* placed it. Who bid is on the honor system, which is the point of a family auction.
- Log export: `Logboek opslaan` downloads the raw `bids.jsonl` as a local file (e.g. to Downloads), generated client-side via a Blob. No email API, no backend.

## Files in this repo

```
index.html      - app: two tabs (Biedingen, Logboek), bid form, GitHub API calls
items.json      - list of estate items (id, desc)
heirs.json      - list of heirs (name, email)
settings.json   - { "minIncrement": <EUR>, "minPercent": <%> } (defaults 5 / 5)
```

## Data model (in `boedel-data:bids.jsonl`)

One JSON object per line, append-only:

```json
{"ts":"2026-07-05T14:32:10Z","heir":"Claar","item":"A02","bid":475}
```

Validity rule (computed client-side on every load, never trusted from cache): the minimum raise is `max(minIncrement, ceil(minPercent% of current valid high))`, so a bid is valid iff `bid >= high + max(minIncrement, ceil(high * minPercent / 100))`, or `bid >= minIncrement` if no valid prior bid exists for that item. Both knobs are independently configurable in `settings.json` (defaults: `minIncrement` €5, `minPercent` 5%). Invalid bids remain in the log (append-only, never deleted) but do not affect the displayed leader/high bid. The current minimum for each item is shown as a placeholder in its bid field.

## Setup

1. `boedel-ui`: create a new **public** repo. Add `index.html`, `items.json`, `heirs.json`, `settings.json` at root. Settings > Pages > Deploy from branch > `main` / `/(root)`.
2. In `index.html`, set `OWNER` to the username of the dedicated second account that owns `boedel-data`.
3. `boedel-data`: create a new **private** repo under the second account. Add an empty `bids.jsonl` file at root (required so the first `GET` returns a valid `sha`).
4. Second account: Settings > Developer settings > Fine-grained tokens > Generate new.
   - Resource owner: the second account itself.
   - Repository access: only `boedel-data`.
   - Permissions: Contents - Read and write.
   - Expiration: pick a date shortly after the auction is expected to end.
5. Share this one token privately within the family (e.g. family group chat or in person — never commit it to a repo; GitHub auto-revokes tokens it detects in public repos). Each family member pastes it in the site once and clicks Opslaan. No collaborator invites, no per-person tokens, no GitHub accounts needed for family members.
6. If the token ever leaks, revoke it in the second account and share a fresh one; nothing else changes.

## Testing

1. Open `https://<username>.github.io/boedel-ui/` in two separate browser profiles or incognito windows.
2. In both windows: paste the shared PAT, click Opslaan, then select a different heir name in each.
3. Place a bid from each window, refresh, confirm leader and high bid match in both.
4. Submit a bid below the required minimum (shown as placeholder in the bid field). Confirm it appears in Logboek marked as invalid (`NEE`) and does not change the leader.
5. Check `https://github.com/<owner>/boedel-data/commits/main/bids.jsonl`, confirm one commit per bid with the heir name and amount in the commit message.
6. Click `Logboek opslaan`, confirm a `bids-<date>.jsonl` file downloads containing the full raw JSONL log.

## Known limitations (by design, for MVP scope)

- GitHub's CDN caches Contents API reads; a just-committed bid can take up to roughly a minute to appear even after a manual refresh. No push mechanism exists on static hosting; this is inherent to the architecture, not a bug to fix without adding a backend.
- No server-side enforcement of the append-only rule or the minimum-increment rule. Both are checked client-side and by convention. A user with write access to `boedel-data` could in principle edit history directly on github.com. Mitigation: this is tamper-evident (visible in commit history), not tamper-proof.
- Identity is entirely self-declared: the shared token means anyone holding it can bid under any name. Commit history proves when bids were appended, not who placed them. Acceptable within a trusting family; the token's only security job is keeping outsiders from reading or writing the data.
- `Logboek opslaan` exports the log as last loaded in the browser (click Vernieuwen first for the freshest state); the canonical record remains `boedel-data:bids.jsonl` itself.
