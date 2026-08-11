# Israeli Shopping — Search Strategies

Agent-facing reference for the substantive know-how behind the plugin's search commands. The commands (`/tech-product-search`, `/general-search`, `/search-zap`, etc.) point here rather than restating the strategy. Read the section relevant to the task at hand.

## Store-metadata merge convention

All commands that consume the Israeli store list must **merge** two sources:

1. **Upstream canonical list** — `https://raw.githubusercontent.com/danielrosehill/Israel-Online-Stores/main/stores.json` (live, 800+ entries). Read this first.
2. **User overlay** — `<plugin-data-dir>/user-stores.json` (optional). The plugin data directory resolves as `$CLAUDE_USER_DATA/israel-shopping/` if `CLAUDE_USER_DATA` is set; otherwise `$XDG_DATA_HOME/claude-plugins/israel-shopping/` if `XDG_DATA_HOME` is set; otherwise `~/.local/share/claude-plugins/israel-shopping/`. See the canonical convention in the `meta-tools:plugin-data-storage` skill. If present, merge/append on top of the upstream list. Dedup by canonical URL / domain — user entries win on conflict.

Do not write user additions back into the plugin install directory — those get clobbered on plugin update. Only `<plugin-data-dir>/user-stores.json` persists across updates.

Schema matches the upstream `stores.json` shape (name, url, categories[], is_tech, description, delivery, eilat_door, zap_profile, etc. — read a live entry to confirm the current shape before writing).

---

## Hebrew-term resolution (mandatory preflight)

Israeli retailer and aggregator search returns nothing useful with English category nouns. Every query must be built as:

```
<Hebrew category noun>  <brand in Latin>  <model in Latin>
```

Examples:
- Anker Prime A2343 charging tower → `עמדת טעינה Anker Prime A2343`
- Logitech MX Master 3S mouse → `עכבר Logitech MX Master 3S`
- LG C4 55" OLED TV → `טלוויזיה LG C4 55`

**Brand and model stay Latin** — retailers index SKUs in Latin. Only the category noun is translated.

### If you don't know the Hebrew noun

1. Check the user's vocabulary — if they used a Hebrew word, use it.
2. Otherwise translate the category noun. Common mappings:
   - mouse → `עכבר`
   - charger → `מטען`
   - cable → `כבל`
   - monitor → `מסך`
   - headphones → `אוזניות`
   - laptop → `מחשב נייד`
   - keyboard → `מקלדת`
   - TV → `טלוויזיה`
   - headlamp → `פנס ראש`
   - rechargeable battery → `סוללות נטענות`
   - charging dock/station → `עמדת טעינה` / `תחנת טעינה` / `מטען שולחני`
3. Have 1–2 Hebrew synonyms ready as fallback.

### Reverse-lookup when the term is unknown

If translation gives weak results, derive the canonical term from a real IL listing:

1. Google `"<brand model>" site:.il` for a known SKU in the class.
2. Open the top 2–3 retail results (skip news/blog domains — ynet, walla, haaretz, calcalist, themarker, geektime).
3. Extract the Hebrew noun from, in order of cleanness:
   - **URL slug** — percent-decode and read the Hebrew noun.
   - **`<title>` tag** — usually starts with the Hebrew product class noun.
   - **Breadcrumb nav** — `.breadcrumb`, `nav[aria-label*="breadcrumb"]`, schema.org breadcrumb markup.
4. Normalise to **singular**. Strip plural `ות`/`ים` unless the category is inherently plural in retail use.
5. If different retailers use different terms, record all variants — common for chargers (`עמדת טעינה` vs `מטען שולחני` vs `תחנת טעינה`).
6. Note exclude terms — e.g. `סוללה ניידת` (power bank) must be excluded when searching for `סוללות נטענות` (rechargeable batteries).

Never trust Google Translate alone for retail jargon. Never invent a Hebrew term — if no IL listing exists for any SKU in the class, ask the user.

---

## Official-importer vs parallel-import tracks

Apply this section to **appliances, electronics, power tools, and other warranty-sensitive durable goods**. Skip it for groceries, clothing, books, inexpensive accessories, and ordinary consumables unless the user asks.

Do not collapse all Israeli offers into one price table. Build three evidence-based buckets:

