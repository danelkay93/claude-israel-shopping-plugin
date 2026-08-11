---
name: general-search
description: Broader Israeli retail search beyond the tier-1 tech four — Ace, Home Center, Office Depot, Audioline, plus Google IL discovery and niche category-dispatched search. Use for household/office/audio/appliance products, or when tier-1 tech retailers don't carry the item.
---

# General Search

Cross-retailer Israeli search covering tier-2 majors (Ace, Home Center, Office Depot, Audioline), Google IL discovery, and category-dispatched sweeps of the merged store list for niche specialists.

> See `docs/search-strategies.md` for the tier-2 retailer table, Google IL query patterns, category-dispatch vocabulary, Hebrew-term resolution, and store-metadata merge convention.

## When to use

- Product leans household / office / appliance / audio rather than pure consumer tech.
- Tier-1 tech search (`/tech-product-search`) and Zap came back empty.
- Niche specialist categories — tactical, lighting, photography, gaming, smart home.
- Discovery-mode queries where no specific retailer is in mind.

For most cross-retailer price comparison, try `/search-zap` first. Come here when Zap misses or for broader discovery.

## Workflow

1. **Classify the query** — does it fit tier-2 majors, a niche category, or does it need Google discovery?
2. **Resolve Hebrew term** (preflight — see strategies doc).
3. **Load the merged store list** — upstream `stores.json` overlaid with `<plugin-data-dir>/user-stores.json (resolved via $CLAUDE_USER_DATA — see docs/search-strategies.md)` (see strategies doc).
4. **Pick path:**
   - **Tier-2 majors** — search Ace, Home Center, Office Depot, Audioline via Playwright homepage form or Tavily.
   - **Category-dispatch** — filter merged store list by matching `categories[]`; pick up to 8 stores; search each.
   - **Google IL** — patterns from strategies doc (`"brand model" site:.il`, `מחיר <hebrew>`, Google Shopping IL).
5. **Rank low-to-high** by VAT-inclusive ILS price. Cap 5 per retailer / 3 per niche store.
6. **For appliances/electronics, split importer tracks** using `docs/search-strategies.md` § Official-importer vs parallel-import tracks. Compare like-for-like official and parallel-import offers, warranty length/provider, and documented service quality; keep ambiguous wording unclassified.
7. **Flag retailers not in the merged store list** — candidates for `/add-store`.

## Output

```
## {product} — general IL search
Hebrew query: {hebrew}
Path: {tier-2 | category:{cats} | google-il | mixed}
Stores/sources searched: {n}

| # | ₪ (inc. VAT) | Product | Retailer | Link |
|---|---|---|---|---|

### Notes
- Retailers not in merged store list (candidates to add): ...
- Retailers that returned nothing / errored: ...
- Out-of-category misses: ...
- Official-vs-parallel price premium, warranty-month delta, and service-quality evidence for appliances/electronics.
```

## Rules

- Only navigate stores where `is_tech` is `true` or `null`.
- Never quote a price you couldn't extract.
- Don't run >3–4 Google query variants per session (rate limits).
- Never infer official-importer status from retailer reputation or vague `יבואן מורשה` wording.
- If none of these paths fit, suggest `/tech-product-search` or `/search-zap` and stop.

