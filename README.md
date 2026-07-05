# Boedelverdeling

Static web app for a family estate bidding process. GitHub is used as both host and datastore. No backend server.

## Architecture

Two repositories, split by trust boundary:

| Repo | Owner | Visibility | Purpose |
|---|---|---|---|
| `boedel-ui` (this repo) | `jandekort` | Public | Static site source, served via GitHub Pages. Contains app code and non-sensitive config (`items.json`, `heirs.json`, `settings.json`). No bid data. |
| `boedel-data` | `jandekort` | Private | Holds `bids.jsonl` only. Read/write exclusively via the GitHub Contents API, authenticated per-user with a Personal Access Token (PAT). |

Rationale: GitHub Pages Free serves only from public repos, and even Pro-tier private-repo Pages still publishes the site itself with no viewer auth. Splitting the repos keeps the served UI public (harmless, no data) while keeping actual bid data behind normal private-repo access control (free, no plan requirement).

`boedel-ui` has no knowledge of `boedel-data` beyond one hardcoded config value: `OWNER` in `index.html`. There is no repo dependency, submodule, or build-time link between them. `boedel-data` is addressed purely at runtime via the GitHub REST API.

## Stack

- Frontend: single HTML file (`index.html`), vanilla JS, no framework, no build step.
- Hosting: GitHub Pages, deploy from `main` / `/(root)`.
- Data layer: GitHub Contents API (`GET`/`PUT` on `boedel-data:bids.jsonl`), used as an append-only ledger.
- Auth: GitHub PATs, entered by each user in the browser, stored in `localStorage`. No server-side secret handling.
- Concurrency control: optimistic locking via the Contents API's `sha` parameter (see `ghGetBids`/`ghPutBids`/`placeBid` in `index.html`).
- Identity: self-declared name (dropdown from `heirs.json`) plus whichever PAT's owner actually authors the commit. Not cryptographically bound; the git commit author is the real audit signal, the dropdown name is a convenience label matched against it in `Biedingen`.
- Email export: `mailto:` link generated client-side in `Logboek opslaan`. No email API, no backend.

## Files in this repo

```
index.html      - app: two tabs (Biedingen, Logboek), bid form, GitHub API calls
items.json      - list of estate items (id, desc)
heirs.json      - list of heirs (name, email)
settings.json   - { "increment": <minimum bid increment in EUR> }
```

## Data model (in `boedel-data:bids.jsonl`)

One JSON object per line, append-only:

```json
{"ts":"2026-07-05T14:32:10Z","heir":"Claar","item":"A02","bid":475}
```

Validity rule (computed client-side on every load, never trusted from cache): a bid is valid iff `bid >= (current valid high for that item) + increment`, or `bid >= increment` if no valid prior bid exists for that item. Invalid bids remain in the log (append-only, never deleted) but do not affect the displayed leader/high bid.

## Setup

1. `boedel-ui`: create a new **public** repo. Add `index.html`, `items.json`, `heirs.json`, `settings.json` at root. Settings > Pages > Deploy from branch > `main` / `/(root)`.
2. In `index.html`, set `OWNER` to the GitHub username that owns `boedel-data` (e.g. `jandekort`).
3. `boedel-data`: create a new **private** repo, owned by the same account as step 2. Add an empty `bids.jsonl` file at root (required so the first `GET` returns a valid `sha`).
4. `boedel-data` > Settings > Collaborators > add each other heir's GitHub account. Each must accept the invite.
5. Owner account: Settings > Developer settings > Fine-grained tokens > Generate new.
   - Resource owner: the owner account itself.
   - Repository access: only `boedel-data`.
   - Permissions: Contents - Read and write.
6. Each collaborator account: fine-grained tokens **do not support** repos owned by another personal account where you're only a collaborator (confirmed GitHub limitation, not an org-only restriction). Use a **classic** token instead:
   - Settings > Developer settings > Personal access tokens > Tokens (classic) > Generate new token (classic).
   - Scope: `repo` (full control of private repositories). This grants access to all repos the account can already reach, in practice just `boedel-data`.
   - No resource-owner selection step; classic tokens are scoped by permission, not by explicit repo, so there is nothing to "approve" from the owner side.

## Testing

1. Open `https://<username>.github.io/boedel-ui/` in two separate browser profiles or incognito windows.
2. Window 1: select an heir name, paste the owner's fine-grained PAT, click Opslaan. Window 2: select a different heir, paste a collaborator's classic PAT, click Opslaan.
3. Place a bid from each window, refresh, confirm leader and high bid match in both.
4. Submit a bid below `high + increment`. Confirm it appears in Logboek marked as invalid (`NEE`) and does not change the leader.
5. Check `https://github.com/<owner>/boedel-data/commits/main/bids.jsonl`, confirm two distinct commit authors (one per PAT owner). If both commits show the same author, identity is not actually bound per-person and the audit-trail advantage over a shared spreadsheet is lost; recheck step 6 of Setup.
6. Click `Logboek opslaan`, confirm a `mailto:` draft opens with a per-item summary followed by the full raw JSONL log.

## Known limitations (by design, for MVP scope)

- GitHub's CDN caches Contents API reads; a just-committed bid can take up to roughly a minute to appear even after a manual refresh. No push mechanism exists on static hosting; this is inherent to the architecture, not a bug to fix without adding a backend.
- No server-side enforcement of the append-only rule or the minimum-increment rule. Both are checked client-side and by convention. A user with write access to `boedel-data` could in principle edit history directly on github.com. Mitigation: this is tamper-evident (visible in commit history), not tamper-proof.
- Identity is only as strong as PAT custody. Anyone holding another person's PAT can commit under that person's authorization scope, though the commit author will still be tied to whichever account's PAT was used.
- `mailto:` has body-length limits in some clients; very long logs may truncate. No current fallback other than manual copy-paste.
