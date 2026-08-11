---
name: tech-product-search
description: Search the four tier-1 Israeli tech retailers — Ivory, KSP, Bug, TMS — for a consumer-tech product (laptop, phone, peripheral, accessory, charger, cable, monitor, TV, headphones, etc.) and return prices ranked low-to-high. Handles Hebrew-term resolution and falls back to Playwright when Tavily/WebFetch is blocked.
---

# Tech Product Search — Ivory / KSP / Bug / TMS

Finds a consumer-tech product across the four tier-1 Israeli tech chains and returns a ranked price table.

> See `docs/search-strategies.md` for the full backend chain, Hebrew-term resolution, URL patterns, KSP regular-vs-Eilat rule, and store-metadata merge convention.

## When to use

- User asks to price, find, or compare consumer tech across the big Israeli chains.
- User names any of Ivory, KSP, Bug, TMS specifically.
- For cross-retailer comparison including tier-2 and niche stores, prefer `/search-zap` first; use this when Zap misses or direct retailer pricing is wanted.

## Workflow

1. **Resolve the Hebrew category noun** (see `docs/search-strategies.md` § Hebrew-term resolution). State it in the first user-facing line.
2. **For each of the four retailers**, run the backend chain (Tavily → WebFetch → Playwright) in parallel where possible. Cap 5 results per retailer.
3. **Retry** with a Hebrew synonym or brand+model only if a retailer returns zero.
4. **KSP price rule:** quote the regular (non-Eilat) price unless the user asked about Eilat.
5. **Rank low-to-high** by VAT-inclusive ILS price.
6. **For warranty-sensitive durable goods, split importer tracks** using `docs/search-strategies.md` § Official-importer vs parallel-import tracks. This includes electronics, appliances, power tools, and similar products routed through this command. Show official and parallel-import options separately, keep ambiguous wording unclassified, and calculate price/warranty parity.

## Output

```
## <product> — Israeli tech stores
Hebrew query: <hebrew noun> <brand> <model>
Backends used: Ivory=<tavily|playwright|…>, KSP=<…>, Bug=<…>, TMS=<…>

| # | ₪ (inc. VAT) | Product | Retailer | Link |
|---|---|---|---|---|
| 1 | … | … | … | … |

### Notes
- Any retailer that returned nothing and what fallbacks were tried.
- Price outliers / stock warnings.
- KSP: regular price quoted (Eilat price if seen: ₪X).
- Official-vs-parallel premium, warranty-month delta, named service providers, and any documented service-quality difference (warranty-sensitive durable goods only).
```

## Rules

- Never invent a price. If you can't extract it, say so.
- Quote ILS inc-VAT — never USD on these retailers.
- Never silently substitute a similar-but-different model. If the exact SKU isn't carried, name the closest match and flag it.
- Never infer `יבואן רשמי` from the retailer name or from `יבואן מורשה`; require explicit evidence and keep ambiguous offers unclassified.
- For retailers outside this set, point the user to `/general-search` or `/search-zap`.

