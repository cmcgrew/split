---
name: split-check
description: Split an itemized receipt into a link the group scans to pick their items and pay by Venmo. Use for split the bill, split the check, itemize a receipt, or who owes what.
---

# Split a bill

Works with any itemized receipt — a restaurant check, a grocery run, a hardware
store order, a group takeout order, a shared online cart. Anything with line
items and a total.

Read the receipt, confirm the numbers with the user, then emit one URL. The
page that renders it is already published — never generate a new HTML file.

## Configuration

Read `config.json`. It has `baseUrl`, `venmo`, and optional `payeeName`.

**If `baseUrl` contains `REPLACE-ME`:** stop. Tell the user to publish the page
(Netlify Drop or GitHub Pages, see README) and put the live URL in
`config.json`. Nothing else works until this is done. This is once, ever.

**If `venmo` contains `REPLACE-ME`:** ask for their Venmo *username* — the
handle under their photo in the app, starting with `@` — not their display name.
Then tell them to paste it into `config.json` so they are never asked again.
You can still build this bill by passing `--venmo=` on the command line.

`encode.js` cleans up whatever they give you: it strips `@`, strips
`venmo.com/u/` prefixes, trims whitespace, and rejects anything that can't be a
real username. If it rejects the input, relay the message — it explains the fix.
Never guess a handle, and never fall back to one from an earlier conversation.

### Someone else paid

The person running this Skill is not always the person who paid. If the user
says a friend paid, override for that bill only — do not edit `config.json`:

```bash
node encode.js bill.json --base="<baseUrl>" --venmo=Sarah-Lee-99 --payee-name=Sarah
```

`--payee-name` is a first name. It puts "Pay Sarah $12.40" on the button, which
is how someone notices the money is heading to the wrong person. Set it whenever
the payee isn't the config default.

## Step 1 — Read the receipt

Extract every line item. For each: name, unit price, quantity.

Rules that matter:

- **Unit price, not line total.** A receipt showing `2 GARLIC BREAD 12.00`
  means `price: 6.00, qty: 2`. Getting this backwards doubles someone's bill.
- **Expand abbreviations** into what the group would recognize.
  `MRG PIZZA` → `Margherita Pizza`. `SD SPKL WTR` → `Sparkling Water`.
  `2X4 STUD 8FT` → `2x4 Stud 8ft`.
- **Keep names under ~24 characters.** Long names inflate the URL and make the
  QR code dense. Trim descriptors, keep the thing itself.
- **Mark shared items** with `"shared": true` — anything the group splits rather
  than one person claiming: appetizers, a bottle of wine, a bag of coffee for
  the house, a tool everyone chipped in on. This does not change what anyone
  sees on the page — every item already has the fraction buttons — but it shows
  up in the encoder's printout, so it is worth setting when you can tell.
- **Do not include tax, tip, fees, service charges, or discounts as items.**
  They go in their own fields.
- **Never merge items whose price differs, even if the name is identical.**
  Group into one row with `qty > 1` only when both the name *and* the final
  price (base + any paid add-ons) match exactly. Two salads priced the same
  merge into `qty: 2`; two salads where only one has feta are two separate
  `qty: 1` rows, because they're no longer interchangeable — an add-on travels
  with the person who ordered it, not with a diluted average price.

Record `statedSubtotal` and `statedTotal` exactly as printed. These are what the
validator checks against — never back-fill them from your own arithmetic, or the
check becomes meaningless.

## Step 1a — Add-ons and fees

Every charge that isn't tax or tip attaches to one item, to every unit of a
line, or to the whole bill. **What it attaches to decides where it goes** — and
in two of the three cases it must carry a visible label, or the price on the
page looks like an unexplained markup.

### A charge on ONE item → fold it into that item as an add-on

If a modifier or surcharge line has its own dollar amount and belongs to a
single line item — `Add Feta Cheese $0.95`, `Extra Shot $0.80`,
`Gift wrap $3.00` on one gift, `Engraving $12.00` on one pen — add that amount
to the item's price and set `"addons"` to a label with the cost in it:

```json
{ "name": "Chicken Wrap", "price": 9.44, "qty": 1, "addons": "+ Feta Cheese ($0.95)" }
```

The person sees what they're being charged for without doing the subtraction
themselves, and the charge follows the person who ordered it.

**Ignore modifiers with no charge** — `No Tomatoes`, `Ranch Dressing`,
`Make it a Wrap`. They don't change who owes what, and listing them is noise.

