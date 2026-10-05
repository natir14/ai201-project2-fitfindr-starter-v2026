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

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

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

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

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
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



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
