# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- Three or four sentences: what a user asks for, and what they get back. -->

FitFindr helps users find secondhand clothing from a plain-language query like
"vintage graphic tee under $30, size M". The agent parses the query into a
description, size and price ceiling, searches a dataset of thrift listings from
Depop, thredUp and Poshmark through an MCP tool, and picks the best match. It
then suggests one or two outfits that pair the item with pieces from the user's
wardrobe (or general styling ideas if the wardrobe is empty) and writes a short
social-media caption about the find. If nothing matches, the planning loop stops
before the outfit step and tells the user what to change in their search.


---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Loads every listing, drops the ones over `max_price` or not in `size`, scores the rest by how many words of `description` (lowercased, split on spaces) appear in the listing's title + description + style tags, and returns them best score first. Size matching is whole-token and case-insensitive: the listing's size is split on anything that isn't a letter, digit or `.`, so `"M"` matches `"S/M"` and `"M/L"`, but `"S"` does not match `"US 9"` and `"L"` does not match `"XL"`. `"One Size"` only matches a request for `"one"` or `"size"`, not `"M"`.
- **Inputs:** `description` (str), `size` (str or None — None skips the size filter), `max_price` (float or None — inclusive; None skips the price filter)
- **Returns:** a `list[dict]` of at most `config.SEARCH_RESULT_LIMIT` (10) listing dicts, highest keyword score first. Each dict has `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand` (str or None), `platform`.
- **When it has nothing:** returns `[]` — an empty list, not None and not an exception. This is what `run_agent` branches on.

### `suggest_outfit`

- **What it does:** Builds a prompt from the new item (title, category, colors, style tags, description) and the user's wardrobe, and asks the model through `generate()` for one or two outfits that pair the item with pieces the user already owns, named exactly as listed.
- **Inputs:** `new_item` (dict — a listing dict), `wardrobe` (dict with an `"items"` key holding a list of wardrobe item dicts: `name`, `category`, `colors`, and optional `notes`)
- **Returns:** a non-empty `str` — the model's outfit suggestions, naming specific wardrobe pieces.
- **When it has nothing:** if `wardrobe["items"]` is empty or missing, it asks the model for general styling ideas for the item instead and returns that string. If the model sends back a blank response, it returns `"No outfit suggestions came back for <title>. Try again."` It never returns `""`. Model errors from `generate()` are not caught here.

### `create_fit_card`

- **What it does:** Asks the model through `generate()` for a social-media caption about the find, built from the item's title, price, platform, condition, colors, style tags and the outfit text. The prompt asks for 2–4 sentences, plain text with no hashtag list, the item, price and platform mentioned once each, and a specific vibe.
- **Inputs:** `outfit` (str — the output of `suggest_outfit`), `new_item` (dict — a listing dict)
- **Returns:** a `str` caption, two to four sentences. The length and the "once each" rules are requested in the prompt, not checked in code.
- **When it has nothing:** if `outfit` is empty or whitespace-only, it returns `"Can't write a fit card for <title> yet — there's no outfit suggestion to build the caption from."` without calling the model. If the model sends back a blank response, it returns `"No caption came back for <title>. Try again."`

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule:** If `search_listings` returns an empty list, put a message in `session["error"]` that repeats what was searched for (description, size, max price) and tells the user what to change — raise the max price, drop the size, or use broader keywords — then return the session without calling `suggest_outfit` or `create_fit_card`. Otherwise take the first result as `session["selected_item"]` and go to `suggest_outfit`, then `create_fit_card`.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** String splitting, no regex and no model call. The query is split on spaces and trailing `,.!?` are stripped. The word after `size` becomes `size`. A word starting with `$`, or a number right after `under`/`below`, becomes `max_price`. Every other word, lowercased and minus a fixed list of filler words (`looking`, `for`, `a`, `the`, `want`, …), is joined back together as `description`. Example: `"vintage graphic tee under $30, size M"` → `{"description": "vintage graphic tee", "size": "M", "max_price": 30.0}`.

**What moves through the session:** `query` → `parsed` (description, size, max_price) → `search_results` (the full list from `search_listings`) → `selected_item` (`search_results[0]`) → `outfit_suggestion` (from `suggest_outfit`, using `selected_item` and `wardrobe`) → `fit_card` (from `create_fit_card`, using `outfit_suggestion` and `selected_item`). If the search comes back empty, `error` is set and `selected_item`, `outfit_suggestion` and `fit_card` stay `None`. The loop is a `while` that picks its next step from which of these fields is still empty, and calls `trace.check_iterations(count)` on every pass.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask 'vintage graphic tee under $30, size M'

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   Here are two outfit ideas pairing the Y2K Butterfly Baby Tee with pieces already in your wardrobe:

### Outfit 1: Casual Y2K Streetwear
*Pair the baby tee with your baggy dark-wash jeans for a classic early-2000s contrast of fitted top and loose bottoms.*

* **Top:** Y2K Baby Tee — Butterfly Print
* **Bottoms:** Baggy straight-leg jeans, dark wash
* **Outerwear:** Black cropped zip hoodie (worn open)
* **Shoes:** Chunky white sneakers
* **Accessories:** Black crossbody bag

### Outfit 2: Elevated Casual / Cottagecore-meets-Street
*Pair the butterfly graphic and pink/purple tones with your khaki trousers for a softer, earth-toned look.*

* **Top:** Y2K Baby Tee — Butterfly Print
* **Bottoms:** Wide-leg khaki trousers
* **Accessories:** Brown leather belt
* **Outerwear:** Vintage black denim jacket
* **Shoes:** Black combat boots

  Fit card: I am so obsessed with this butterfly print Y2K baby tee that I just listed on Depop for $18. It comes in excellent condition with the cutest pink and purple details, and it fits like a dream. You can style it with baggy dark-wash jeans and a zip hoodie for that ultimate early-2000s streetwear vibe, or dress it down with wide-leg khakis and combat boots for an effortless cottagecore-meets-street look.

0 model calls this session, 2 served from cache
```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags': ['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}, {'id': 'lst_012', 'title': 'Oversized Crewneck Sweatshirt — Vintage Navy', 'description': 'Perfectly faded navy crewneck. Genuinely vintage — not manufactured distressed. Ribbed cuffs and hem. No graphics, clean.', 'category': 'tops', 'style_tags': ['vintage', 'basics', 'oversized', 'classic'], 'size': 'XL (fits oversized)', 'condition': 'good', 'price': 20.0, 'colors': ['navy'], 'brand': None, 'platform': 'thredUp'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}]
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
Here are two outfit suggestions that seamlessly integrate the vintage Levi's 501s into your existing wardrobe, using your exact item names:

### Outfit 1: Effortless & Casual (Great for daytime or running errands)
*   **Bottoms:** Vintage Levi's 501 Jeans — Medium Wash
*   **Tops:** White ribbed tank top
*   **Outerwear:** Vintage black denim jacket
*   **Shoes:** Chunky white sneakers
*   **Accessories:** Black crossbody bag

**Why it works:** The straight-leg cut of the 501s balances nicely with the cropped fit of the black denim jacket for a classic denim-on-denim look. Tucking in the white ribbed tank top gives a clean, fitted silhouette against the structured denim, while the chunky white sneakers and black crossbody bag keep the vibe sporty, modern, and effortless.

---

### Outfit 2: Elevated Streetwear (Great for cooler weather or hanging out)
*   **Bottoms:** Vintage Levi's 501 Jeans — Medium Wash
*   **Tops:** Black cropped zip hoodie
*   **Shoes:** Black combat boots
*   **Accessories:** Brown leather belt, Black crossbody bag

**Why it works:** Pairing the medium-wash 501s with the black combat boots creates a classic, grunge-leaning streetwear aesthetic. Adding the brown leather belt breaks up the denim and ties in nicely with the vintage feel of the jeans. Throwing on the black cropped zip hoodie highlights the high-waisted fit of the 501s and creates a sharp contrast against the chunkiness of the boots.
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
Scored these vintage Levi's 501 jeans in a great medium wash for just $38.00 on Depop. They have that perfectly broken-in indigo look that's impossible to fake. I'm keeping the rest of the outfit super clean by just pairing them with crisp white sneakers for an effortless streetwear vibe.
```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:* Help fixing an `IndentationError` in my `search_listings()`
  draft — indentation only, without rewriting the function.
- *What came back:* Claude fixed the indentation and pointed out that the
  indentation had been hiding a logic bug: the `if word in text:` check sat
  outside the `for word in keywords:` loop, so only the last keyword was ever
  scored, and the check was duplicated, so each match added 2. It also flagged
  that my size filter used a plain substring test, so `size='S'` returned the
  `US 9` sneakers.
