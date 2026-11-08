# NEXTANT: Implementation Plan for an Agentic Resale System

Nov 5, 2026

## Name and thesis

**Nextant**: "next" + "extant" (still in existence). Every object is still here, waiting for its next owner. Alternates: *Relay*, *Trove*, *Reprise*.

Nextant turns one phone photo into a priced, timed, measured, labeled listing on the right marketplace, run by ten cooperating agents, two of them optional. The hard part is not generating text; it is making every agent's claim *checkable* and every physical item *unmixable*.

Five design principles that most resale tools get wrong:

1. **Price against sold comps, never asking prices.** Active listings are the prices that *failed* to clear. Asking-price data is survivorship-biased upward, often by 20-40% in slow categories.
2. **Price and time are one decision.** A price is a point on a curve of sale probability over days. Nextant outputs "$48 sells in ~9 days, $62 in ~40" and lets the seller pick, instead of one magic number.
3. **Never fabricate the item's photo.** Buyers and platforms treat an AI-generated image of a used item as misrepresentation. Nextant *cleans* real photos (cutout, white balance, shadow) and shows catalog stock images only as labeled extras.
4. **The physical item is the primary key.** The SKU is bound to the object the moment it is photographed, and every later step (pick, pack, label) is verified by scan, not memory.
5. **Agents publish confidence, the orchestrator spends it.** Each agent returns a value *and* a calibrated confidence. Low confidence routes to a human check or a cheaper re-shoot request, not a guess.

## Architecture

Ten agents share one item record keyed by the physical tag.

```
Seller photo or video
        |
        v
+-------------------+     +-------------------------+     +----------------------------+
| 1. Understand     | --> | 2. Decide               | --> | 3. Act                     | --> Sold, packed, shipped
|  Lens             |     |  Pulse    sell-through  |     |  Muse       buyer copy     |
|  Caliper          |     |  Almanac  when to list  |     | [Keymaster] auto-listing   |
|  Studio           |     |  Ledger   price vs days |     | [Tether]    future phase   |
|                   |     |  Venue    by season     |     |  [ ] = optional            |
+-------------------+     +-------------------------+     +----------------------------+
        ^                            ^                                 ^
        |                            |                                 |
        v                            v                                 v
+------------------------------------------------------------------------------------------+
|  Orchestrator and shared item record                                                       |
|  Tag code is the primary key; every agent writes a value plus confidence                  |
+------------------------------------------------------------------------------------------+
```

Agents never call each other directly: each reads and writes the shared item record, and the orchestrator runs whichever agent's inputs are ready, so Pulse, Almanac and Caliper can run in parallel once Lens finishes.

## Agent 1: Lens (study the image)

Lens produces a structured identity: brand, model, variant, era, material, condition grade and a canonical ID (GTIN, eBay ePID, or MPN) when one exists.

- **Coarse-to-fine, not one shot.** A vision-language model proposes top-5 candidates; an embedding search over a catalog of reference images re-ranks them; only then does the VLM verify the winner against specific tells.
- **Read the object's own text first.** Labels, tags, serial plates, date codes, hallmarks and barcodes are the highest-precision signal. Run OCR and barcode decoding before any visual guessing; a GTIN collapses identification to a lookup.
- **Active re-shoot requests.** When candidates disagree on a discriminating feature ("2019 vs 2021 model differs only in the port layout"), Lens asks for one specific extra photo. That single targeted shot beats ten more generic ones.
- **Condition is a separate head.** Grade wear on a fixed scale mapped to each platform's vocabulary (eBay's condition IDs, Poshmark's NWT/NWOT). Flag defects as bounding boxes so Agent 7 can disclose them honestly.
- **Authenticity risk flag.** For counterfeit-prone categories (sneakers, handbags, watches), output a risk score and route to the platform's authentication program rather than asserting authenticity yourself.

Output: `identity {candidates[], chosen, gtin?, epid?, condition, defects[], confidence}`.

## Agent 2: Pulse (find the demand)

Pulse answers one question: how many buyers clear this item per week, and how many competing listings are waiting for them.

- **The metric is sell-through, not search volume.** Sell-through = sold in window ÷ (sold + still active). Above ~50% in 90 days is a liquid item; under ~15% is a slow item that needs a lower price or a niche channel.
- **Mind the data access trap.** eBay's sold-history endpoint, the [Marketplace Insights API](https://developer.ebay.com/api-docs/buy/marketplace-insights/overview.html), is a Limited Release API for approved developers and returns only 90 days of sales. Plan for this: apply early, and build a fallback that samples sold-listing pages within each platform's terms, plus your own sales history as it accumulates.
- **Deduplicate relists.** The same unsold item relisted three times looks like three listings. Cluster near-duplicate photos and titles before computing supply.
- **Demand by variant, not by model.** Colorway, size and condition move sell-through more than the model name. Report demand at the attribute level Lens identified.

