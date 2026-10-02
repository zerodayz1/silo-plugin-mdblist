> [!IMPORTANT]
> **This plugin is retired. Use the official MDBList plugin instead:**
> [Silo-Server/silo-plugin-metadata-mdblist](https://github.com/Silo-Server/silo-plugin-metadata-mdblist),
> available in Silo's built-in plugin catalog as **MDBList Ratings**.
>
> The official plugin does everything this one did and more:
> - **Backfill without a rescan.** It opts into Silo's hourly *Bulk Metadata
>   Enrichment* task, which fills ratings and age ratings for titles already in
>   your library without refreshing their metadata. Run it now from
>   Admin → Tasks.
> - **Batched API calls**, up to 100 titles per MDBList request, with a pause
>   (not a failure) when your daily quota runs out.
> - **Rotten Tomatoes on current Silo.** Since Silo build-1058, Silo only stores
>   RT scores from a plugin that declares them. This plugin does not, so its RT
>   scores are dropped. (The official plugin's declaration lands in its 0.5.0.)
> - Age ratings (Common Sense), certifications, per-source vote counts.
>
> **Switching:** both plugins use the plugin id `silo.mdblist`, so installing
> the official one from the catalog replaces this one in place. Then re-enter
> your API key (its setting is called *MDBList Account*), check that **MDBList**
> sits below TMDB/TVDB in each library's provider chain, and remove any leftover
> "IMDB"/"TMDB" rows from this plugin. Existing ratings are kept.

# silo-plugin-mdblist

A [Silo](https://siloserver.org) metadata-provider plugin that enriches media
items with **ratings** and a few **gap-fill** fields from the
[MDBList API](https://api.mdblist.com). It is the successor to `silo-plugin-omdb`
and is intended to be the **only** ratings provider Silo runs.

- **Plugin ID:** `silo.mdblist`
- **Capabilities:** two `metadata_provider.v1` capabilities — `imdb` and `tmdb`
- **API:** `https://api.mdblist.com`, authenticated with an `apikey` query parameter
- **Data flow:** pull-based — Silo calls `GetMetadata`; the plugin looks the title
  up on MDBList by an existing provider ID and returns ratings (+ optional gap-fill)

---

## Why MDBList (vs OMDb)

OMDb gave us the IMDb rating and the Rotten Tomatoes **critic** score. MDBList
returns those **plus the Rotten Tomatoes audience score**, from one richer API,
on a 100k/day Plus key. Everything the Silo database can actually store for
ratings now comes from a single source.

MDBList returns far more (Metacritic, Trakt, Letterboxd, Roger Ebert, MyAnimeList,
budget, revenue, streaming availability, awards, parental ratings, …) — but the
Silo schema has **no column** for any of it, so this plugin deliberately ignores
those fields. See *What Silo can store* below.

---

## What Silo can store (the contract)

Silo ingests a plugin's `MetadataItem` in
`internal/metadata/plugin_provider.go`. Two facts drive this plugin's entire
design:

1. **Ratings** are read from a `google.protobuf.Struct` and only **four float
   keys** are recognised:

   | Struct key     | Silo DB column      | Scale | MDBList source |
   |----------------|---------------------|-------|----------------|
   | `imdb`         | `rating_imdb`       | 0–10  | `imdb`         |
   | `tmdb`         | `rating_tmdb`       | 0–10  | *(skipped — TMDB owns it; MDBList reports it 0–100)* |
   | `rt_critic`    | `rating_rt_critic`  | 0–100 | `tomatoes`     |
   | `rt_audience`  | `rating_rt_audience`| 0–100 | `popcorn`      |

   Any other key (metacritic, trakt, …) is silently dropped by Silo.

2. From the generic `metadata` struct, Silo reads **only** `keywords`. MDBList's
   title endpoint provides no keywords, so we emit none — nothing else from a
   generic metadata blob would be persisted anyway.

The remaining `MetadataItem` scalar/array fields *are* persisted (title,
overview, genres, runtime, content_rating, countries, language, dates, images,
people, provider IDs). This plugin populates only the **gap-fill subset** below,
and never title/overview/genres/images/people, which TMDB and TVDB own.

---

## Non-destructive by construction

Silo merges providers with `MergeFillEmpty` (`internal/metadata/merge.go`): the
**first non-empty value wins**, and later providers only fill blanks. Provider
IDs always **accumulate**. Consequences:

- TMDB/TVDB don't supply IMDb or Rotten Tomatoes *ratings*, so MDBList fills
  `rating_imdb`, `rating_rt_critic`, `rating_rt_audience` — pure gain.
- Gap-fill scalars (content rating, language, country, year, release/first-air
  date, movie runtime) only land where TMDB/TVDB left a blank. They can never
  overwrite richer data, regardless of provider order.
- Cross-provider IDs (`tvdb`, `trakt`, `mal`, `mdblist`, …) are backfilled onto
  the item — also pure gain.

Gap-fill can be turned off entirely (ratings-only) with the **Gap-fill** switch
in config.

---

## Two capabilities, one request

The plugin declares two `metadata_provider.v1` capabilities, `imdb` and `tmdb`.
A Silo metadata provider is only consulted for an item that already carries a
provider ID **matching the capability ID**. Declaring both means the plugin
fires for any item with *either* an IMDb or a TMDB ID — i.e. essentially the
whole library.

Because both capabilities can fire for the same title in a single refresh, a
short-lived (5 min) in-process cache keyed by every ID in the response collapses
the two back-to-back lookups into **one** MDBList request. This halves quota
spend during a full library refresh.

`GetMetadata` selects which ID to look up by, in preference order:
`imdb → tmdb → tvdb → trakt → mal`. Silo's item type `movie`/`series` maps to
MDBList's `movie`/`show`.

---

## Rate limiting & graceful degradation

The Plus key allows 100k requests/day, but the account will be **downgraded to a
cheaper tier after the initial ingest**, so the client is built to degrade
cleanly:

- **Header-aware.** Reads `X-RateLimit-Remaining` / `X-RateLimit-Reset` on every
  response; honours `Retry-After` on `429`.
- **Cooldown gate.** When remaining hits 0 or a `429` lands, the client records a
  cooldown until the reset time and **short-circuits further calls without
  touching the network** — a depleted quota costs zero wasted requests.
- **Throttle.** An optional `requests_per_second` setting spaces outbound calls
  so a backfill spreads across the daily budget instead of bursting through it.
- **Retriable vs terminal.**
  - Rate-limit / transient → `GetMetadata` returns gRPC `Unavailable`. Silo logs
    it as a provider error and skips it for this pass; the item's ratings fill on
    a later refresh once the quota resets (cheap to re-run thanks to
    `MergeFillEmpty`).
  - `404` / `400` / `422` (missing or malformed ID) → terminal "not found"
    (`nil`), so the item is not retried forever.
  - `401` / `403` / `5xx` → surfaced as an error so it is visible/retried.

> **Note on queuing:** a metadata provider is *pull-based* — it cannot push
> results back to Silo asynchronously. "Queuing" is therefore achieved by (a) the
> throttle + cooldown avoiding wasted calls, and (b) Silo's metadata
> refresh-debt scheduler (and/or a manual re-run) re-refreshing un-filled items
> after the quota resets.

---

## Configuration

Global config key `mdblist`:

| Field                 | Control   | Default | Purpose |
|-----------------------|-----------|---------|---------|
| `api_key`             | password  | —       | MDBList API key from mdblist.com/preferences/#api (required). |
| `gap_fill`            | switch    | `true`  | Also fill content rating, language, country, year, release/first-air date, and (movie) runtime where another provider left them empty. Off = ratings only. |
| `requests_per_second` | number    | `0`     | Throttle outbound calls. `0` = unlimited. Raise the spacing after downgrading to a cheaper tier. |

---

## Updating only the missing ratings (without a full library refresh)

A full library refresh in `complete` mode re-runs **every** metadata provider
against **every** item — correct, but slow and quota-hungry. If all you want is
to fill ratings that are still blank (e.g. just after installing the plugin, or
after importing new titles), there are two cheaper paths.

### Option A — refresh only the items you choose, through Silo

Silo can refresh a single item or a single library on demand. **Only `complete`
mode runs metadata providers** (so it is the mode that actually calls this
plugin); `quick` mode skips providers.

```sh
# one item
curl -X POST https://<silo>/api/v1/admin/items/<content_id>/refresh-metadata \
     -H 'Content-Type: application/json' -d '{"mode":"complete"}'

# one library
curl -X POST https://<silo>/api/v1/libraries/<library_id>/refresh-metadata \
     -H 'Content-Type: application/json' -d '{"mode":"complete"}'
```

This goes through the plugin normally, so it benefits from the per-title cache
and rate-limit handling above. Great for a handful of items; for tens of
thousands it still issues one MDBList lookup per title.

### Option B — bulk backfill via MDBList's BATCH endpoint (most quota-efficient)

For a large library the cheapest way to fill blanks is to look items up in
**batches** and write only the missing rating columns directly. This is how a
~183k-item library was backfilled in well under one day's quota.

**1. Find what's eligible.** An item is worth a lookup only if it (a) is missing
at least one MDBList-fillable rating and (b) already carries an `imdb` or `tmdb`
ID (without one, the plugin has nothing to look up):

```sql
SELECT mi.content_id, mi.type,
       COALESCE(pi.provider_id,'') AS imdb,
       COALESCE(pt.provider_id,'') AS tmdb
FROM media_items mi
LEFT JOIN media_item_provider_ids pi ON pi.content_id = mi.content_id AND pi.provider = 'imdb'
LEFT JOIN media_item_provider_ids pt ON pt.content_id = mi.content_id AND pt.provider = 'tmdb'
WHERE (mi.rating_imdb IS NULL OR mi.rating_rt_critic IS NULL OR mi.rating_rt_audience IS NULL)
  AND (pi.provider_id IS NOT NULL OR pt.provider_id IS NOT NULL);
```

To target only the genuine coverage gap (items with **no** rating at all, far
fewer calls), change the rating clause to `AND` across all three columns:
`(rating_imdb IS NULL AND rating_rt_critic IS NULL AND rating_rt_audience IS NULL)`.

**2. Batch the API calls.** MDBList's batch endpoint costs **one quota unit per
call regardless of how many IDs you send** — so send ~100 at a time:

```
POST https://api.mdblist.com/{provider}/{type}/?apikey=<KEY>
Content-Type: application/json
User-Agent: <any non-default UA>          # REQUIRED — see gotchas

{"ids": ["tt0120338", "tt0111161", ...]}
```

- `{provider}` = `imdb` or `tmdb`; `{type}` = `movie` or `show` (Silo's `series` → `show`).
- Group eligible items by `(provider, type)` and send ~100 IDs per request.
- Math: ~183k items ÷ ~100 IDs/call ≈ **~1,800 calls** — comfortably inside a
  100k/day key, and fine even on a much smaller tier.

**3. Map results and write only the blanks.** From each result take
`imdb` → `rating_imdb`, `tomatoes` → `rating_rt_critic`,
`popcorn` → `rating_rt_audience`, and write with `COALESCE` so existing values
are never touched:

```sql
UPDATE media_items mi SET
  rating_imdb        = COALESCE(mi.rating_imdb, v.imdb),
  rating_rt_critic   = COALESCE(mi.rating_rt_critic, v.crit),
  rating_rt_audience = COALESCE(mi.rating_rt_audience, v.aud)
FROM (VALUES /* (content_id, imdb, crit, aud), ... */) AS v(content_id, imdb, crit, aud)
WHERE mi.content_id = v.content_id;
```

**Why it's safe and cheap**

- **Non-destructive:** `COALESCE` fills only NULLs — the same effect as the
  plugin's `MergeFillEmpty`; `rating_tmdb` and any existing scores are preserved.
- **Resumable:** the eligibility query re-selects NULLs each run, so if you stop
  (or hit a `429`) just run it again to continue where it left off.
- **Immediate:** Silo's browse/detail views read the `rating_*` columns
  directly, so filled ratings show in the UI right away (no reindex). Note this
  direct write does **not** bump `updated_at`.

**Gotchas**

- **User-Agent is mandatory.** MDBList sits behind Cloudflare, which returns
  **403** to a default/empty UA (e.g. `Python-urllib`). Set any custom
  `User-Agent` (curl's default is fine).
- **Stop on 429.** A `429` means the daily quota is spent; it resets at
  **00:00 UTC**. Stop and resume later rather than hammering it.
- **Some titles stay NULL forever.** RT critic/audience exist only for reviewed
  titles; obscure entries may return an IMDb score but no RT, or MDBList may only
  hold Metacritic/Trakt/Letterboxd — none of which Silo can store (see *What Silo
  can store*). That's missing upstream data, not a bug. In practice ~95% of
  eligible items ended up with ≥1 storable rating; re-running to chase the last
  ~5% yields almost nothing.

> A reference implementation of Option B (eligibility query, batching, retry +
> quota handling, and the `COALESCE` write) is the `mdblist-backfill.py` script.
> Supply your own database access and API key via environment variables — never
> hardcode the key.

---

## Project layout

```
main.go              Runtime + MetadataProvider servers, Configure, GetMetadata,
                     the per-title cache, manifest loading.
mapping.go           MDBList MediaInfo -> Silo MetadataItem mapping, ID selection,
                     item-type and ratings mapping.
mdblist/types.go     API response models + rating accessors.
mdblist/client.go    Rate-limit-aware HTTP client (apikey query auth, cooldown,
                     throttle, 1 MiB body cap, 15s timeout).
manifest.json        Two metadata_provider.v1 capabilities + config schema.
*_test.go            Unit tests (client behaviour, mapping, cache, manifest).
Makefile             build / test / lint / build-all (linux amd64+arm64, darwin arm64).
.github/workflows/   CI (test) and Release (tag-driven multi-arch build + manifest checksum).
```

---

## Build & test

```sh
make test        # go test ./...
make build       # local binary
make build-all   # dist/ binaries for all supported platforms
```

Releases are tag-driven (`v*`): the workflow builds each platform, injects the
binary's SHA-256 into `manifest.json` (`__CHECKSUM__`), and publishes a GitHub
release. CI fetches the (private) SDK via `GOPROXY=direct` + `GOPRIVATE`, and
**rejects** any committed machine-local `replace` of the SDK.

---

## Migrating from silo-plugin-omdb

1. Install and configure `silo-plugin-mdblist` (set the API key).
2. Disable/remove the OMDb plugin.
3. Trigger a metadata refresh. MDBList backfills `rating_imdb`,
   `rating_rt_critic`, and the new `rating_rt_audience` across the library
   (existing values are preserved; only blanks are filled unless you force a
   full overwrite refresh).
