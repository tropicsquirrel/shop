# Clip-Boy Run 2 — Preorder Storefront: Design Spec

**Date:** 2026-08-21 · **Rev 2.1** (reconciled against 3 reviews + owner rulings)
**Status:** DRAFT — build-ready on money/security/mode per the rev-2.1 re-review; pending owner approval.
**Nothing goes live** until the owner flips the go-live flag. Commits allowed on branch
`run2-preorder` only; no merge to `main`, no redeploy, no go-live without explicit go-ahead.
**Author:** shop agent (with owner "Data" Owen)

> **Rev 2 changelog:** resolved the 1-card-vs-3 fork (one card, **multi-color cart lines**);
> KS per-backer cap kept and **set to 5** (scoped to exclusive-color units); international =
> **country allowlist + manual SDN screen before each ship**; email = keep Gmail + **queue on
> quota** + **show order code on success page**. Added: server-side donation clamp, color→price
> allow-map, `allowed_shipping_countries` gating, backend-flag-driven open/closed mode with a
> countdown ported to the public page, go-live runbook, Stripe test-key preview path. Reclassified
> international checkout and the public-page open/close mode as **NEW** work (not "reuse").
>
> **Rev 2.1:** quantified the donation clamp ($ max + reject rule) and marked the donation line
> **non-taxable**; made the KS exclusive cap a **cumulative cross-order, cross-surface** counter and
> resolved the `ks/`-page price/cap collision (**retire `ks/` before go-live**); reconciled §4.2 vs
> §4.3 for **US territories**; suppressed the legacy `sellableTotal()===0` trigger under `PREORDER_LIVE`;
> fixed the Stripe param name to `shipping_address_collection[allowed_countries]`.

---

## 0. Live-store item to verify NOW (independent of run 2)
A review flagged that the `donation`/`amount` checkout param may not be clamped server-side
(the KS page clamps only client-side, bypassable by calling `/exec` directly). If
`create_checkout` does not coerce `donation` to a **non-negative integer within a sane max**
before adding it to the total, a crafted `&donation=-5000` could *subtract* from a badge order
**on the live store today**. `Code.gs` is editor-only — **owner to paste `create_checkout` for
verification**, or confirm the clamp exists. Rev-2 mandates the clamp for run 2 regardless (§8.1).

---

## 1. Goal & framing
Convert the **existing storefront page** (`index.html`, where `brycebadges.com` already forwards
interest traffic) from *sold-out interest-gathering* into a **preorder** store for a 2nd run.

Design principle: **one product, color choices — not "editions."** Run 1's edition cards confused
buyers. Charge **in full at preorder** (reuses Stripe Checkout). Build to whatever sells (no run
cap). Ship after production (~**mid-December 2026**).

A single order may mix **multiple colors** (e.g. a validated backer buying 1 SpaceBadge + 1 Overseer
+ 1 gray/black + 2 custom trims = 5 units). Each color is its own cart line with its own qty.

### Non-goals (YAGNI)
Deposit/installment billing; live inventory cap / sold-out run logic; DDP/VAT collection; a new
KS-validation subsystem (reuse `ks_auth`); free-text arbitrary trim colors (curated list only);
DEF CON pickup (DC34 is past — mail-order only). *Pickup removal: owner to confirm (§13).* 

---

## 2. Reuse vs NEW (honest inventory)
Reviews corrected the first draft's over-broad "reuse" framing. Split:

**Genuinely reused (verified against `index.html`/`ks/index.html`):**
- `sku:color` **variant cart-key data model** + per-color qty rows (`state.cart[sku+':'+color]`,
  `cartQtyForSku` prefix match). ← this is exactly what multi-color orders need.
- Stripe **full-charge** path (`action=create_checkout` → `handlePaymentSuccess`).
- **Domestic** qty-tiered shipping display + delivery toggle.
- KS **validate** flow: `GET /ks_auth?email=…&backer_id=…` through the CF per-IP proxy →
  `handleKsAuth`/`ksAuth*` (`security.gs`), per-email lockout + `checkRateLimitN`.
- Per-order cap machinery (`currentMax()`), consent tick (§2a), Discord pings, polling payment
  detection, refund sync, `#donateOnlyBtn` donation-in-closed-mode guard.

**NEW work (do not mistake for parameter tweaks):**
- **International *checkout*** — run 1 had only an *email inquiry* for intl (`action=intl_inquiry`;
  `create_checkout` only ever passed `delivery=pickup|ship` + a US address). Buy-now intl needs: a
  new delivery value, `allowed_shipping_countries` handling, intl address collection, the intl
  shipping tier in the **server** recompute, and the no-tax branch.