1. **Official importer — `יבואן רשמי`**: the product page, manufacturer, or named importer explicitly says `יבואן רשמי`, `אחריות יבואן רשמי`, or identifies the brand-recognised Israeli importer. A large retailer name alone is not proof.
2. **Parallel import — `יבוא מקביל`**: the listing explicitly says `יבוא מקביל`, `יבואן מקביל`, or clearly identifies an independent importer/warranty provider while disclaiming official import.
3. **Unverified importer status**: vague phrases such as `יבואן מורשה`, `אחריות יבואן`, `אחריות על ידי החנות`, or a warranty-company name without an explicit official/parallel statement. Never translate `יבואן מורשה` into “official importer.” Keep these offers visible but outside strict official-vs-parallel parity calculations.

For each offer capture:

- all-in price, exact model/variant, bundle, VAT, and delivery;
- importer classification plus the exact supporting wording and source;
- warranty duration in months and named warranty/service provider;
- whether the manufacturer recognises the warranty in Israel;
- service method: in-home service, pickup/return, carry-in lab, or retailer-only handling;
- material coverage differences: parts, labour, battery/motor/compressor/panel, accessories, exclusions, and years with reduced coverage;
- seller rating/recent-review count separately from warranty-provider quality.

Then produce two recommendation tracks plus an unverified appendix:

### Official-import track

Rank only eligible, destination-verified official-import offers. Show the cheapest recommended option and any meaningfully better service package.

### Parallel-import track

Rank only eligible, destination-verified parallel-import offers. Apply the same seller-rating and stock checks as the official track.

### Price/warranty parity

Compare like-for-like exact variants:

- official premium in ILS: `official total - parallel total`;
- official premium percentage: `(official - parallel) / parallel * 100`;
- warranty-duration delta: `official months - parallel months`;
- if the official warranty is longer, optional cost per extra warranty month: `official premium / extra months`;
- qualitative service delta based only on documented terms or credible service-provider evidence.

Do not assume official import is automatically longer or higher quality. If both tracks provide the same duration, say so. If service quality cannot be substantiated, label it `unknown` rather than treating “official” as a quality score. If one strict track has no eligible offer, say that parity cannot yet be calculated and optionally show an excluded-market reference separately.

---

## Tier-1 Israeli tech retailers — Ivory / KSP / Bug / TMS

The four mainstream tech chains. Used by `/tech-product-search`.

| Retailer | Search URL pattern | Backend notes |
|---|---|---|
| Ivory | `https://www.ivory.co.il/catalog.php?act=cat&q=<URL-encoded Hebrew>` | Usually scrapable via Tavily / WebFetch. |
| KSP | `https://ksp.co.il/web/cat/?search=<URL-encoded Hebrew>` | Bot-detected — Playwright required. Two prices (regular + Eilat); always quote the **regular** (higher) price unless the user specifies Eilat pickup. |
| Bug | `https://www.bug.co.il/` (homepage search) | Bot-detected — Playwright required. URL pattern not stable; use homepage search box. |
| TMS | `https://tms.co.il/index.php?route=product/search&search=<URL-encoded Hebrew>` | Scrapable via Tavily / WebFetch. |

**URL-encode the Hebrew query.** Example: `מחשב` → `%D7%9E%D7%97%D7%A9%D7%91`. Don't paste raw Hebrew into the URL.

Treat the URL patterns as last-known-good hints, not contracts. If a documented pattern returns a homepage redirect or 404, fall back to the Playwright homepage-form flow and note it in the output.

### Backend chain (per retailer)

Try in order; pick one and proceed; note the backend used in the output.

1. **Tavily search** (`mcp__jungle-shared__tavily__tavily_search`) with `query: "<hebrew> <brand> <model> site:<domain>"`, `search_depth: "basic"`, `max_results: 5`. Fast. Works for Ivory and TMS.
2. **WebFetch** the retailer's search URL directly if you know the pattern (verify real results, not a homepage redirect).
3. **Playwright fallback** (`mcp__plugin_playwright_playwright__browser_*`) when steps 1–2 return no results, or for KSP/Bug by default:
   - `browser_navigate` to the retailer homepage.
   - `browser_snapshot` to locate the search input.
   - `browser_type` the Hebrew query with `submit: true`.
   - `browser_snapshot` again, extract the top 5 product cards (title, price ILS, link, stock).
   - `browser_close` when done.

If all three fail for a retailer, record "no results / blocked" and continue — don't fail the whole run.

### KSP regular-vs-Eilat price rule

KSP lists two prices: regular and Eilat (tax-exempt). Always quote the **regular (higher)** price unless the user explicitly asked about Eilat pickup. If both appear, show the regular price in the table and note the Eilat price in a comment.

