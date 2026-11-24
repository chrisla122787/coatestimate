# CoatEstimate

An estimating tool for **independent interior repaint painters** —
owner-operators and small crews of 1–5 painters replacing the paper notebook
and the spreadsheet. Tagline: **"Price every paint job for profit."**

It is deliberately **not** a CRM and **not** a PaintScout replacement. It
does one job: profitable estimates with a clean client PDF.

Most painters price repaints from experience — a bedroom is $650, the crew
is $500 a day — but never check those prices against their real costs, so a
"busy" month can still lose money. CoatEstimate takes the prices the painter
already uses (Quick mode) or full room measurements (Measured mode), builds
up the true cost underneath, and shows the profit, the true margin, and the
minimum price for the painter's target margin — privately, before the
estimate goes out. If a price falls below target, a **Profit Risk Warning**
says so in plain words.

Pricing: **$19/month founding-member pricing** for the first 25 customers,
locked in while the subscription stays active. Standard price is **$29/month**
after the founding spots are gone. The estimator and a watermarked preview
are free; Pro unlocks unlimited clean PDFs, saved estimates, the estimate
dashboard, and backup export. **The first clean PDF is free** (portfolio
rule — see below).

Plain static HTML/CSS/JS — no build step, no backend.

## Files

| File | What it is |
|---|---|
| `index.html` | Landing page: promise, how it works, free/Pro features, a static sample estimate, founding pricing, privacy + billing/cancellation notes. |
| `app.html` | The tool: Quick + Measured estimator, watermarked estimate preview, PDF estimate (jsPDF via CDN), estimate dashboard with statuses and totals, backup export/import + CSV export. |
| `README.md` | This file. |

## Before launch

- In `index.html` and `app.html`, replace the `STRIPE_LINK` placeholder with
  the real Stripe subscription payment link for CoatEstimate ($19/month
  founding price). There are two in `index.html` (hero + pricing), and in
  `app.html` the subscribe button's href is set from the `STRIPE_LINK`
  constant at the top of the inline script. (Placeholders sit inside quoted
  strings — replace the quoted value only, then re-extract the inline script
  and run `node --check` before deploying.)
- Contact email in the footer is `chrisla122787@gmail.com`.

## The two modes

The mode toggle at the top of the builder picks how this estimate is priced.
**Quick is the default.** Every saved estimate records which mode made it;
the dashboard, statuses, totals, and backup treat both modes identically.

### Quick mode (default)

The painter adds simple lines:

- **Room lines** — a room name + their usual flat price for that room.
- **Crew-day lines** — days × their selling day rate.

Plus job cost inputs: **materials $, prep $, travel $, other $**.

The model (deliberately the simplest coherent one):

```
lineAmount(room) = the entered flat price          ← these ARE selling prices
lineAmount(crew) = days × selling day rate
totalPrice       = Σ lineAmounts

laborCost  = Σ crew-days × loadedDayCost           ← default $320/day, editable
totalCost  = laborCost + materials + prep + travel + other
profit     = totalPrice − totalCost
trueMargin = profit ÷ totalPrice × 100
minPrice   = totalCost ÷ (1 − targetMargin)        ← target = the margin % setting
```

Notes on the model:

- Room-line prices are the painter's selling prices, so Quick never
  auto-prices. The margin % setting is the **target**: it drives `minPrice`
  and the Profit Risk Warning, nothing else.
- Labor cost comes only from crew-day lines (days × loaded daily labor
  cost — what a crew-day actually costs the business, wages + burden).
  A job priced purely per-room with no crew-day line shows labor cost $0;
  painters who want labor in the cost build-up add a crew-day line.
- **Profit Risk Warning:** when `trueMargin < targetMargin` (and cost and
  price are both > 0), the private panel shows the warning plus the minimum
  price that would hit the target.