### A charge on EVERY UNIT of a line → fold in per unit, and still label it

Some charges apply once per unit rather than once per bill or once per single
item: a per-ticket processing fee, a per-passenger booking fee, a per-seat
convenience charge, a per-night resort fee. A receipt showing five movie
tickets with a $2.09 processing fee on each is this case.

Handle it exactly like a single-item add-on — fold the per-unit amount into that
line's price and attach an `addons` label — the only difference being that the
label applies to every unit in the `qty`, not to one:

```json
{ "name": "Adult Ticket", "price": 17.59, "qty": 3, "addons": "+ Processing Fee ($2.09)" }
```

Here the printed ticket price was $15.50 and the fee $2.09. The label is what
tells someone why they're being asked for $17.59.

**Do not put these in `fees`.** Fees are prorated across the whole subtotal, so
a per-ticket fee dumped there gets charged partly to people who bought no
ticket. Folding it into the unit price keeps it with the units it belongs to.

**Do not fold it in silently.** The arithmetic is right either way, so nothing
will fail validation — but a bare $17.59 next to a receipt that says $15.50
reads as a markup nobody explained. This is the failure mode the `addons` field
exists to prevent.

If the same fee repeats across lines priced differently — adult and senior
tickets, say — each line gets its own row with its own price and its own copy of
the label. Do not merge them.

### A charge on the WHOLE bill → put it in `fees`

Delivery, service charge, small-order fee, bag fee, resort fee, booking fee,
environmental or recycling levy, fuel surcharge, credit-card surcharge — any
charge the whole group incurred rather than one person.

Add them all under `fees`, either as one number or as a labelled list:

```json
"fees": [
  { "name": "Delivery", "amount": 5.00 },
  { "name": "Service charge", "amount": 3.50 }
]
```

The encoder sums them and puts the total in the link. The page prorates that
total the same way it prorates tax — by each person's share of the subtotal —
and shows it as a **Fees** line in the summary at the bottom of the screen.
On an even split, fees simply fold into the amount being divided.

The labels are for the confirmation summary in Step 4 and the encoder's
printout; only the total travels in the link, so long fee names cost nothing.

**The name of a charge does not decide the bucket — its scoping does.** The same
words appear in both lists on purpose. A flat $25 resort fee on a hotel folio is
a whole-bill fee; the same phrase billed per night against a 3-night line is the
per-unit case above. Read what it is charged against, not what it is called.

**One judgement call to get right:** a service charge that is really an
automatic gratuity belongs in `tip`, not `fees` — see Step 2. If a receipt
prints something like `Service charge (18%)` in the tip position, treat it as
tip. If it's a flat operational fee, it's a fee.

## Step 1b — Splitting evenly

If the user says the group is splitting the whole bill evenly rather than by
item — "just split it 8 ways", "everyone pays the same", "divide it evenly" —
add `"splitEvenly": 8` to the bill JSON. Count **everyone sharing the cost,
including the person who paid.** Eight people is `8`, not `7`.

This changes the guest page: items become read-only, and each person sees one
share and a Pay button with nothing to select.

**Ask for the headcount if it isn't stated.** Never infer it from the number of
items or guess from a party size printed on the receipt.

The share is rounded up to the cent so the payer is never left short; the
encoder reports how much extra that collects across the group.

Splitting evenly is a decision the whole group makes together, so it belongs in
the link. There is deliberately no guest-side toggle — if everyone could each
choose, their shares would not sum to the bill. If the group changes its mind,
generate a new link.

## Step 2 — Tip

Report only what's on the receipt. Never ask, and never assume a percentage.

- Tip printed on the receipt → use that exact number.
- Auto-gratuity or a percentage service charge → use that number, put it in
  `tip`, and note in the summary in Step 4 that it's already included, so nobody
  double-tips.
- No tip anywhere on the receipt → `tip: 0`. Don't default to 20%, don't guess,
  don't ask what they want to leave. Plenty of receipts this Skill handles —
  groceries, hardware, retail — have no tip line at all, and that's normal.

This mirrors the rest of the Skill: transcribe what's actually on the receipt,
show it in the Step 4 summary, and let the user correct anything that's wrong
or missing. Tip is not a special case that gets its own question — it's one
more line in the same confirm-or-fix summary as everything else.

## Step 3 — Build and validate

Write the bill to a JSON file in this shape:

