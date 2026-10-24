---
name: online-groceries
description: 'Build an online grocery basket on French retail sites (Carrefour, Courses U / Super U, with Leclerc, Auchan, Picard for comparison) through a logged-in browser tab: who delivers, store promotions, cart API, free-delivery threshold, recipe page. Use when the user says "do my groceries", "fais mes courses", "refais le panier", "promos", "promo code", "cheapest", "who delivers", or mentions Carrefour, Super U, drive.'
---

# Online groceries

Method refined over several rejected baskets before one was accepted. The user's tastes, refused products and kitchen equipment live in `preferences.md` next to this file (local, never committed, see `preferences.example.md`): read it before picking a single product.

Requires a browser automation tool that drives the user's own logged-in browser (e.g. Claude in Chrome). Without one, stop and say so: the sites need the user's session.

## First run (no `preferences.md` yet)

Do not guess a profile. Interview the user in one round of questions (AskUserQuestion, at most 4 questions per call, two calls if needed), then write `preferences.md` from `preferences.example.md` and show it back:

1. Postcode and preferred retailers (or "find who delivers").
2. Household size and target budget per order (or "just reach free delivery").
3. Diet and allergies (vegetarian, halal, gluten-free, lactose, nut allergy...), and cuisines they dislike.
4. Kitchen equipment, and staples they already have at home.

Later sessions: re-read the file, never re-ask what it already answers. Update it every time the user removes a product or states a taste.

## Hard rules

- Never pay, never type a password, never create an account (not even a "test account"): the user logs in and pays. Hand payment to the user as an explicit task, one open at a time.
- Delivery address: the one saved on the account profile, never written to a file.
- Two baskets ready at two retailers: say clearly to pay for only one.
- A tab on the checkout page belongs to the user: work in a new tab, never navigate it.
- Bot protection (DataDome "Access temporarily restricted", CAPTCHA): stop, fall back to a web search, do not bypass it. On a 429: slow down, do not force.
- Allergies and diet in `preferences.md` are absolute. When a product's ingredients are not certain (ready meals, sauces, "may contain"), open its product page and check; if still unclear, leave it out and say so. Never claim a basket is allergen-free: tell the user to check labels.
- Prices, fees, thresholds and endpoints below were observed in 2026 and change: read the live values on the site (cart page, delivery info) before relying on them.

## Workflow

1. **Preferences** (`preferences.md`, or the first-run interview): tastes, diet, products removed earlier (never suggest them again), equipment, budget.
2. **Who delivers** (first run, or if asked): test eligibility on each site with the user's postcode, record the result in `preferences.md` (retailer, delivery or pickup only, minimum, fees, free-delivery threshold). Results vary a lot by area: some retailers only offer pickup (drive), some sit behind bot protection, some have a minimum basket. The store that prepares the order is set by the site from the address: note which one, its promotions are store-specific.
3. **Start from the promotions, not from products.** Fetch the store's full promotion list, then every special section (see per site). Searching a product and then checking whether it is on sale was rejected: it misses most of the deals.
4. **Cover every category**: all food aisles (breakfast, sweets, bread, drinks, beer included), hygiene, household cleaning. Skip baby, pets, home, appliances, tech, clothing unless asked.
5. **Quantities that trigger offers**: "2nd at -X%" → take 2, "3 for 2" → 3.
   **Maximize loyalty cashback**: for an equivalent product at a close price, take the "X% in € on the loyalty card" version; compare price minus cashback. Also report dated operations (e.g. a butcher voucher on a given day) and the delivery slot that triggers them.
6. **Cleaning products in the entry-level range**: branded promos usually stay more expensive (bin bags on promo 3.95 € vs 0.58 €).
   **Cold cuts**: entry-level ranges are often the weakest point of a basket; prefer a brand or a higher range (Label Rouge, IGP, regional), even in smaller quantity, unless `preferences.md` says otherwise.
