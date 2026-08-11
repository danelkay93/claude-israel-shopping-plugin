---
name: search-zap
description: Search zap.co.il — the canonical Israeli price-comparison aggregator. Zap indexes prices across all tier-1 and tier-2 IL retailers plus many niche specialists, so one Zap search often beats querying each retailer individually. Use as the default first-pass for any cross-retailer price comparison in Israel. Fall back to per-retailer skills only for SKUs Zap doesn't index.
---

# Search Zap

Zap is the canonical Israeli price-comparison aggregator. One Zap search returns a ranked table of all vendors carrying the product, with prices, so it's materially more efficient than hitting each retailer's search endpoint independently.

**Make this the default first-pass** for any cross-retailer price comparison. Only fall through to `/tech-product-search` or `/general-search` when Zap misses (new SKU, niche specialist product, or Zap search returns nothing).

> See `docs/search-strategies.md` § Zap for full URL patterns and workflow, § Hebrew-term resolution for the preflight, and § store-metadata merge convention for the merged-list rule used when flagging vendors.

## Backend

Use a live browser session (Codex in-app Browser / Playwright). Zap has bot detection; indexed web snapshots are often stale and usually omit the seller table. Web search is discovery-only, never the authoritative source for Zap prices.

## Preflight — Hebrew term resolution

Zap search is dramatically better with Hebrew queries. Build: `<hebrew term> <brand> <model>` — brand + model stay Latin.

Example: `עמדת טעינה Anker Prime A2343`, not `"Anker Prime A2343 desktop charging station"`.

Full resolution procedure (including reverse-lookup from listings when the term is unknown) is in `docs/search-strategies.md` § Hebrew-term resolution.

## URL patterns

### Keyword search (start here)

```
https://www.zap.co.il/search.aspx?keyword=<encoded query>
```

Returns a list of matching products and/or model pages.

### Model page (comparison table)

Once you've identified the model, Zap has a dedicated page with the full vendor price table:

```
https://www.zap.co.il/model.aspx?modelid=<id>
```

The `modelid` is visible in the search results URL. This is the page you actually want — it has all vendors side-by-side.

### Category browse (fallback)

If keyword search is unhelpful, browse by category:

```
https://www.zap.co.il/models.aspx?sog=<category>
```

Less precise but useful when you don't know the exact product name.

## Workflow

1. **Resolve Hebrew term** (preflight above).
2. **Navigate to Zap keyword search** with the Hebrew query.
3. **Find the right model.** Zap groups by SKU — pick the result that matches the specific model the user asked about. If unsure, list the top 3 model pages and confirm with user.
4. **Open the model page** (`model.aspx?modelid=...`). This has the price-comparison table.
5. **Extract every vendor row, not only the rows initially visible.** On the current model page, regular offers are represented by `.compare-item-row.product-item`; useful structured fields include `data-site-name`, `data-product-price`, `data-delivery-price`, `data-delivery-time`, ratings, reviews, warranty metadata, and the `/fs.aspx?...` destination link. Each row has:
   - Vendor name
   - Price (ILS, usually VAT-inclusive — verify)
   - Delivery info
   - Link through to the vendor listing
6. **Detect non-neutral placement before ranking.** Never infer rank from vertical position. Check row/card text, accessibility labels, badges, classes, and nearby containers for Hebrew and English promotion variants, including `zap choice`, `ממומן`, `מקודם`, `מודעה`, `קידום`, `תוכן שיווקי`, `בשיתוף`, `Sponsored`, and `Promoted`.
   - Treat `zap choice` as a non-neutral placement/recommendation badge. It is not proof that an offer is bad or paid, but it is not an organic cheapest-price signal and earns no trust or ranking bonus. A screenshot may show a `zap choice` offer above cheaper ordinary offers.
   - Flag Zap-controlled sales separately: `Zap+`, `זאפ סטור`, `רכישה בזאפ`, `shop.zap.co.il`, `.compare-item-row-mp`, and `BuyBox[data-buybox-type="MarketPlace"]`. Exclude these house marketplace/store offers from purchase recommendations by default; if useful, show them in a separate disclosed section.