- *What I changed:* I had the size filter switched to whole-token matching,
  then in a separate step had the `if` moved back inside the loop, the duplicate
  check removed and my debug `print()`s deleted. I kept my own scoring approach
  and re-ran the test commands — `size='S'` now returns only `S/M` items and an
  impossible search returns `[]`.

**Moment 2**

- *What I asked for:* Help improving `create_fit_card()` after the Before
  evaluation showed criterion 4 at 1/5 — the captions sounded like I was
  selling the item ("I just listed on Depop").
- *What came back:* Claude traced it to the prompt: it gave the model a
  listing-style block (price, platform, condition) and never said the poster
  bought the item. It suggested telling the model the writer bought it.
- *What I changed:* I limited the fix to the prompt — no other code changed —
  reran `python run_eval.py --label after`, and confirmed criterion 4 went from
  1/5 to 5/5 with the other four criteria still at 5/5.

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

Produced by `python run_eval.py --label before` (caching off, temperature 0.9),
written to `results/run_2026-10-11_1317_before.md`.

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. A matching query completes all three tools | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. An impossible query stops before the second tool | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. The item search found is the item the fit card is about | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 4. The fit card reads like a post from the buyer | 4 of 5 | PASS | FAIL | FAIL | FAIL | FAIL | MISSED (1/5) |
| 5. Search respects the size and price the user asked for | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

**Real output from one try**, pasted as text, naming the file and function
that produced it. Criterion 4, try 2 — the fit card is from
`tools.py::create_fit_card`, the trace from `agent.py::run_agent`:

```
- stopped early: no
- selected_item: Y2K Baby Tee — Butterfly Print ($18.0, depop)
- search_results: 10

Fit card:
I am completely obsessed with this butterfly print Y2K baby tee I just listed on Depop for $18. It is giving major early 2000s nostalgia, and the pastel pink and purple colors look so cute paired with baggy dark-wash jeans and a zip hoodie. Grab it before I change my mind and keep it for myself!

Trace:
[1] parse_query
      in:  vintage graphic tee under $30
      out: {'description': 'vintage graphic tee', 'size': None, 'max_price': 30.0}
[2] search_listings (via MCP)
      in:  {'description': 'vintage graphic tee', 'size': None, 'max_price': 30.0}
      out: 10 items: Y2K Baby Tee — Butterfly Print, Graphic Tee — 2003 Tour Bootleg Style, Vintage Band Tee — Faded Grey … +7 more
[3] select_item
      in:  10 search results
      out: Y2K Baby Tee — Butterfly Print ($18.0, depop)
      →    took the first (best-scoring) result
[4] suggest_outfit
      in:  Y2K Baby Tee — Butterfly Print + 10 wardrobe items
      out: Here are two outfit suggestions that pair the Y2K Baby Tee with pieces already in your wardrobe:  **Outfit 1: …
[5] create_fit_card
      in:  outfit (1368 chars) + Y2K Baby Tee — Butterfly Print
      out: I am completely obsessed with this butterfly print Y2K baby tee I just listed on Depop for $18. It is giving m…
```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 | A matching query completes all three tools | 4 of 5 | **PASS** — MET (5/5) | All five tries of `vintage graphic tee under $30` ended with `error` None and a non-empty `fit_card`; the trace shows all five steps each time. No rate-limit failure happened, so the one-miss allowance wasn't needed. |
| 2 | An impossible query stops before the second tool | 5 of 5 | **PASS** — MET (5/5) | All five tries of `designer ballgown size XXS under $5` show `search_listings (via MCP)` returning `[] (empty)`, then the `stop` step; `outfit_suggestion` and `fit_card` stayed None, and the error named the description, size and price and said to raise the price, drop the size or broaden the keywords. |
| 3 | The item search found is the item the fit card is about | 5 of 5 | **PASS** — MET (5/5) | In all five tries the first title in the search step equals `selected_item` (Y2K Baby Tee — Butterfly Print), and every fit card states its price as `$18` or `$18.00`. |
| 4 | The fit card reads like a post from the buyer | 4 of 5 | **FAIL** — MISSED (1/5) | All five cards were 3 sentences, so length passed every time. Tries 2, 3 and 5 say "I just listed on Depop"; try 4 has none of the listed words but ends "Grab it for your wardrobe before I change my mind and keep it for myself!", which describes the item as being sold, so I scored it FAIL on the criterion's main clause. Only try 1 reads as a buyer. (Counting keywords alone gives 2/5 — still missed.) |
| 5 | Search respects the size and price the user asked for | 5 of 5 | **PASS** — MET (5/5) | All five tries of `top size S under $20` returned exactly one listing, Mesh Long-Sleeve Top — Black, size `S/M`, $15.00 — under the ceiling and with `S` as a whole size token. No `US 9` or `XS` items appeared. |