### Fallback when a retailer returns nothing

1. Retry with a Hebrew synonym from the term-resolution list.
2. Retry with **brand + model only** (drop the Hebrew noun) — some retailer search engines reject the noun but accept the SKU.
3. Google `<brand model> site:<retailer-domain>` — Google's index of the retailer often beats the retailer's own search.

---

## Tier-2 major retailers — Ace / Home Center / Office Depot / Audioline

For products that lean household/office/audio rather than pure consumer tech. Used by `/general-search`.

| Retailer | Best for |
|---|---|
| Ace | Hardware, tools, garden, some electronics |
| Home Center | Home improvement, appliances, fixtures |
| Office Depot | Office supplies + tech accessories |
| Audioline | Audio equipment specialist |

Same backend chain as the tier-1 group (Tavily → WebFetch → Playwright homepage form). Same Hebrew preflight. Same fallback chain. Out-of-category misses are normal (e.g. searching Audioline for a vacuum) and not worth flagging unless every retailer missed.

---

## Zap — canonical price-comparison aggregator

Zap.co.il indexes prices across tier-1, tier-2, and many niche IL retailers. For most cross-retailer price-comparison queries, Zap is the fastest first pass — one query returns the ranked vendor table.

### Backend

Playwright required — Zap has bot detection, Tavily snapshots are stale and miss the price table.

### URL patterns

- **Keyword search:** `https://www.zap.co.il/search.aspx?keyword=<encoded Hebrew query>`
- **Model page (vendor comparison table):** `https://www.zap.co.il/model.aspx?modelid=<id>` — this is what you actually want; it has all vendors side-by-side. `modelid` is visible in the search results.
- **Category browse:** `https://www.zap.co.il/models.aspx?sog=<category>` — fallback when keyword search is unhelpful.
- **Store profile (for retailer ratings):** `https://www.zap.co.il/shop.aspx?sog=shop-<id>` — used by `/store-zap-lookup`.

### Workflow

1. Resolve Hebrew term (preflight).
2. Navigate to keyword search with the Hebrew query.
3. Find the right model — Zap groups by SKU. If unsure between multiple matches, list the top 3 and confirm with user.
4. Open the model page. Extract the vendor table: vendor name, ILS price (verify VAT-inclusive), delivery info, link.
5. Check for Eilat / ex-VAT badges on unusually low prices — use the regular price.
6. Rank low-to-high. Cap at 10 vendors unless asked.

### Caveats

- Zap's cached price can lag reality by 24+ hours — visit the vendor page before quoting.
- Zap sometimes lists ex-VAT prices without a clear badge — verify on the vendor page.
- Show Zap's own "Zap price" (recommendation) alongside the cheapest vendor — it reflects Zap's deal arrangements.

### When Zap misses

If keyword search returns no model page or the vendor table is empty:

1. Google IL discovery (below) — catches specialists Zap doesn't index.
2. Tier-1 tech retailer sweep (`/tech-product-search`).
3. Category-dispatched search against the merged store list.

Tell the user why Zap didn't work (new SKU, niche product, variant not indexed).

---

## Google IL discovery

Google often beats per-retailer search for discovery, for niche products, and as a Hebrew-term sanity check. Used within `/general-search` and whenever retailer-specific searches return nothing.

### Backend

Playwright MCP — Google Shopping blocks most scrapers; Tavily is unreliable here. Navigate `https://www.google.com/search` and submit the query.

Locale params:
- `gl=il` — geolocation Israel (prices in ILS, IL retailers ranked first)
- `hl=he` — Hebrew interface (Hebrew-only pages surface)
- `tbm=shop` — Google Shopping vertical

### Query patterns

Try in this order; don't run more than 3–4 per session (Google rate-limits).

1. **`"<brand model>" site:.il`** — best for known-SKU lookups, Hebrew-term reverse-lookup, finding any IL retailer carrying the product. Quote the model number to stop Google relaxing it.
2. **`"<brand model>" site:co.il`** — narrower, commercial-only (skips `.org.il`, `.ac.il`, `gov.il`). Use when (1) is too noisy.
3. **`מחיר <hebrew term> <brand>`** — the word `מחיר` ("price") is a strong commercial-intent signal. Good when you have a category but not a specific model.
4. **`קנה <hebrew term>`** — similar but noisier; fallback.
5. **`<hebrew term> בארץ` / `ישראל`** — when Google returns too many international results and `site:.il` over-filters.
6. **Google Shopping IL** — `https://www.google.com/search?tbm=shop&gl=il&hl=he&q=<hebrew query>`. Patchy coverage but fast price-ranked cross-check.

