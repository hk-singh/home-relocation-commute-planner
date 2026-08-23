# The Aldgate Planner — how it works and what building it taught me

## What this is

A new job lands in Aldgate. One person in the household commutes to it; the other works
remotely, so that single journey is the only one that binds — and the ceiling is 40–45 minutes
door to door, not more. There's a car that needs somewhere to live. And ideally you end up
somewhere you can actually *walk around* — parks, a high street, a coffee you didn't have to
drive to.

That's four constraints pulling in different directions, and the honest problem is that they
trade off against each other in ways no property portal will tell you. Rightmove will happily
show you a beautiful flat in Stoke Newington and never mention that you'll spend the next three
years circling for a parking space. So this is a small tool that holds all four constraints at
once and lets you watch them fight.

Open `index.html` in any browser. That's it — no server, no build, no install.

---

## The one insight that made the whole thing worth building

I started out assuming this was a "how far is it to Liverpool Street" problem. It isn't.

The office is at **Portsoken House, 155–157 Minories, EC3N 1LJ**, on the eastern edge of the City. Look at where that actually
sits and something interesting falls out — it's within a short walk of *six* different stations,
and the walks are wildly different lengths:

| You arrive at | Walk to the desk |
|---|---|
| Aldgate | 3 min |
| Tower Gateway (DLR) | 4 min |
| Fenchurch Street | 5 min |
| Aldgate East | 6 min |
| Tower Hill | 6 min |
| Liverpool Street | 9 min |

Six minutes doesn't sound like much. It's twenty hours a year. And more importantly, it
*re-ranks the entire map*. Consider:

- **Upminster** is at the far eastern edge of the London map — it looks absurdly far. But the c2c
  runs it into **Fenchurch Street in 22 minutes**, and Fenchurch Street is a five-minute walk. Total:
  39 minutes. It beats places that look twice as close on a tube map.
- The **Metropolitan line literally terminates at Aldgate**. No change, no interchange, three-minute
  walk. That's why Wembley Park — which feels like a different city — comes in at 42 minutes.
- Meanwhile anything routing via Liverpool Street pays a nine-minute tax at the end of every
  single journey.

This is the sort of thing you only find by picking one specific address and being pedantic about
it. "Near Aldgate" would have hidden it completely. **Lesson: specificity is where the insight
lives.** A model built on a vague anchor gives you vague answers that feel right and are wrong.

---

## How the commute number is actually built

Every journey is five costs, not one:

```
door-to-door = walk to station
             + platform wait
             + time on the train
             + interchange penalty
             + walk from the arrival station to the office
```

Four of those five are invisible on a train timetable. This is precisely why "22 minutes to
London" is a lie people repeat to themselves for years.

A few decisions worth explaining:

**Platform wait is half the headway, capped at eight minutes.** If trains come every four minutes
you turn up and go; if they come every fifteen you check the app and time your walk. Modelling it
as half the gap is crude but it's directionally right, and it's the thing that correctly punishes
Chingford and Highams Park (a train every twelve minutes) relative to the Elizabeth line (every
four). Without this term, Chingford looks better than it is.

**The walk to the home station is applied identically to every area.** This one is deliberate and
slightly counter-intuitive. It would be "more accurate" to give each area its own typical walk.
But then you'd be comparing two things at once and couldn't tell which was moving. Holding it
constant makes the comparison *clean*: every difference you see between two areas is a difference
in the trains, not in my guess about where you'd buy. The slider then lets you ask the real
question — "what if we bought within five minutes of the station?" — and watch the whole board
re-rank at once.

**The bad-day figure is `15% + 4 minutes`.** It's a rule of thumb, not a modelled distribution,
and it's labelled as one. Being honest about the precision of your numbers is more useful than
inventing false precision.

---

## The architecture, such as it is

One file. 76KB. No dependencies, no build step, no framework.

```
index.html
├── <style>           design tokens, three theme states, component styles
├── <script> #1       LINES, TERMINI, AREAS — the data
├── <script> #2       state, the commute model, scoring, small render helpers
├── <script> #3       the console, the result cards, the shortlist table
└── <script> #4       the polar time map, the method essay, tab wiring
```

**Why one file with no framework?** Because the constraints pointed there. It needed to be
publishable as an Artifact (which runs under a strict CSP that blocks external scripts), viewable
on a phone while standing outside a house you're considering, readable as a single reviewable diff
in git, and still working in five years when whatever build tool I'd have picked has bit-rotted.
React would have bought me nothing here — there are four views and one state object. `innerHTML`
and a `renderAll()` is a perfectly good reactive system when the whole app fits in one head.

**Splitting one script into four `<script>` tags** is a small trick worth knowing: top-level
`const` and `let` in classic scripts share the same global lexical scope, so they can see each
other's declarations. It let me write the file in four appends without any module plumbing.

**State lives in one object and persists to `localStorage`**, wrapped in try/catch because private
browsing windows throw on access rather than returning null. Every mutation goes
`state.x = v; save(); renderAll();`. It's the simplest thing that works, and "simplest thing that
works" is a real engineering position, not a cop-out.

---

## The two pieces of design I'm actually pleased with

### The journey strip

Every area's commute is drawn as a horizontal bar broken into its five parts — hatched for
walking, flat grey for the platform, and the **real TfL line colour** for time on the train.

The trick is that **every strip is drawn on the same 0–60 minute scale**, with a dashed magenta
line at the 45-minute limit. So you never have to read a number to compare two areas. You just
see which bars cross the line. Encoding the same fact twice — once as a number for precision, once
as a position for instant comparison — is most of what good information design is.