- The client PDF lists the lines by name with their scope ("Per-room flat
  price" / "Painting crew — N crew-days") and prices. **No measurements
  exist in Quick mode, and none are shown.**

### Measured mode

Full measurement math (below): square footage from dimensions, pooled
gallons, primer, prep, labor hours at the crew rate, every job-level cost,
and margin or markup auto-pricing — with optional manual overrides for
total hours and final price.

## Free vs Pro

**Free (no payment, no account):**

- Full estimator — both modes, every input, live math, profit-risk and
  low-margin warnings.
- Watermarked estimate preview rendered on screen (diagonal
  "PREVIEW — CoatEstimate" watermark), showing company name, client, scope
  lines, and prices.
- **One complete branded PDF estimate, free and unwatermarked** (see the
  first-estimate-free rule below).

**Pro (subscription):**

- Unlimited clean branded PDF estimates: company logo + contact info,
  client and project, per-area scope with measurements and coats (Measured)
  or per-line scope (Quick), line prices, total, validity period (default
  30 days), included/excluded terms, an optional deposit line, start-window
  wording, and a client acceptance signature block.
- Saved estimates + estimate dashboard (statuses, pipeline/accepted totals,
  duplicate). Saved estimates record their mode.
- Backup export (JSON) / import-restore, and CSV export of estimates.

**The customer PDF and preview never show internal numbers — in either
mode.** Paint prices per gallon, coverage, production rates, crew rate,
loaded day cost, costs, profit, markup, and margin appear only in the
private "Your numbers" panel — the estimate carries scope and prices only.

### First-estimate-free rule

Free users can download exactly **one** complete, unwatermarked, branded PDF
estimate. The counter (`coatestimate.freepdf.v1` in `localStorage`) is
incremented **only after the PDF is successfully generated** — if the PDF
library fails to load, the free PDF is not consumed. After the free PDF is
used, the PDF button asks the user to subscribe or unlock Pro. Pro users
never consume the counter.

## ⚠️ MVP access note (read before launch)

Pro unlocking is **client-side only** for launch validation:

- The "I already subscribed — unlock Pro" button in `app.html` simply sets a
  `localStorage` flag (`coatestimate.pro.v1 = "1"`) in the visitor's own
  browser. There is **no real payment verification** — no server checks the
  Stripe subscription, and anyone who finds the button can unlock Pro without
  paying.
- This is a deliberate MVP trade-off to validate demand cheaply on a $0
  static stack (the landing page and app say so in plain words: unlocking is
  on the honor system during opening validation).
- Before scaling past validation, replace this with a real entitlement check
  (e.g. a small serverless function that verifies the Stripe subscription by
  email before issuing an unlock token).

## The Measured-mode formula

All math is in `areaMath()` / `calculateMeasured()` in `app.html`
(`calculateQuick()` for Quick mode; `calculate()` dispatches on mode).

### Measurements (per area)

```
perimeter   = 2 × (length + width)
wallSqFt    = max(0, perimeter × wallHeight − doors × 21 − windows × 15)
ceilingSqFt = length × width            (only when "Paint ceiling" is on)
trimSqFt    = trimLinearFt × 0.5        (1 lin ft of 6-inch trim ≈ 0.5 sq ft)
```

Door and window deductions are the industry-standard 21 sq ft and 15 sq ft.

### Paint — POOLED gallons convention

Paint is **pooled across the whole job by surface** (walls pool, ceiling
pool, trim pool, primer pool), because that's how paint is actually bought
— one order for the job, not a separate rounded gallon per room:

```
coatSqFt(surface) = Σ over areas of (sqFt × coats)
gallons(pool)     = ceil(poolCoatSqFt ÷ coverageSqFtPerGal)
paintCost(pool)   = gallons × pricePerGallon
```

Each pool's cost is allocated back to its areas by coat-sq-ft share, so
client line prices still weight by area cost. Exact (fractional) gallons
are never used for cost. Defaults: coverage 350 sq ft/gal (walls,
ceilings), 400 (trim); prices $38/gal walls, $32/gal ceilings, $45/gal
trim — all editable, and all persist across sessions (see Data & privacy).

Pooling example: two small rooms with 70 sq ft of walls each at 2 coats
contribute 140 coat-sq-ft each. Per-area rounding would buy
ceil(140 ÷ 350) = 1 gal **each** (2 total); pooled, ceil(280 ÷ 350) =
**1 gal total**.

### Primer (per area, optional)

An area can be marked "prime first" with 1–2 primer coats. Primer covers
the surfaces that get finish paint (walls + ceiling, not trim):

```
primedSqFt     = wallSqFt + ceilingSqFt
primerCoatSqFt = primedSqFt × primerCoats        → feeds the primer pool
primerGallons  = ceil(Σ primerCoatSqFt ÷ primerCoverage)     (default 300)
primerCost     = primerGallons × primerPrice                 (default $42/gal)
primerHours    = primerCoatSqFt ÷ wallProductionRate
```

Primer gallons and cost are their **own line** in the private panel. The
client PDF scope mentions "primer + N coat(s)" for primed areas — no
figures.

### Labor

```
paintingHours(surface) = sqFt ÷ productionRate        (sq ft per hour)
areaPaintingHours      = walls + ceiling + trim hours
prepHours              = prepFactor × areaPaintingHours
                         OR the area's explicit prep-hours override
                         (blank = use the factor)
areaHours              = areaPaintingHours + primerHours + prepHours
laborCost              = totalHours × crewRate
```

Production-rate defaults: walls 150, ceilings 120, trim 100 sq ft/hour —
editable and persistent. Prep factors (constants): **light 0.10, standard
0.25, heavy 0.50**. Crew rate default $45/hour.

**Hours override:** a job-level "total hours" override replaces the
computed total — each area's hours scale proportionally (or split evenly
if nothing was measured) and labor cost follows.

### Job totals and pricing

```
otherCost = supplies + equipment + protection/masking + travel + overhead
totalCost = Σ area (allocatedPaintCost + laborCost) + otherCost

MARGIN mode (default):  price = totalCost ÷ (1 − margin% / 100)
MARKUP mode:            price = totalCost × (1 + markup% / 100)
PRICE OVERRIDE:         price = the entered override (wins over both modes)

profit     = price − totalCost
trueMargin = profit ÷ price × 100     ← margin is a % of the PRICE, in both modes
minPrice   = totalCost ÷ (1 − targetMargin%)
```