7. **Variety**: one protein + one starch on repeat is the most common rejection. Mix cooking styles (slow-cooked, grilled, marinated, oven or air fryer), several proteins (for a vegetarian: legumes, eggs, tofu, cheese), fish if eaten, vegetables, a regional speciality, breakfast items. Size quantities to the household. Use the user's equipment in the recipes, and respect its limits.
8. **Aim for the target**: the user's budget if set, otherwise the free-delivery threshold **net of discounts** with a small margin. Recount after every change.
9. **Re-read the real cart** (names + quantities) after every batch of additions: keyword search adds false friends (see Error log).
10. **Recipes**: one page (filter by appliance, temperature and time on each card, "what you need at home"), republished with every new basket.
11. When the user removes products: re-read the cart, add the removals to `preferences.md` (never suggest again), refill to the threshold with what they kept. If they say "I don't like it" without detail: ask what is wrong before rebuilding.

## Carrefour (carrefour.fr)

All requests from a logged-in carrefour.fr tab, as same-origin `fetch` with `{'Accept':'application/json','x-requested-with':'XMLHttpRequest'}`.

- Search: `GET /s?q=<words>` → `data[].attributes` (title, ean, format, `offers[ean][offerServiceId].attributes.price.{price,perUnit,perUnitLabel}`, `availability.purchasable`, `promotion`).
- Store promotions: `GET /promotions?page=N` (30 per page, ~120 pages). Drop `promotion.isLoyalty` (cashback) if you want immediate discounts only.
- Cart: `GET /api/cart` → `cart.items[].products[]` (`counter`, `product.attributes.ean`, `offerServiceId`), `totalProductsPrice`, `totalDiscount`, `totalFees`, `totalAmount`.
- Add / set a quantity: `PATCH /api/cart` body `{trackingRequest:[{productId:ean,quantity:n,position:1,listName:'search',source:'search'}], items:[{basketServiceId:offerServiceId, counter:n, ean, subBasketType:'drive_clcv'}]}`. `counter` is **absolute**.
- `counter: 0` **does not remove** (200 but the product stays). To remove a product: the "Supprimer" or "−" button on `/cart/driveclcv`. To empty: "Vider mon panier" button (site modal) or `DELETE /api/cart`.
- Delivery fees seen in 2026 for one store: 7.90 € under 100 €, 4.90 € from 100 to 150 €, free from 150 € **net of discounts**, 60 € minimum. They differ by store: read `totalFees` from `GET /api/cart` at different totals.
- Code `BINGO20` (20 € off from 80 €): first online order only.

## Super U / Courses U (coursesu.com)

The store is chosen on the site from the postcode (URL `/drive-<store>`, e.g. `/drive-superu-<city>`): note it in `preferences.md`. Home delivery, where offered, goes through Shopopop (free from 150 € in 2026; check).