**Diagnoses**

**Criterion 4 — missed 1/5. Place: the model's output, driven by the prompt in
`tools.py::create_fit_card`.** The tool ran and returned a caption every time;
what went wrong is the voice. The prompt opens "Write a caption for a social
media post about this thrift find", then hands the model a block that looks like
a listing — `Price: $18.00`, `Platform: depop`, `Condition: excellent` — and
only says "sound like a real person posting their find". It never says the
poster *bought* the item. On a resale platform like Depop, a person posting an
item with its price is most often the seller, so at temperature 0.9 the model
regularly picks that reading: "I just listed on Depop", "Grab it before I change
my mind and keep it for myself!".

The pattern across the whole run supports this. The same query
(`vintage graphic tee under $30`, same item) was run 15 times across criteria
1, 3 and 4, and 5 of those 15 captions were seller-framed — 4 landed in
criterion 4's five tries, 1 in criterion 1's, 0 in criterion 3's. That spread is
sampling variance on identical input, not a difference between the scenarios,
so the honest rate for this item is about 1 in 3 — still well above the one
miss in five the target allows. The other two items in the run (the denim
jacket and the mesh top) produced 0 seller-framed captions in 10, so the
ambiguity bites hardest on this item, but nothing in the prompt rules it out
for any item. It's one problem, not four: the prompt leaves the poster's role
open, and the model fills it in.



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

Both traces are try 1 from `results/run_2026-10-11_1334_after.md`, printed by
the `trace.step()` calls in `agent.py::run_agent`.

**Happy path**

```
[1] parse_query
      in:  vintage graphic tee under $30
      out: {'description': 'vintage graphic tee', 'size': None, 'max_price': 30.0}
[2] search_listings (via MCP)
      in:  {'description': 'vintage graphic tee', 'size': None, 'max_price': 30.0}
      out: 10 items: Y2K Baby Tee — Butterfly Print, Graphic Tee — 2003 Tour Bootleg Style, Vintage Band Tee — Faded Grey … +7 more
[3] select_item
      in:  10 search results
      out: Y2K Baby Tee — Butterfly Print ($18.0, depop)
      →    took the first (best-scoring) result
[4] suggest_outfit
      in:  Y2K Baby Tee — Butterfly Print + 10 wardrobe items
      out: Here are two outfit suggestions that pair the Y2K Butterfly Baby Tee with pieces already in your wardrobe:  **…
[5] create_fit_card
      in:  outfit (1165 chars) + Y2K Baby Tee — Butterfly Print
      out: I am obsessed with this Y2K butterfly baby tee I just scored on Depop for only eighteen dollars. The pink and …
```

**Empty search**

```
[1] parse_query
      in:  designer ballgown size XXS under $5
      out: {'description': 'designer ballgown', 'size': 'XXS', 'max_price': 5.0}
[2] search_listings (via MCP)
      in:  {'description': 'designer ballgown', 'size': 'XXS', 'max_price': 5.0}
      out: [] (empty)
[3] stop
      out: Nothing matched 'designer ballgown' in size XXS under $5.00. Try raising your max price, dropping the size, or…
      →    branch: no listings matched, so suggest_outfit and create_fit_card are skipped
```

The empty search is visibly shorter: it stops at step 3, and `suggest_outfit`
and `create_fit_card` never run.

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->

In `mcp_server.py` I registered `search_listings` with
`@mcp.tool(name="search_listings")`, typed inputs (`description: str`,
`size: str | None`, `max_price: float | None`) and a description written for a
caller who can't see my code, including the empty-list case. In
`agent.py::run_agent` I replaced the direct `search_listings(...)` call with
`call_tool("search_listings", {...})`, passing the same parsed values. Nothing
behaved differently: `python mcp_client.py` listed the tool with all three
inputs, and on three queries (7 results, 1 result and 0 results) the MCP call
returned exactly the same list of dicts as the direct call, still a `list`. The
agent's output and branch were unchanged, and the trace now shows the step as
`search_listings (via MCP)`.



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:** One change, to the prompt in `tools.py::create_fit_card`:
it now says who the poster is. The opening line went from "Write a caption for
a social media post about this thrift find" to "I just bought this item
secondhand and I'm posting about my find. Write the caption for my post." The
listing-style labels now point the same way — `Price:` became `Price I paid:`
and `Platform:` became `Bought on:` (same values). And one rule was added: "I
bought this; I'm not selling it. Never say it's listed or for sale, and never
tell people to buy or grab it." The guard, the other rules, the `generate()`
call and the fallback are unchanged, and no other file changed.