Output: `demand {sold_90d, active_now, sell_through, median_days_to_sell, by_channel{}}`.

## Agent 3: Studio (produce the stock-quality image)

Studio makes the seller's real photo look like a catalog shot without changing the object.

- **Edit the background, never the item.** Segment the object (a promptable segmentation model), replace the background with seamless white or a soft gradient, and synthesize a contact shadow so it does not float. Pixels inside the object mask are locked except for global color correction.
- **Correct color from a reference, not by eye.** A grey card or the white of the tag gives a white-balance anchor. Wrong color is the top reason for "not as described" returns in apparel.
- **Hero-image rules per platform.** eBay favors a plain background and a large object fill; Depop and Instagram-style feeds reward lifestyle context and a 4:5 crop. Render one master, then export platform-specific crops.
- **Catalog stock images are a second, labeled slot.** When Lens found a GTIN or ePID, attach the manufacturer image as "reference photo", never as the hero.
- **Integrity check.** Compute a perceptual diff between the original and edited object region; reject edits that alter shape, logos or defects. Keep the original as the last gallery image.

Output: `images {hero, gallery[], platform_crops{}, original_kept: true}`.

## Agent 4: Almanac (season of demand)

Almanac decides *when* to list: now, or hold until a known demand window.

- **Decompose, then forecast.** Weekly sold volume by category splits into trend, yearly seasonality and holiday spikes (STL decomposition or a Prophet-style model with holiday regressors). Search-interest series (Google Trends) serve as a leading indicator when sold data is thin.
- **List before the peak, not on it.** Search-ranking systems reward listings with some age and engagement. The rule of thumb is to list 2-4 weeks ahead of the window: coats in late September, pool gear in April, Halloween costumes by early October.
- **Hold vs. sell is an inventory-cost calculation.** Holding costs shelf space, capital and the risk that a newer model lands. Almanac compares expected peak price minus holding cost against selling now, and recommends "hold" only when the margin clears it.
- **Watch event-driven demand.** Discontinuation announcements, a celebrity sighting or a reboot can move a niche item more than the calendar. Subscribe to news and trend feeds for items Lens tagged as collectible.

Output: `timing {list_now: bool, best_window, expected_uplift_pct, reason}`.

## Agent 5: Ledger (optimal price)

Ledger prices to maximize what the seller *keeps*, given how long they are willing to wait.