Why divide by `(1 − margin)` in margin mode: a profit margin is a share of
the final price, not a markup on cost. Pricing this way means the margin
the user picks (say 30%) is the share of the price that is actually
profit. Marking cost up by 30% would only deliver a 23.1% true margin.

**Price override + risk warning:** when a final-price override is set, the
override is what prints on the estimate. If it prices the job below the
target margin, the private panel shows the Profit Risk Warning and the
minimum price — the same warning Quick mode uses.

Warnings: true margin under 10% (Measured, auto-priced) shows a
plain-language caution; a target margin ≥ 100% in margin mode is rejected
as unpriceable.

### Client line prices (Measured)

Each area's line price on the estimate is its share of the total price,
weighted by its share of measured cost:

```
linePrice(i) = price × areaCost(i) ÷ Σ areaCost
```

Lines are rounded to cents; the **last line absorbs the rounding
difference** so the lines always sum exactly to the estimate total.
(Job-level costs are spread across areas by this same weighting — they
never appear as their own client line. If areas carry no cost but a price
override exists, the full price lands on the last line.)

### Worked example — Measured, defaults, one room

12 × 10 ft room, 9 ft ceilings, 1 door, 1 window, 2 coats, standard prep,
ceiling on, supplies $60, all other job costs $0, margin mode at 30%:

- Wall area: (2 × 22) × 9 − 21 − 15 = **360 sq ft** · Ceiling: **120 sq ft**
- Wall pool: 360 × 2 ÷ 350 = 2.06 → **3 gal** × $38 = **$114.00**
- Ceiling pool: 120 × 2 ÷ 350 = 0.69 → **1 gal** × $32 = **$32.00**
- Painting hours: 360 ÷ 150 + 120 ÷ 120 = 2.4 + 1.0 = **3.4 hrs**
- Prep (standard 25%): **0.85 hrs** → total **4.25 hrs** × $45 = **$191.25**
- Job cost: $114 + $32 + $191.25 + $60 = **$397.25**
- Price: $397.25 ÷ 0.70 = **$567.50** · Profit: **$170.25** · True margin: **30.0%**

Same job in markup mode at 40%: price $397.25 × 1.40 = **$556.15**, true
margin 28.6%. (Verified end-to-end by the DOM-stub test suite.)

### Worked example — Quick

Lines: Bedroom $650 (room) + Kitchen $450 (room) + Painting crew 2 days ×
$500/day. Costs: materials $120, prep $30, travel $25, other $10, loaded
day cost $320, target margin 30%:

- Price: $650 + $450 + $1,000 = **$2,100.00**
- Labor cost: 2 × $320 = **$640.00** · Total cost: $640 + $120 + $30 + $25 + $10 = **$825.00**
- Profit: **$1,275.00** · True margin: **60.7%** — above target, no warning
- Minimum price at target: $825 ÷ 0.70 = **$1,178.57**
- Raise the target to 70% and the warning fires: minimum price becomes
  $825 ÷ 0.30 = **$2,750.00** (the price itself never auto-changes).

## Data & privacy

Everything lives only in the visitor's own browser (`localStorage`):

| Key | Contents |
|---|---|
| `coatestimate.estimates.v1` | Saved estimates (all fields + areas + quick lines + mode, name, status, date). |
| `coatestimate.settings.v1` | Company name/phone/email, logo (data URL), paint + primer prices, coverages, production rates, crew rate, loaded day cost. |
| `coatestimate.pro.v1` | Pro unlock flag (see MVP note above). |
| `coatestimate.freepdf.v1` | First-free-PDF counter (0 or 1). |

**Settings persistence (verified):** production rates, coverages, and
paint/primer prices are setup fields — every change is written to
`coatestimate.settings.v1` on the next render and re-applied on load, so
they persist across sessions. The test suite scrambles the fields and
re-applies settings to prove the restore.

Nothing is sent to a server. The app tells users plainly: "Your data is
stored only in this browser on this device — export a backup regularly."
Backup export downloads a JSON file with estimates + settings; import
restores it (files from other apps are rejected). **Backups saved before
Quick mode existed have no mode field or quick lines — they import and
load as Measured estimates.** CSV export lists estimates with their
computed totals.

Estimate statuses: Draft, Sent, Accepted, Declined, Completed. Dashboard
totals: pipeline = Draft + Sent estimate prices; accepted work = Accepted +
Completed estimate prices. Declined estimates count in neither.

## Deploy

Any static host works — copy the three files (or at least the two HTML
files) to the host as-is:

- **Netlify / Vercel:** drag-and-drop the folder, or connect the repo; no
  build command, publish directory = this folder.
- **Cloudflare Pages:** same — framework preset "None", no build command.
- **GitHub Pages:** fine for demos/preview only — GitHub Pages terms
  prohibit primarily commercial SaaS use, so don't run the paid product
  from there.

The only external request the app makes is the jsPDF CDN script in
`app.html` (cdnjs). If it fails to load, everything except the PDF button
still works and the user gets a clear message — and a failed load never
consumes the free PDF.