The line colours aren't decoration either. They tell you which train you'd actually be on, which
matters when you're thinking about whether one line going down strands you.

### The time map

A polar plot where the **angle is the true compass bearing** from the office but the **radius is
commute minutes, not miles**. Concentric rings at 15/25/35/45/55 minutes, with the target ring
picked out in magenta.

What it shows immediately: London is radically asymmetric from Aldgate. The north-east — the
Central line and the Essex corridors — is dense with options inside the ring. The south-west is
empty. That's not a rendering artifact, that's the actual shape of the rail network from that
address, and it means the search area is really "east and north-east London" whether you like it
or not.

---

## Bugs I hit, and what they taught me

### 1. The one that ate the file (Python `str.replace` with an empty needle)

I was patching the file with a Python script and wrote this:

```python
old = s[s.index(START) : s.index(END)]     # find the block to replace
s = s.replace(old, new)
```

`s.index(END)` searches **from the beginning of the string**. `END` was a common pattern that
occurred earlier in the file, so it returned an index *before* `START`. A backwards slice in
Python doesn't error — it quietly returns `''`.

And `s.replace('', new)` doesn't do nothing. It inserts `new` **between every single character**.
A 76KB file became 108MB.

Three lessons, in order of importance:

1. **When you search for the end of a range, search from the start of the range.**
   `s.index(END, i)` — that one extra argument was the whole bug.
2. **Know your language's quiet failures.** Python has a family of these: backwards slices return
   empty, `replace('')` explodes, `"".join()` on a string iterates characters. None of them raise.
   An error is a gift; silence is the expensive case.
3. **The recovery was only possible because the corruption was deterministic.** `replace('', x)`
   produces `x + s[0] + x + s[1] + ...` — a perfect period of `len(x)+1`. So I found the period by
   searching for the second occurrence of the first 200 characters, and pulled the original back
   out with `corrupted[L::L+1]`. Every character recovered, verified against known landmarks in
   the file.

   The general habit that made this recoverable: **understand the mechanism of your failure before
   you panic**. I nearly rewrote the file from scratch. Five minutes of thinking about *what
   exactly* `replace('', x)` does turned a rewrite into a one-line fix. Corrupted data is often
   more recoverable than it looks, because corruption usually has structure.

### 2. The headless browser that hung forever

Screenshots suddenly started timing out on `page.goto(...)` — even with `waitUntil:
'domcontentloaded'`. The cause: the page links Google Fonts, and a blocking `<link
rel="stylesheet">` that never resolves will hold up DOMContentLoaded indefinitely. The sandbox's
network proxy had gone from *rejecting* the request fast to *hanging* on it.

The fix was to test against a copy with the font links stripped, rather than to change the real
file. **Lesson: don't let your test harness's environment leak into your deliverable.** The
instinct to "just remove the fonts so the test passes" would have quietly degraded the actual
product to fix a problem the product didn't have.

### 3. Dead code that survived a refactor

While rewriting the map label placement I left a line in that computed `L.lx` with a garbled
ternary, immediately overwritten by the next line. Harmless, and syntactically valid, so nothing
caught it. It only died because I re-read my own diff before committing.

**Lesson: read your diff as if someone else wrote it and you're looking for the mistake.** Every
tool in your chain — the parser, the linter, the tests — is checking whether the code *runs*. Only
you are checking whether it *makes sense*.

### 4. Labels that piled on top of each other

The first version of the time map placed each label just outside its dot. Ten of the areas sit
between bearings 68° and 82° — the whole Essex/Elizabeth corridor — so the labels landed in a
heap and the map was unreadable.

The fix is the trick a cartographer does by hand: place labels where they want to go, then
repeatedly nudge apart any pair whose bounding boxes overlap, and draw a leader line to any label
that had to move far. Ninety iterations of an O(n²) loop over fourteen labels is nothing —
about a thousand comparisons. **Lesson: "too slow" is a claim you should check, not assume.
Brute force is usually fine at small n, and it's always easier to read than the clever version.**

---

## Things I chose not to build, and why

Naming what you deliberately left out is part of the deliverable. It's the difference between a
gap and an oversight.

- **No live TfL API.** Journey times come from scheduled services and typical peak frequencies.
  A live API would make the numbers look more authoritative without making them more true — you'd
  be getting today's disruption baked into a decision about the next five years. What the page
  says instead, prominently, is: go do the journey once at 8am on a Tuesday.
- **No street-level data.** Parking is scored per *area*, but it genuinely varies street by street.
  The tool says so rather than pretending otherwise.
- **No schools, no crime stats.** Out of scope for what was asked, and both deserve better than a
  1–5 score.
- **Nothing beyond 60 minutes.** Leigh-on-Sea and St Albans are lovely. Neither is close.

---

## What it says, if you just want the answer

Two answers, depending on whether the car is truly non-negotiable.

**If parking could flex:** Forest Gate (29 min), Stratford (26) and Walthamstow Central (36) win
outright — all three pair a short commute with real high streets and serious green space. And Mile
End at 21 minutes beats everything on every axis except parking, where it's genuinely bad.

**If the car is fixed:** Wanstead (39 min) is the best all-rounder on the list. Upminster (39) is
the same commute for about £300/month less and an actual driveway, bought at the cost of being far
out. Highams Park (44) sits between them and looks underpriced for what it offers.

Worth a real test run before ruling in or out: Chingford (49), Gidea Park (46), South Woodford
(43, with the easiest parking of any Central line stop).