- **Model time-to-sale, not just price.** Fit a survival model (Cox or accelerated failure time) on sold *and* unsold comps, with price relative to the comp median as a covariate. Unsold listings are right-censored data, not noise to discard. This gives P(sold by day T | price).
- **Optimize net, per channel.** Fee schedules differ sharply (Poshmark takes 20% on a $100 sale, Mercari 10%, per [Closo's 2026 breakdown](https://closo.co/blogs/crosslisting/cross-list-poshmark-to-mercari)), so the best list price differs by channel for the same item.
- **Anchor for offers.** On offer-heavy platforms, list 10-20% above the target so the expected accepted offer lands on it. Set auto-accept and auto-decline floors from the model, not from feel.
- **Scheduled markdowns.** If no sale by the model's median day, drop 5-10% and re-score. A fixed markdown ladder beats ad-hoc cuts and refreshes the listing in search.

The objective, where f(p) is fees, s is shipping cost and h is holding cost per day:

```latex
p^{*} = \arg\max_{p} \; \big(p - f(p) - s\big)\,P(\text{sold} \le T \mid p) \; - \; h \cdot \mathbb{E}[\min(\text{days}, T) \mid p]
```

Output: `price {by_channel{list, floor, auto_accept}, curve[(price, p_sold_30d)], markdown_plan}`.

## Agent 6: Caliper (dimensions from the picture)

Caliper outputs length, width, height and estimated weight with an explicit error bar, because shipping cost depends on them.

- **One photo cannot give scale.** Monocular depth knows shape but not size: a toy car and a real car look the same. Every estimate needs a scale source, in this order of preference:
    1. A catalog spec when Lens found a GTIN (exact).
    2. Phone depth data: LiDAR on Pro iPhones via ARKit, or a short orbit video reconstructed with photogrammetry (±1-2% typical).
    3. A known reference in frame: a printed fiducial marker (ArUco or AprilTag) or a standard card, with homography correction for tilt (±3-5%).
    4. Priors from the identified category ("adult hoodie, size M") as a last resort, flagged low-confidence.
- **Size the box, not the item.** Carriers bill on dimensional weight (length × width × height ÷ a divisor, commonly 139 for US domestic in inches). Caliper picks the smallest standard box or poly mailer from the seller's supply list, adds padding, and hands that to Tether (Agent 10), or shows it to the seller when Tether is off.
- **Weight is inferred, then verified.** Estimate from material and volume, but require a scale reading at pack time. Underpaid postage gets charged back to the seller weeks later.

Output: `dims {item_lwh_cm, package, est_weight_g, method, error_pct}`.

## Agent 7: Muse (description for the person who wants it)

Muse writes for a specific buyer, not for the item.

- **Persona from evidence.** Mine the titles and descriptions of *sold* comps for the words buyers responded to: a collector wants "original box, serial matches", a parent wants "washed, smoke-free, fits age 4-5", a flipper wants model number and lot size. Pick the persona with the highest sell-through on that channel.
- **Title is a search query, description is a sales page.** Titles front-load the exact tokens buyers type (brand, model, size, color) within each platform's character limit. Prose belongs in the description.
- **Structured fields beat prose.** Filling every item-specific and attribute field (eBay item specifics, Depop style tags) affects search placement more than any sentence. Muse fills those first.
- **Disclose defects up front.** Every defect Lens boxed appears in the description with its photo. Honest disclosure cuts returns and protects seller ratings, which compound.
- **Grounded generation only.** Muse may state only facts present in the identity record. A post-check rejects any claimed spec that Lens, Caliper or the catalog did not supply.

Output: `copy {persona, title_by_channel{}, description, item_specifics{}, hashtags[]}`.

## Agent 8: Venue (best platform for this item, this season)

Venue picks the platform where this item's buyers are most active at the time it will be listed, by asking Almanac for a seasonal curve *per platform* instead of one market-wide curve.

- **Each platform has its own calendar.** The same jacket can peak on one app in September and on another in November, because their buyers differ in age, budget and buying occasion. Almanac fits a separate seasonal index for every (category, platform) pair and Venue reads all of them at the planned list date.
- **Borrow strength when data is thin.** A niche platform may have only a few dozen sales of a category. Fit the per-platform curves as a hierarchical model, so a sparse platform's curve shrinks toward the category's overall pattern instead of chasing noise.
- **Score is net per day, in season.** For each platform: expected net (Ledger's price minus that platform's fees and shipping) × the seasonal demand index for the sale window ÷ expected days to sale (Pulse). A $60 item that nets $54 in 5 days can beat $70 that takes 45.
- **Category fit sets the shortlist** before any scoring:

| Category | First platform | Second platform | Why |
| --- | --- | --- | --- |
| Mainstream fashion | Poshmark, Vinted | Depop | Fashion-first buyers, social sharing |
| Streetwear, sneakers | Grailed, Depop | eBay (authentication) | Collector buyers, brand search |
| Electronics, parts | eBay | Mercari | Model-number search, buyer protection |
| Bulky, local items | Facebook Marketplace | OfferUp | Pickup avoids freight |
| Collectibles, cards | eBay, Whatnot | Collector forums | Auctions and live sales find the top bidder |

- **Stagger, don't spray.** List first on the platform whose peak is nearest; add the second platform only if the item is unsold after Pulse's median days. This keeps the primary listing's early engagement concentrated, which most search rankings reward.
- **Re-check at markdown time.** When Ledger schedules a price drop, Venue re-scores platforms, since a better season may have opened elsewhere.

Output: `venue {primary, secondary[], seasonal_index_by_platform{}, list_on_date, reason}`.

## Agent 9: Keymaster (optional: automated listing with Apple passkey)

Keymaster uses passkeys for what they are actually good at: proving the seller approved each listing.

**Optional.** The seller picks one of two modes per platform:

- **Copy-paste mode (default).** Nextant shows the finished listing (title, description, price, item specifics, photos) with a copy button per field and a one-tap photo download, and the seller pastes it into the platform app. No marketplace connection, no automation risk.
- **Automated mode.** Keymaster publishes and delists for the seller, as described below.

- **The constraint most plans miss.** A passkey is a WebAuthn credential bound to one website's domain (its relying-party ID). Nextant cannot use the seller's eBay passkey to sign in to eBay on their behalf, by design. Passkeys authenticate the seller *to Nextant*; marketplace access comes from each platform's own authorization.
- **Passkey as a signature, not just a login.** Before publishing, the server issues a WebAuthn challenge that includes a hash of the final listing (price, photos, copy). Face ID or Touch ID signs it. The result is a tamper-evident record that a human approved exactly that listing, which matters once agents act autonomously.
- **Marketplace connections by API tier.**
    - eBay and Shopify: official OAuth APIs, server-side (eBay's Sell Inventory and Offer APIs).
    - Poshmark, Mercari, Depop, Vinted: no public listing API, per [Closo's integration notes](https://help.closo.co/a/how-the-crosslister-works). Use a user-present browser session the seller watches, or a "prepared draft + one-tap paste" handoff. Never store their marketplace passwords.
- **Inventory lock against double-selling.** When one channel reports a sale, Keymaster delists everywhere else within minutes, and holds a soft reservation so a second buyer is refunded cleanly if a race happens.

Output: `listing {channel_ids{}, approval_assertion, status, delist_hooks}`.

## Agent 10: Tether (optional, future phase: tag items and mail labels)

Tether makes it physically hard to ship the wrong item, using two scans that must agree.

**Optional and planned for a later phase.** Tether is the only agent that needs physical supplies (a label printer, tags, a scale), so the first release ships without it. Until then, the seller uses each platform's own shipping label and Nextant shows the item's hero photo and box size from Caliper at packing time.

- **Tag at intake, label at sale.** The item tag is printed the moment the item is photographed. The shipping label cannot exist yet: the buyer's address comes later, and Poshmark and Mercari issue their own prepaid labels. So Tether *binds* the platform's label to the item tag after the sale instead of trying to own the label.
- **A code humans can read aloud.** Each tag carries a QR code plus a 6-character code in Crockford Base32 (no I, L, O or U, so no 0/O or 1/I confusion) with a check character, a thumbnail of the hero photo and the bin location. The thumbnail lets a packer catch a mismatch with no scanner at all.
- **Two-scan pack gate.** At packing, scan the item tag, then the shipping label. Tether checks both resolve to the same order; a mismatch blocks the "mark shipped" step. This is the warehouse pick-pack-verify pattern, scaled down to a phone camera.
- **Weigh and photograph at the gate.** Capture the scale reading (catches Caliper errors before postage is underpaid) and a photo of the open box with the tag visible. That photo is the seller's evidence in an "item not as described" dispute.
- **Tag the bag, not the item** for apparel and anything where a sticker leaves residue: a zip bag or hang tag carries the code.

Output: `fulfillment {tag_code, bin, label_ref, pack_verified, weight_g, pack_photo}`.

## Roadmap, metrics and risks

Ship the listing workflow first; automated listing and physical tagging come last, as optional add-ons.

1. **Phase 1: Core listing (weeks 1-4).** Item record, Lens with OCR and barcode, Studio, Muse, and copy-paste mode for every platform. Gate: a seller goes from photo to a pasted listing in under 5 minutes.
2. **Phase 2: Money (weeks 5-10).** Pulse, Ledger's survival model, Almanac, Caliper. Gate: median realized price within 10% of Ledger's prediction.
3. **Phase 3: Placement (weeks 11-14).** Venue with per-platform seasonality. Gate: items listed on Venue's first choice sell faster than a fixed-platform control group.
4. **Phase 4: Optional automation (weeks 15-20).** Keymaster's automated mode, starting with eBay OAuth, plus the delist lock. Gate: zero double-sells over 500 crosslisted items.
5. **Phase 5: Future, physical (later).** Tether tags, the two-scan pack gate, and Caliper with LiDAR and fiducials. Gate: 50 real items shipped with zero mix-ups.

Metrics that matter: minutes from photo to live listing, realized price ÷ predicted price, days to sale vs. predicted, return rate, mis-ship rate (target: 0), and net per item after fees and postage.

| Risk | Effect | Mitigation |
| --- | --- | --- |
| Sold-data access denied | Pricing runs on asking prices | Apply for eBay access early; log your own sales from day one |
| Platform terms on automation | Account suspension | Official APIs where they exist; user-present sessions elsewhere |
| Misidentified variant | Wrong price, returns | Confidence threshold plus targeted re-shoot request |
| AI-altered photos | "Not as described" claims | Locked object mask, original photo always kept |
| Underpaid postage | Carrier chargebacks | Weigh before buying postage (Tether's pack gate enforces it later) |

## Sources

- [eBay Marketplace Insights API overview](https://developer.ebay.com/api-docs/buy/marketplace-insights/overview.html): Limited Release, 90-day sold history
- [Closo: how the crosslister works](https://help.closo.co/a/how-the-crosslister-works): which marketplaces have public listing APIs
- [Closo: Poshmark vs. Mercari fees](https://closo.co/blogs/crosslisting/cross-list-poshmark-to-mercari)
