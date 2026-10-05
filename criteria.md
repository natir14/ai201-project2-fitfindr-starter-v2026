# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:**
The search half of this path is deterministic — the string-split parser and
`search_listings` give the same result for the same query every time — but
`suggest_outfit` and `create_fit_card` both call the model through
`generate()`, and `run_agent` doesn't catch model errors yet. If the service
rate-limits past `MAX_RETRIES`, `generate()` raises and the run ends with no fit
card. One miss in five leaves room for that; more than one means something in
my code is wrong, not the service.

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**
Nothing on this path calls the model. Parsing is string splitting,
`search_listings` is plain filtering and keyword counting, and the branch is
`if not results:` in `run_agent`, so the same query takes the same path every
time. The error message is built from `session["parsed"]`, not generated, so
it always names the description, size and price that were tried. Anything less
than 5 of 5 would mean the branch itself is broken.

---

## 3. The item search found is the item the fit card is about

For a matching query, `session["selected_item"]["id"]` equals
`session["search_results"][0]["id"]`, and the fit card states the selected
item's price (e.g. `$18` or `$18.00` for the Y2K Baby Tee) — in 5 of 5 tries.

**Why this target:**
The price only reaches `create_fit_card` through `session["selected_item"]` —
the outfit text from `suggest_outfit` doesn't include it — so a correct price in
the caption shows the item travelled search → session → third tool intact. If
the wrong item were passed along, the price would be wrong or missing. The
first half is pure code and should never fail; the second half depends on the
model following the prompt's "mention the price" rule, but the price is in the
prompt word for word, so I'm holding it to 5 of 5.

---

## 4. The fit card reads like a post from the buyer

Running the same matching query 5 times with the cache off
(`AI201_CACHE=0`), the fit card is two to four sentences long and does not
describe the item as being sold by the poster (no "listed", "selling",
"for sale" or "DM me") — in at least 4 of 5 tries.

**Why this target:**
Both rules are instructions in the `create_fit_card` prompt, not checks in
code, and `TEMPERATURE` is 0.9, so the wording changes every run and the model
can drift. The prompt says "a real person posting their find" but never says
the poster bought it, so the model has room to read it as a resale listing.
4 of 5 allows for one drift; a caption that reads like an ad more often than
that isn't doing its job.

---

## 5. Search respects the size and price the user asked for

For the query `"top size S under $20"`, every listing in
`session["search_results"]` has a price of $20.00 or less and has `S` as a
whole size token (e.g. `S` or `S/M`, never `US 9` or `XS`) — 5 of 5 tries.

**Why this target:**
Both filters are plain comparisons in `search_listings`: `price > max_price`
and a whole-token check on the size split on non-alphanumeric characters. No
model is involved, so the same query gives the same results every time. The
size check exists because a substring test lets `"s"` match `"us 9"` and return
shoes for a small top. One wrong item in the results means the filter is
broken, so 5 of 5.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