- **Preorder open/closed mode on the PUBLIC page** — today the public page derives interest mode
  purely from `sellableTotal()===0` and has **no countdown** (that's KS-page only). With no run cap,
  `sellableTotal()` may never hit 0, so the old trigger won't open/close correctly. Mode must be
  **driven by backend flags** and a **countdown ported to the public page**.
- The **multi-color add-a-color selector + carousel preview** UI (the variant *data* model is reused;
  this interaction is new).

**Source-of-truth caveat:** `Code.gs`/`security.gs`/`Interest.gs`/`ShipPrefs.gs` live **only in the
Apps Script editor** (gitignored). Backend changes below are interface-level; verify names/lines in
the editor first, and deliver `.gs` deltas **as files** (chat→editor paste truncates long lines).

---

## 3. Product & pricing model

### 3.1 One card, multiple color lines
- **One product card:** "Clip-Boy" (gunmetal body).
- A **color selector** adds colors to the cart; each added color is a **line with its own qty**:
  - **Black** — base price, default.
  - **Custom** — curated dropdown of sourceable colors; **+$10/unit**; carousel swaps to that color.
  - **KS-exclusive** (SpaceBadge, Overseer) — appear in the selector **only after a backer validates**;
    **base price, no +$10, no discount** (color access is the whole perk).
- Total across all lines is capped at **5** (§4.1). A single order can mix all classes.
- Carousel shows the currently-previewed color (placeholder images for now).

### 3.2 Pricing config + authoritative server-side price map
Prices in **Script Properties** (backend authoritative); frontend only displays. Backend
**recomputes every order total** at `create_checkout` — client is never the price authority.

| Key | Meaning | Placeholder |
|---|---|---|
| `PRICE_BASE_CENTS` | base (Black / KS-exclusive) | `TBD` (supplier quote pending) |
| `PRICE_CUSTOM_UPCHARGE_CENTS` | added per custom-trim unit | `1000` ($10) |

**Color → price-class allow-map (server-side, authoritative):** an explicit map
`{ colorId → { price_class: base|custom, ks_gated: bool } }`. Rules at `create_checkout`:
- Parse a **canonical `colorId`** from each `sku:color` cart key; a **malformed key ⇒ reject**.
- Any cart color **not in the map ⇒ reject** the checkout (no "unknown defaults to base").
- `price_class=custom` ⇒ add the upcharge per unit.
- `ks_gated=true` ⇒ require a **validated backer** for this request or reject (§5).

This enumeration is what makes price integrity *provable* rather than asserted.

---

## 4. Shipping, tax & OFAC

### 4.1 Tiers (by total badge qty in the order)
| Destination | 1–3 | 4–5 | >5 |
|---|---|---|---|
| **Domestic (US)** | $20 | $30 | contact us |
| **International** | $35 | $55 | contact us |

- **Soft cap = 5 total.** Client disables buy at >5 and routes to the **contact path**; **server also
  rejects >5** (backstop). Boundary tests: qty 5 allowed, 6 blocked, both client and server.
- **`currentMax()` retune:** default cap becomes **5** for a single shipping vocabulary (today
  `maxPickup=10` lets a 10-cart build before selection — must not survive pickup removal).
- Shipping computed **server-side** from the selection + total qty; frontend value is display only.

### 4.2 Selection vs shipped-to address cannot diverge
Shipping/tax are chosen from the **Domestic/International** selection, but Stripe collects the address
*after* the amount is set. To stop "pick Domestic $20 → enter a UK address", gate the Stripe param
`shipping_address_collection[allowed_countries]` by the selection:
- **Domestic ⇒ `['US','PR','GU','VI','AS','MP']`** — the US **plus its territories** (their own ISO
  codes), so the §4.3 "territories route as Domestic" promise isn't blocked by a US-only set. APO/FPO
  are `US` addresses (state AA/AE/AP) and are already covered.
- **International ⇒ the curated non-embargoed intl list** (`INTL_COUNTRIES`, already excludes
  Cuba/Iran/NK/Syria), minus any code in the Domestic set.
Selection and address then physically cannot disagree.

### 4.3 Tax
- **US:** Stripe Tax at checkout (as run 1). Confirm the **tax-on-shipping** setting matches the cart
  note "tax on badges & shipping."
- **International:** **no tax/VAT/duties collected** (DDU); checkout copy states customs/duties are the
  buyer's responsibility. $35/$55 = shipping only.
- **US territories / APO-FPO** (PR, GU, VI, AS, MP): route as **Domestic** for postage (their ISO
  codes are in the Domestic `allowed_countries` set, §4.2); Stripe Tax handles their differing tax
  treatment. Confirm at review (§13).

### 4.4 OFAC / SDN (owner ruling)
Automating intl checkout must preserve the screening the old inquiry flow did by hand:
- Stripe `allowed_countries` limited to the curated list (country-level embargo gate).
- **Manual SDN/party screen before each international shipment** (hold-and-screen): intl paid orders
  are flagged for a human check against SDN before fulfillment. This is a compliance commitment, so
  it appears in the fulfillment runbook + the Orders/Discord flag for intl orders.

---

## 5. KS backer flow (owner rulings folded in)
- Backer enters **email + backer #** → `GET /ks_auth` (through the CF rate-limit proxy) →
  `handleKsAuth` validates against the **original SpaceBadge backer export** (same list as run 1;
  prior validations still work). Rate-limited via per-email lockout + `checkRateLimitN`.
- On success: **SpaceBadge + Overseer** colors become selectable (client unlock).
- **Server re-validation at checkout:** if a cart contains a `ks_gated` color, `create_checkout`
  **re-runs the real backer-list validation** (not mere presence of the email/backer# params) and
  routes that check through the **same per-email lockout** — because `create_checkout` is *direct to
  Apps Script, off the CF proxy*, so the lockout is the only per-identity throttle on that path.
- **Per-backer cap = 5 exclusive-color units** (owner: "reset to 5"), enforced server-side as
  anti-abuse. **The cap is CUMULATIVE across the backer's paid + reserved orders** (bind to the count
  `ks_auth` already returns — `remaining`/`cap`), checked at `create_checkout`, and **must include the
  35-min reserved window** (§ reservation timing) to avoid both oversell and false lockout. A per-order
  check is toothless here: the universal per-order cap is also 5, so a per-order exclusive cap could
  never bind within one order — only the cumulative sum stops a leaked email+backer# minting 5-at-a-time
  forever. *(Confirm §13: cap counts exclusive-color units only, vs. all units.)*
- **Single exclusive surface.** The existing `ks/index.html` sells the *same* exclusive colors at a
  *different* price ($120 flat) with its *own* `ksCap=5` counter. If both stay live, a backer gets 5
  exclusives **on each** at **two prices**, and the caps don't share. **Resolution: retire `ks/` before
  go-live** so run 2's main-store panel is the only exclusive surface, one price (base), one cumulative
  counter. *(Owner confirm §13.3.)*
- **No discount, no separate KS price.**
- **Entry point:** a "Kickstarter backer? Unlock your colors" **validate panel on the main store**
  (fits the one-store framing). *(Confirm §13.3 — ties to retiring `ks/`.)*

---

## 6. Preorder semantics
- **Framing = PREORDER everywhere:** buy button, cart, **Stripe success page**, and confirmation email
  say *preorder* and show the **est. ship window** (`SHIP_ESTIMATE`, config; default "end of December
  2026"). Biggest expectation-management risk of charging upfront — belt-and-suspenders.
- **Charge full** at checkout.
- **Open/closed driven by backend flags** in the `status` payload: `PREORDER_LIVE` (go-live) and
  `PREORDER_DEADLINE` (close date). A **countdown is ported to the public page**. At/after deadline →
  **preorders-closed** state; the **donation/College-Fund flow stays reachable**. The working
  `#donateOnlyBtn` reveal lives only inside `enterInterestMode()` (reachable today via
  `sellableTotal()===0`/preview) — the deadline-closed path **must call that same surfacing logic**, not
  merely "show a button" (the exact 2026-08-16 regression class). Also **suppress the legacy
  `sellableTotal()===0` interest trigger while `PREORDER_LIVE`** (or guarantee preorder items report
  `available>0`), so "unlimited" preorder stock encoded as `available:0` can't wrongly show sold-out.
- **Refund/cancellation:** full refund on request **before the deadline / before production starts**,
  issued **manually**. **After production starts:** state the stance explicitly (e.g. no cancellation
  once units are in production) so refund requests don't age into chargebacks. Name a **backup
  approver** so a single-owner absence can't strand a refund past a dispute window.

---

## 7. Frontend UX (index.html)
1. **Hero:** "Preorder — Run 2," countdown to `PREORDER_DEADLINE`, est-ship line.
2. **Product card (single):** gunmetal Clip-Boy, base price, **color selector** (Black | Custom → color
   dropdown → carousel swap | KS-exclusive after validate), each chosen color becomes a **qty line**;
   running subtotal (+$10 per custom unit).
3. **KS unlock panel:** email + backer# validate (reuse), unlocks exclusive colors.
4. **Cart summary:** per-color lines, Domestic/International selector → shipping tier, US-only tax note,
   preorder + est-ship note, refund note, 30-min checkout note, §2a consent tick, total.
5. **>5 units:** buy disabled → **contact path** (repurpose the modal; **fix its fields** — a bulk
   *domestic* order shouldn't be asked for Country; make Country optional/relevant only when intl).
6. **Success page:** **display the order code** (so a queued/late email never leaves a buyer without it)
   + preorder + est-ship + refund copy.
7. **Go-live gate:** entire preorder UI behind the backend `PREORDER_LIVE` flag **OR** the secret
   preview token **`?preview=majesticm00se`** (owner-chosen). Default (no token, flag off) = the current
   interest page, byte-for-byte unchanged for the public. The token is the **interim reveal** so the
   owner can view/click the preorder page before it's ready; `PREORDER_LIVE` stays **off** until launch.
   The token must not collide with the existing `?preview=interest` handling. `?preview` survives the
   brycebadges.com forward (utm params pass through — owner-confirmed).

---

## 8. Backend changes (interface level; `.gs` delivered as files)
1. **Donation clamp (concrete):** on **both** `create_checkout` (`donation`) and `donate` (`amount`),
   on **both** pages, **reject** (don't floor) any value that is not a **non-negative integer** ≤
   `DONATION_MAX_CENTS` (Script Property, default **$2000 = 200000**). Server is the only guard (the
   public store has no client-side max). The **donation line is NON-TAXABLE** — set its Stripe
   `tax_behavior`/exempt so the College-Fund contribution isn't taxed (the cart copy taxes "badges &
   shipping" only, §4.3).
2. **Server recompute + color allow-map (§3.2):** total = Σ(base or base+upcharge per line) + shipping;
   reject any color not in the map; reject `ks_gated` colors without a validated backer.
3. **International checkout (NEW):** third delivery value; intl shipping tier ($35/$55); no-tax branch;
   `allowed_shipping_countries` gated by selection (§4.2); intl orders flagged for SDN screening (§4.4).
4. **Qty ≤ 5** enforced server-side (reject >5 with a "contact us" error).
5. **KS re-validation at checkout** through the per-email lockout (§5); per-backer exclusive cap = 5.
6. **Preorder flags** in Script Properties: `PREORDER_LIVE`, `PREORDER_DEADLINE`, `SHIP_ESTIMATE`;
   `status` reports them so the frontend renders the right mode (and a not-yet-redeployed backend keeps
   the public in interest mode — a *safe* half-apply).
7. **Orders sheet:** capture trim **color** (existing `H`). **Do NOT add an "international" column** —
   **infer intl from the existing `X Country`** (avoids the M/N clobber class). **No writes to `M`
   (Fulfilment_Status) / `N` (KS_Backer_ID).** Enumerate columns before any `setValue`/`setValues`.
8. **Confirmation email:** preorder framing + est-ship + refund; sent via GmailApp from
   `coruscantproductions@gmail.com` (inbox-safe via gmail.com SPF/DKIM). **On quota exhaustion
   (100/day), queue** the confirmation (record the order + code in the sheet regardless; the success
   page already shows the code, so no customer is blocked). A daily trigger drains the queue.
9. **Success-page order code** surfaced from the paid order.

---

## 9. Rollout / go-live (runbook, not "one flip")
- **Develop on branch `run2-preorder`.** `main`/live page untouched until flip.
- **Preview safely:** a **staging Apps Script deployment** carrying **Stripe TEST keys** + a **test
  Orders tab** in its own Script Properties. Reviewers hit the preview flag against staging so
  `create_checkout` click-through creates **test** sessions, never live charges. (This effectively
  *requires* a separate deployment — matches the frozen-snapshot lesson from ship-prefs.)
- **Go-live is a coordinated set — enumerate all coupled actions:**
  1. `PREORDER_LIVE=true` (Script Property)
  2. merge `run2-preorder` → `main`
  3. Apps Script **Public redeploy → new version**
  4. **repoint `APPS_SCRIPT_URL`** in `index.html` if a fresh deployment is used (frozen-snapshot trap)
  5. front/back agree on the exact **`delivery` param vocabulary** (`pickup`/`ship` → domestic/intl) —
     mismatched tokens silently mis-key shipping/tax
- **Safe half-apply:** because mode is driven off the backend `status` flag (§8.6), a not-yet-redeployed
  backend keeps the public in interest mode rather than showing a broken preorder UI.
- **Verify** the live `status` echoes `PREORDER_LIVE=true` before announcing. Reversible (flip back).

---

## 10. Anti-abuse (summary)
Server-side donation clamp (§8.1); authoritative color→price allow-map, unknown ⇒ reject (§3.2);
KS re-validation + per-email lockout + per-backer cap 5 on the checkout path (§5); qty ≤5 server
backstop (§4.1); `allowed_countries` gating (§4.2); honeypot on new forms; `ks_auth` per-IP proxy.

---

## 11. Testing plan (synthetic only; test Orders tab; zero writes to live sheet)
- **Mixed multi-color cart:** Black + custom(+$10) + KS-exclusive in one order; per-line qty; total math.
- **Price integrity:** tampered client amount overridden; unknown color rejected; custom upcharge applied.
- **Donation:** negative/oversized/non-integer rejected; donation-only in **every** mode incl. the new
  **deadline-closed** mode (regression guard).
- **KS:** validate ok; **spoof at checkout rejected** (crafted request w/ exclusive color, no valid
  backer); per-backer cap 5 enforced **cumulatively across multiple orders** (not resettable by placing
  a new order); lockout fires on the checkout path.
- **Donation:** value = max allowed, max+1 rejected, `5.5`/`1e9`/negative rejected; the donation line
  is **not taxed** in a US order.
- **Territories:** a Puerto Rico (PR) address completes a **Domestic**-priced order (not blocked).
- **Shipping/tax:** domestic 1–3 / 4–5; intl 1–3 / 4–5; qty 5 allow / 6 block (client+server);
  Domestic selection cannot ship to a non-US address; no tax on intl; US territory routing.
- **Deployment:** preview click-through hits **Stripe test** keys; go-live runbook dry-run; `status`
  flag drives mode; `APPS_SCRIPT_URL`/`delivery` vocab consistent.
- **Email:** confirmation copy; quota-exhaustion path queues + success page still shows code.

---

## 12. Copy checklist
"PREORDER" on buy button, cart, **success page**, email subject/body · est-ship visible pre-purchase +
in confirmation · refund/cancellation (incl. post-production stance) visible pre-purchase + in email ·
intl duties-are-buyer's-responsibility at checkout · **order code on success page** · public
appreciation stays "Clip-Boy" (no minor's name publicly; customer emails may say "Bryce & dad" per
run-1 precedent).

---

## 13. Owner rulings (CONFIRMED 2026-08-21) + remaining set-later items
1. ✅ **DEF CON pickup removed** — mail-order only.
2. ✅ **KS per-backer cap of 5 = exclusive-color units only** (not all badges in the order).
3. ✅ **Retire `ks/` before go-live** and **redirect `ks/` → `/`** — the main-store panel is the only
   exclusive-color surface (one price = base, one cumulative cap).
4. ✅ **US territories/APO-FPO route as Domestic** (via the §4.2 country set).
5. ✅ **Stripe posture acknowledged** (owner briefed): the refund window (closes at preorder deadline)
   is the owner's *promise*; the **card-network chargeback window (~120 days from charge)** is separate
   and outlives it, and a small account doing preorders may face a **rolling reserve**. Mitigations
   (no code): put ship-window + refund terms **on the Stripe checkout/product too**; keep a cash buffer;
   watch dispute rate. Refund policy itself unchanged.
6. **Preview gate token = `?preview=majesticm00se`** (owner-chosen, hard-to-guess). Public sees the
   current interest page; only this token reveals the preorder UI. Do **not** go live / no
   `PREORDER_LIVE=true` until the owner says so — the token is the interim reveal.

**Set-later (non-blocking):**
- **Price & deadline values** — TBD (owner sets in Script Properties once supplier quote + open date are
  set). `SHIP_ESTIMATE` default "mid-December 2026."
- **Carousel assets** — placeholders now; real per-color renders later.

---

## 14. Process
PLAN (this doc) → 2 independent reviews **[done]** → focused money-critical re-review **[done → rev
2.1]** → **owner approves spec** → implement on `run2-preorder` → **2 impl-vs-plan reviews** → owner
go-live decision. No implementation before owner approval; no merge to `main` / no go-live / no
redeploy without explicit go-ahead.