```json
{
  "merchant": "Northside Hardware",
  "tax": 8.00,
  "tip": 0,
  "fees": [
    { "name": "Delivery", "amount": 5.00 },
    { "name": "Recycling fee", "amount": 2.50 }
  ],
  "statedSubtotal": 100.00,
  "statedTotal": 115.50,
  "items": [
    { "name": "Cordless Drill", "price": 60.00, "qty": 1 },
    { "name": "Drill Bit Set", "price": 20.00, "qty": 2, "shared": true }
  ]
}
```

`merchant` is the name shown at the top of the page — the restaurant, store, or
service. Leave it out if the receipt doesn't name one. (`restaurant` is still
accepted as an older spelling of the same field.) Omit `fees` entirely when
there are none; the **Fees** line then doesn't appear at all.

Then run:

```bash
node encode.js bill.json
```

`encode.js` reads `config.json` itself for `baseUrl`, `venmo`, and `payeeName`.
Do not pass `--base`. Omit `venmo` from the bill JSON entirely — leaving it out
is what makes it inherit from config. Only add `--venmo=` / `--payee-name=` when
someone other than the config owner paid.

It prints the parsed bill, flags arithmetic that doesn't reconcile, and prints
the URL.

**If it reports a mismatch, do not hand over the link.** Show the user what's
off and where you think the missing line is. A mismatch on the total when the
subtotal reconciles almost always means a fee was missed — check the receipt
for a delivery or service line before assuming an item is wrong. A silently
dropped charge means the payer eats the difference.

This run is for validation only — use its output to build the confirmation
summary in Step 4. Don't hand anything to the user yet.

## Step 4 — Confirm before sending

Show the parsed bill — items, quantities, prices, add-ons, tax, tip, each fee,
total — exactly as read from the receipt. Receipt OCR misreads `0` as `O` and
drops decimal points, and a tip of `0` because none was found looks identical to
a tip that was missed — this is the only place either gets caught.

Then use `ask_user_input_v0` with exactly two options: **"Looks Good"** and
**"Type changes below…"**. Don't ask an open question about the tip, the fees,
or anything else — everything they'd want to correct is already visible in the
summary above, and a wrong number is easier to spot than to compose a sentence
about.

The second option is a label, not a question. It exists so the buttons aren't
the only visible way forward — it points at the text box, which is where a
correction actually gets typed. Never follow it with "what would you like to
change?"

- **"Looks Good"** — proceed. Do not ask again or re-summarize.
- **"Type changes below…"** — say nothing more than a short acknowledgement
  and wait. They are about to type. Do not re-show the summary yet.
- **They type a correction** (with or without tapping either button) — apply it,
  re-show the summary, and offer the same two options again. Loop until they
  pick "Looks Good."

### Handing over the link

**Never paste the URL fragment into your chat reply as text.** It's a
150–250+ character opaque string, and generating it as prose — even when
copying from a tool result you saw a moment ago — carries a real risk of
silently swapping a character. This has broken real links twice. The fix is
structural, not "be more careful":

```bash
node encode.js bill.json --out=/mnt/user-data/outputs/link.md
```

This writes both finished URLs straight from memory to disk — the same
strings `encode.js` validated, never retyped — as a small file with two
clearly labeled, tappable links: **HOST LINK** (opens the QR code and
payment tracker — this is theirs to open) and **GUEST LINK** (what a scanned
QR resolves to; only worth sending separately if they're not using the QR).
Use `present_files` to hand that file to the user. `--out=` also accepts an
`.html` path if a styled clickable card is ever preferred instead. **Do not
additionally type either raw URL anywhere in your reply.** The file is the
only copy of either fragment that should exist in the conversation — typing
the fragment directly into your chat reply reintroduces the exact
transcription risk this exists to remove, since that risk comes from the
string passing through your own text generation at all, not from where you
copied it from.

If the user explicitly asks to see the raw link as text — for pasting
somewhere the file doesn't help — copy it from the `host url:` line in the
tool's own stdout, character for character, never reconstructed.

## Notes

- The fragment holds the whole bill, so the URL is long. That's expected — never
  try to shorten it by dropping items.
- If `encode.js` warns the URL is over 1800 characters, shorten item names and
  re-run rather than shipping a QR nobody can scan.
- Never generate a new page file. It is published once and reused forever.
- The published page contains no personal data. Several people can safely share
  one published copy — the payee travels in the link, not in the file. If two
  people in a group both use this Skill, they do not need two published pages.