### Extract + filter

For each result: domain, page title, visible price (if in snippet). Drop:
- News sites (ynet, walla, calcalist, haaretz, themarker, geektime).
- Blogs/reviews unless the user asked for them.
- Dead or parked domains.
- Duplicate domains — keep the best-ranked per domain.

Group by domain, surface the top 3–5 retailers. Flag any retailer not already in the merged store list — candidate for user to add via `/add-store`.

---

## Category-dispatched search

When the query lies outside the mainstream tier-1 and tier-2 chains — e.g. tactical gear, specialist audio, photography accessories — filter the merged store list by category and search the matching stores directly.

### Vocabulary

`computers`, `mobile`, `peripherals`, `appliances`, `photography`, `audio`, `tv_av`, `gaming`, `office`, `lighting`, `tactical`, `power_battery`, `smart_home`, `security`, `tools`, `spare_parts`, `accessories`.

### Workflow

1. Map the user's request to one or more categories. Examples: "headlamps" → `lighting` + `tactical`; "DSLR bag" → `photography` + `accessories`; "UPS" → `power_battery`.
2. Filter the merged store list: `is_tech != false` AND any category in `store.categories[]` matches.
3. Pick up to 8 stores. Prioritise `is_tech: true` over `null`, then stores whose description mentions the specific product type, then stores with multiple matching categories.
4. Resolve Hebrew terms.
5. Search each store. Try in order:
   - Playwright: homepage, locate search form (`input[type=search]`, `input[name=q]`, `input[name=s]`, `#search`, `#qSearch`), submit Hebrew query.
   - URL-pattern probes: `/?s={q}` (WordPress), `/search?q={q}`, `/catalogsearch/result/?q={q}` (Magento).
   - Tavily: `site:{domain} <hebrew query>`.
6. Try alternate Hebrew variants if primary returns nothing.
7. Rank low-to-high. Cap at 3 results per store.

Only navigate to stores where `is_tech` is `true` or `null`. Never `false`.

---

## Sourcing waterfall

When a candidate has survived shortlisting and you need landed cost + availability, apply the waterfall in order:

1. **IL domestic tier-1 tech** — Ivory/KSP/Bug/TMS via `/tech-product-search`. Capture store, ILS price (VAT-incl; add 18% if ex-VAT), stock, warranty. KSP: use regular (non-Eilat) price.
2. **IL domestic tier-2 broad** — Ace/Home Center/Office Depot/Audioline via `/general-search`.
3. **Niche IL specialists** — category-dispatched search (above).
4. **Amazon** — only if domestic missing or exceeds markup threshold, and spec hasn't disabled Amazon. Verify "ships to Israel" on the product page. Add shipping + 18% VAT + possible customs duty (imports >$75 USD de minimis). Compute landed total in ILS.
5. **AliExpress** — only if spec/preferences allow. Branded items: seller rating ≥95%. Capture USD price + shipping, rating, lead time (2–6 weeks to IL), landed ILS.
6. **Manufacturer direct / B&H / specialist international** — last resort.

Pick the best channel per candidate based on landed cost, lead time vs spec tolerance, warranty, channel viability.

---

## VAT and markup verdict scale

Israeli VAT is currently 18% (as of 2025+). All retail pricing should be quoted VAT-inclusive. If a listing is ex-VAT (common for Eilat, B2B quotes, some Zap rows), normalise by adding 18% before comparing.

### RRP-vs-IL verdict scale

Used by `/rrp-check`:
- ≤5% → GOOD DEAL
- 5–15% → FAIR (typical IL import markup)
- 15–30% → ABOVE AVERAGE
- 30–40% → MILD MARKUP
- \>40% → HEAVY MARKUP (consider importing)

An IL markup of 30–40% is often still cheaper than importing once shipping + VAT + customs are counted — factor landed cost before recommending import.

---

## Playwright usage notes

- Always `browser_close` when done with a retailer to free the session.
- Use `browser_snapshot` over `browser_evaluate` for extracting product cards — more robust to DOM changes.
- When a homepage search form isn't obviously matchable, fall back to trying common URL patterns (see category section) before giving up.
- Never invent prices — if you can't extract one, say so.

