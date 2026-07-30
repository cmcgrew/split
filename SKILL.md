---
name: split-check
description: Split a restaurant receipt photo into a link friends scan to pick their items and pay by Venmo. Use for split the check, split the bill, itemize a receipt, or who owes what for dinner.
---

# Split a restaurant check

Read the receipt, confirm the numbers with the user, then emit one URL. The
page that renders it is already published — never generate a new HTML file.

## Configuration

Read `config.json`. It has `baseUrl`, `venmo`, and optional `payeeName`.

**If `baseUrl` contains `REPLACE-ME`:** stop. Tell the user to publish
`split.html` (Netlify Drop or GitHub Pages, see README) and put the live URL in
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

The person running this Skill is not always the person who picked up the tab.
If the user says a friend paid, override for that bill only — do not edit
`config.json`:

```bash
node encode.js bill.json --base="<baseUrl>" --venmo=Sarah-Lee-99 --payee-name=Sarah
```

`--payee-name` is a first name. It puts "Pay Sarah $12.40" on the button, which
is how a guest notices the money is heading to the wrong person. Set it whenever
the payee isn't the config default.

## Step 1 — Read the receipt

Extract every line item. For each: name, unit price, quantity.

Rules that matter:

- **Unit price, not line total.** A receipt showing `2 GARLIC BREAD 12.00`
  means `price: 6.00, qty: 2`. Getting this backwards doubles someone's bill.
- **Expand abbreviations** into what a friend would recognize.
  `MRG PIZZA` → `Margherita Pizza`. `SD SPKL WTR` → `Sparkling Water`.
- **Keep names under ~24 characters.** Long names inflate the URL and make the
  QR code dense. Trim descriptors, keep the dish.
- **Mark shared plates** with `"shared": true` — appetizers, bottles of wine,
  dessert to split, sides for the table. The page labels these "shared plate" so
  guests know to use the fraction buttons rather than claiming the whole thing.
- **Do not include tax, tip, service charge, or discounts as items.** They go in
  their own fields.
- **Paid add-ons fold into the item's price and get a label with its cost.**
  If a modifier line has its own dollar amount — `Add Feta Cheese$ 0.95` —
  add that amount to the item's price and set
  `"addons": "+ Feta Cheese ($0.95)"`, so the guest can see what they're
  being charged for without doing the subtraction themselves. **Ignore
  modifiers with no charge** — `No Tomatoes`, `Ranch Dressing`,
  `Make it a Wrap` — they don't change who owes what, and listing them is
  noise.
- **Never merge items whose price differs, even if the name is identical.**
  Group into one row with `qty > 1` only when both the name *and* the final
  price (base + any paid add-ons) match exactly. Two salads that are priced
  the same merge into `qty: 2`; two salads where only one has feta are two
  separate `qty: 1` rows, because they're no longer interchangeable — an add-on
  travels with the person who ordered it, not with a diluted average price.

Record `statedSubtotal` and `statedTotal` exactly as printed. These are what the
validator checks against — never back-fill them from your own arithmetic, or the
check becomes meaningless.

## Step 1b — Splitting evenly

If the user says the table is splitting the whole bill evenly rather than by
item — "just split it 8 ways", "everyone pays the same", "divide it evenly" —
add `"splitEvenly": 8` to the bill JSON. Count **everyone sharing the cost,
including the person who paid.** Eight people at dinner is `8`, not `7`.

This changes the guest page: items become read-only, and each guest sees one
share and a Pay button with nothing to select.

**Ask for the headcount if it isn't stated.** Never infer it from the number of
items or guess from the party size on the receipt.

The share is rounded up to the cent so the host is never left short; the encoder
reports how much extra that collects across the table.

Splitting evenly is a decision the whole table makes together, so it belongs in
the link. There is deliberately no guest-side toggle — if guests could each
choose, their shares would not sum to the bill. If the table changes its mind,
generate a new link.

## Step 2 — Tip

Don't ask. If the receipt shows a tip or an auto-gratuity/service charge, use
it — and if it's auto-gratuity, put it in `tip` and note it in the summary in
Step 4 so nobody double-tips.

If the receipt shows no tip, assume **20% of the pre-tax subtotal** and carry
that number forward. The confirmation step is where the user sees this number
and can change it — that's the only tip conversation that needs to happen.

## Step 3 — Build and validate

Write the bill to a JSON file in this shape:

```json
{
  "restaurant": "Luca's Trattoria",
  "venmo": "Christopher-McGrew",
  "tax": 8.60,
  "tip": 20.00,
  "statedSubtotal": 101.00,
  "statedTotal": 129.60,
  "items": [
    { "name": "Caesar Salad", "price": 12.00, "qty": 1 },
    { "name": "Margherita Pizza", "price": 18.00, "qty": 1, "shared": true },
    { "name": "Garlic Bread", "price": 6.00, "qty": 2 },
    { "name": "House Red Glass", "price": 14.00, "qty": 3 }
  ]
}
```

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
off and where you think the missing line is. A silently dropped item means the
host eats the difference.

This run is for validation only — use its output to build the confirmation
summary in Step 4. Don't hand anything to the user yet.

## Step 4 — Confirm before sending

Show the parsed table — items, quantities, prices, tax, the **assumed** tip,
total. Receipt OCR misreads `0` as `O` and drops decimal points, and the
assumed tip is a guess, not a fact — this is the only place either gets
caught.

Then use `ask_user_input_v0` with two options: **"Looks good"** and **"Needs
changes."** Don't ask an open question about the tip or anything else —
everything they'd want to correct is already visible in the summary above,
and a wrong number is easier to spot than to compose a sentence about.

- **"Needs changes"** — ask what to fix, in plain text. Apply it, re-show the
  summary, and offer the same two-button choice again. Loop until they pick
  "Looks good."
- **"Looks good"** — proceed. Do not ask again or re-summarize.

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
Use `present_files` to hand that file to the user. **Do not additionally type
either raw URL anywhere in your reply.** The file is the only copy of either
fragment that should exist in the conversation.

If the user explicitly asks to see the raw link as text — for pasting
somewhere the file doesn't help — copy it from the `host url:` line in the
tool's own stdout, character for character, never reconstructed.

## Notes

- The fragment holds the whole bill, so the URL is long. That's expected — never
  try to shorten it by dropping items.
- If `encode.js` warns the URL is over 1800 characters, shorten item names and
  re-run rather than shipping a QR nobody can scan.
- Never generate a new `split.html`. It is published once and reused forever.
- `split.html` contains no personal data. Several people can safely share one
  published copy — the payee travels in the link, not in the file. If two people
  in a group both use this Skill, they do not need two published pages.