7. **Separate sale types.** Keep regular offers apart from Eilat/ex-VAT, refurbished/clearance/display, and store-pickup-only groups. Apply rules from `data/shopping-rules.md`; do not mix their prices into the regular delivered-price ranking.
8. **Compute and sort by all-in price.** Parse the numeric product price and delivery cost, add them, then sort ascending yourself. Sponsored/promoted status is a disclosure column, never a ranking advantage. Leave offers with unknown or conditional delivery unranked until resolved, or place them after the entire known-total ranking; never interleave them based on product price alone.
9. **Apply the canonical seller-quality gate** from `docs/search-strategies.md` § Zap. Keep excluded and low-confidence rows in a clearly labelled appendix when their price is informative, and record both the average rating and recent-review count.
10. **For warranty-sensitive durable goods, build importer tracks.** This includes appliances, electronics, power tools, and similar products where warranty coverage materially affects value. Apply `docs/search-strategies.md` § Official-importer vs parallel-import tracks. Keep `יבואן רשמי`, `יבוא מקביל`, and unverified importer wording in separate buckets; do not infer official status from `יבואן מורשה`. Compare the cheapest eligible verified offer in each strict track, including price premium, warranty-month delta, named service provider, and documented service-quality differences.
11. **Resolve and verify shortlisted redirects.** Navigate through each shortlisted Zap `/fs.aspx?...` link and record the final retailer URL. On the retailer page verify all of the following:
   - exact model/variant and accessory bundle
   - current price and delivery condition
   - in-stock or active purchase control
   - warranty duration/provider and importer status

   Classify each offer as `verified`, `mismatch/broken redirect`, `unavailable`, or `uncertain`. Never recommend a row whose redirect lands on a category, a different model, or a page with no current price/purchase path.
12. **Note vendors not in `data/israeli-stores.json`** — Zap often surfaces small specialists worth adding.

## Output

```
## {product} — Zap price comparison
Hebrew query: {hebrew}
Zap model page: {url}
Vendors found: {count}

| # | All-in price (₪, inc. VAT) | Vendor | Rating / recent reviews | Placement | Verification | Delivery | Link |
|---|-------------------------------|--------|-------------------------|-----------|--------------|----------|------|
| 1 | ...                           | ...    | 4.6 / 42                | Organic / zap choice / Zap house marketplace | Verified / broken / unavailable / uncertain | ... | ... |

### Notes
- Any VAT adjustments applied
- Promotion labels observed (quote the exact label); if none were visible, say so rather than assuming the rows are organic
- Seller-quality threshold used and sellers excluded for low/insufficient ratings
- Offers excluded because their Zap redirect was broken, mismatched, or unavailable
- Vendors not in our curated store list: ...
- Delivery cost differences worth flagging

### Importer-track parity (warranty-sensitive durable goods only)

| Track | Best eligible offer | All-in ₪ | Warranty | Provider / service | Evidence |
|---|---|---:|---|---|---|
| Official importer | ... | ... | ... months | ... | exact `יבואן רשמי` wording/source |
| Parallel import | ... | ... | ... months | ... | exact `יבוא מקביל` wording/source |

- Official-import premium: ₪... (...%)
- Warranty-duration delta: ... months
- Documented service-quality delta: ... / unknown
- Unverified importer-status offers: ...
```

## When Zap misses

If the keyword search returns no model page, or the vendor table is empty/very short, fall through in this order:

1. **`/general-search`** — Google IL discovery + category-dispatched search catches specialists Zap doesn't index.
2. **`/tech-product-search`** — direct queries to KSP/Ivory/Bug/TMS.

Tell the user why Zap didn't work (new product, niche, SKU variant not indexed) so they know whether the fallback results are representative.

## Rules

- Never quote Zap prices without visiting the vendor page — Zap's cached price can lag reality by 24+ hours.
- Never use Zap's page order as the comparison order. Extract all rows and independently sort the VAT-inclusive all-in totals.
- Always disclose `zap choice` and other sponsored/promoted labels. Give them no ranking or trust bonus; rank on verified total price after the seller-quality gate.
- Exclude Zap's own store/marketplace offers from recommendations by default and disclose the conflict separately.
- Apply the seller-rating thresholds defined in the canonical Zap strategy rather than maintaining separate values here.
- For warranty-sensitive durable goods, always split eligible offers into official-import and parallel-import tracks, with ambiguous importer wording kept unclassified. Compare warranty duration and documented service quality; never assume official import is better without evidence.
- Always verify VAT treatment on the vendor page — Zap sometimes lists ex-VAT prices without a clear badge.
- If Zap has its own store/marketplace price, disclose it separately and exclude it from recommendations by default — it reflects Zap's own deal arrangements.
- Cap output at 10 vendors unless the user explicitly asks for more.