- Sections to scan (`/c/<slug>?start=N&sz=36`, ~19 extra carousel tiles per page, dedupe by `data-pid`): `promotions`, `prix-coutant`, `prix-mini`, `produits-locaux-et-regionaux`, `20-jours-plus`, `produits-bleu-blanc-coeur`, `nouveautes`, `bio`, plus the **dated operations** listed in the menu (e.g. `en-retrait-le-15-octobre--34-en-bon-dachat-sur-la-boucherie-volaille`: 34% of all fresh meat back as a voucher if delivered that day → tell the user so they pick that slot). "Idées repas" are recipe pages, not products.
- **Sequential requests with ~800 ms pause**: in parallel the site returns 429.
- Product tile: `.product-tile`, `data-pid`, price "1,67 €", unit price "4,18 €/kg", badges "X% EN € CARTE U" (cashback), "-50% SUR LE 2ÈME", "PRIX MINI".
- Search: `GET /recherche?q=<words>`.
- Add: `POST /on/demandware.store/Sites-DigitalU-Site/fr_FR/Cart-AddProduct?format=ajax` (form-urlencoded `pid`, `Quantity`, `quantity`, `csrf_token` read from `input[name=csrf_token]`). `?format=ajax` is required since October 2026: without it the reply is a 200 HTML page and nothing is added; a good reply starts with `<script> var dataLayer = {"cart":...`. Adding a product already in the cart increments it.
- Remove: on `/panier`, button `aria-label="Retirer une quantité de <name>"`. The DOM re-renders after each click: find the button again before each removal. "Supprimer le produit" by element ref does nothing; a ref click on "−" works only sometimes. Reliable method: scroll to it, wait 2 s (the page shifts by ~40 px once hydrated, the "Ce produit est idéal avec" link appears), screenshot, click the "−" by coordinates. Verify by reloading `/panier`, not from the page state (a line at 0 keeps its button until reload).
- Cart: `/panier` (estimated total, "Plus que X € pour la livraison offerte", "Je cagnotte X € Carte U"). Empty: "Vider le panier" + "confirmer".
- Modal "Tic tac.. Votre choix de créneau arrive à terme": click "Renouveler mon créneau" (keeps the user's slot).
- **Personal loyalty offers**: `/espace-client?ec-page=fidelite&ec-section=offres` (logged in). Activate them all on every order (free, only applies if bought): click each "Activer l'offre" button in JS with ~900 ms pause, then check "Activée(s) : N". Mostly brands: only add the product if it is really useful and price minus offer stays good.
- Promo codes: read `/conditions-generales-offres.html`; codes there are often limited to one specific store.

## Comparing two retailers

Compare price per kg/litre on the quantities of the reference basket, with strict matching (include AND exclude: `/huile.*colza/` without `/mayonnaise|vinaigrette/`). Drop lines whose equivalent is wrong instead of counting them, and say how many lines were compared out of the total. Entry-level ranges (Eco, Prix mini) explain a good part of the gaps: say so. One run in 2026: Leclerc ~20% cheaper but pickup only; Super U ~5% cheaper than Carrefour, offset by Carrefour's promotions.

## Error log

- Searched products then checked for a promo → most deals missed. Start from the promotion list.
- Basket judged bland and unoriginal, rebuilt 4 times. Ask what is wrong before rebuilding; diversify from the first draft (slow-cooked, marinated, fish, regional, breakfast).
- Carrefour `counter: 0` left products in place (36 instead of 28). Remove through the UI, then recount.
- The 150 € threshold is computed after discounts: at 150.53 € gross, fees stayed at 4.90 €.
- Keyword search false friends: "huile d'olive" → tabbouleh with olive oil, "fleur de sel" → chocolate with fleur de sel, "papier toilette" → wet toilet paper, "lessive" → black-clothes detergent, "colza" → mayonnaise, "cordon bleu surgelé" → fresh cordon bleu, "parmesan" → hachis parmentier. Anchor regexes on the start of the name (`^U Lessive liquide.*Fraîcheur`), list exclusions, re-read added names.
- "Saucisse de Toulouse purée MARIE" is a ready meal, not raw meat. For meat to cook, require the butcher aisle or exclude `/pur.e|cuisin|plat|lentille/`.
- Navigating the tab wipes data kept in `window` (promotion scan lost). Scan and add within the same page load, or keep everything in `localStorage`.
- Browser automation tools block output containing long digit runs or cookie-like text (`[BLOCKED: Cookie/query string data]`). Never print EANs, sanitize outputs (`replace(/[^\w\s%,.]/g,' ')`), keep identifiers in `window`.
- "Le panier a expiré" = U session expired (login page) and slot lost, not an emptied cart. The cart remained, with a "Produits indisponibles" section (fresh products "Bientôt disponible"): remove them ("Tout supprimer") and replace them instead of rebuilding. After logging back in, the `csrf_token` of a tab opened earlier is stale: add from a tab reloaded after login, with the scan kept in `localStorage`.
- Always verify an addition by re-reading `/panier` (item count), never on a 200 status alone.
- A JS execution longer than 45 s freezes the tool: run long loops in the background (`(async()=>{...})()` + a `window.__done` flag) and read the state afterwards.