**Which failure it was meant to fix:** Criterion 4, missed 1/5. The diagnosis
put it in the model's output, caused by the prompt: it gave the model a block
that reads like a resale listing (price, platform, condition) and never said the
poster bought the item, so at temperature 0.9 the model often wrote as the
seller — "I just listed on Depop", "Grab it before I change my mind and keep it
for myself!".

### Run Log — After

Produced by `python run_eval.py --label after` (caching off, temperature 0.9),
written to `results/run_2026-10-11_1334_after.md`. Scored the same way as the
Before run, including counting "grab it / before I change my mind" as seller
framing for criterion 4.

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. A matching query completes all three tools | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. An impossible query stops before the second tool | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. The item search found is the item the fit card is about | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 4. The fit card reads like a post from the buyer | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. Search respects the size and price the user asked for | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

**Did it help, and how do I know:** Yes. Criterion 4 went from MISSED (1/5) to
MET (5/5). Every caption now uses buyer language — "I just scored on Depop",
"Found this … on Depop for just $18" — and none says listed, selling, for sale,
DM me, or grab it.

Five tries is a small sample, and the Before run showed how much it can swing:
identical input gave 4 seller-framed captions in criterion 4's five tries but
only 1 in criterion 1's five. So I also counted across every fit card in each
run, not just criterion 4's:

| | Before | After |
|---|---|---|
| Seller-framed captions, all 25 fit cards | 5 / 25 | 0 / 25 |
| Seller-framed, the 15 runs of `vintage graphic tee under $30` | 5 / 15 | 0 / 15 |
| Captions outside 2–4 sentences | 0 / 25 | 0 / 25 |

The change didn't break anything else: criteria 1, 2, 3 and 5 stayed at 5/5,
every caption in the After run is still 2–4 sentences, and every criterion 3
caption still states the selected item's $18 price.


---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->

No criterion is still missed after the improvement — all five are MET in the
After run. These are the problems I know about that my five criteria don't
cover:

- **Rate limits and MCP failures still crash the run.** `run_agent` only
  catches `ModelUnavailable`. When the service keeps rate-limiting past
  `MAX_RETRIES`, `generate()` raises a plain `RuntimeError`, and when the MCP
  server can't be reached, `call_tool` raises `MCPError` — neither is caught in
  `agent.py::run_agent`, so both end in a stack trace instead of a message in
  `session["error"]`. Neither happened in the 60 eval runs (6 scenarios × 5 tries, Before and After), which is why no
  criterion caught it. The fix is two more `except` branches that set
  `session["error"]`, each tested by forcing the failure.

- **The query parser only knows a few phrasings.** `run_agent` sets `max_price`
  only from a `$` word or a number right after "under"/"below", and `size` only
  from the word after "size". "graphic tee less than 30" parses to
  `max_price: None` with "less than 30" left in the description, and "jeans in
  medium, max 40 dollars" gets no size and no price. The price ceiling is
  silently dropped — the user gets results, just not filtered the way they
  asked. My scenarios all use the phrasings the parser handles, so the eval
  never saw this. I'd add "less than", "max" and "dollars", and map "small /
  medium / large" to S / M / L.

- **Search ranks by keyword count and ignores category.** For "graphic tee",
  `tools.py::search_listings` returns Low-Rise Cargo Pants because their
  description ends "Great for layering with a long tee." The pants never came
  first in any run, so `selected_item` was always a top, but they're in the
  results list. Weighting title and style-tag matches above description
  matches would push them down.

- **Criterion 4 is fixed in this run, not proven fixed forever.** The prompt
  now rules out seller framing, and 0 of 25 captions had it, but it's still an
  instruction to a model at temperature 0.9, not a check in code. The
  "mention the item, price and platform exactly once" rule isn't measured by
  any criterion either.

I stopped here because the unit asks for one improvement, measured properly,
and adding more changes would have made the Before/After comparison for
criterion 4 harder to trust.

<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
