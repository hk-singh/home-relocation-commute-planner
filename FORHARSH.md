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

### The corollary, which took a good question to surface

The first version of this had 44 areas and a blind spot. Someone asked two questions — *why no
Bexleyheath?* and *why nothing with fast trains?* — and both turned out to have the **same
answer**, which is the terminus table above read upside down.

A fast train only pays off if it arrives somewhere close. Sort the termini by that final walk and
London splits cleanly in two:

- **Liverpool Street (9 min) and Fenchurch Street (5 min)** are close enough that speed survives.
  They serve Essex and the Thames-side towns — and nothing else.
- **Cannon Street is 14 minutes away**, and every Southeastern train out of Bexley and north-west
  Kent runs there. That walk costs about ten minutes against an Essex town at the same rail
  distance. Bexleyheath has three-bed semis with driveways under £500k, a direct train, and Danson
  Park — and lands *fifteen minutes over*, purely because of where its train stops.
- **St Pancras, Paddington, Marylebone, Waterloo** are worse still. Ebbsfleet does 19 minutes on
  High Speed 1 — the fastest rail leg in the whole dataset — and still can't get under 50, because
  it arrives at the wrong end of town.

So the answer to "why so few fast trains" isn't that I forgot them. It's that **this office's
geography only rewards fast trains from one direction.** The one genuine express that earns its
keep is Shenfield: 22 minutes into Liverpool Street, the same as Upminster, from twice the
distance — and it *was* missing, so it got added along with twelve others.

**Two lessons here, and the second is the uncomfortable one:**

1. **A good model doesn't just rank the options — it explains the shape of the answer.** Once the
   terminus table existed, "why not Bexleyheath" stopped being a matter of opinion and became
   arithmetic. That's the difference between a tool that scores things and a tool that
   *teaches you the territory*.
2. **A blind spot in your data looks exactly like an absence of options.** Nothing in the first
   version was wrong. Bexleyheath scored no points because it wasn't there to score, and a
   ranked list gives you no way to tell "we checked and it lost" from "we never looked". The fix
   was to include the losers explicitly, with their numbers and the reason. **If your tool can
   only show you what it considered, it will quietly convince you that's everything there is.**

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

### The geographic map, and a constraint that turned out to be a gift

The natural way to build "show me where these places are" is Leaflet plus OpenStreetMap tiles.
Fifteen lines of code, everyone's seen it.

That was impossible here. Artifacts run under a Content Security Policy that blocks requests to
external hosts — no CDN scripts, no tile servers, no remote images. The map had to be drawn from
scratch or not at all.

So it's drawn from scratch: an equirectangular projection with a cosine correction for London's
latitude (without it, everything stretches east–west by about 38%), a hand-traced Thames, and all
44 areas at their real coordinates. Rail corridors are polylines from the office out through each
line's stations in order of distance — stations along a line are roughly collinear, so this comes
out looking like the actual routes without any route geometry at all. Dots are coloured by
commute verdict rather than by line, so the shape of the answer is visible before you read a
single label.

And the constraint made it *better*. A tile map would have shown you all of London in
photographic detail, most of which is irrelevant. This shows exactly six things — the river for
orientation, the office, the corridors, the areas, their commute verdict, and a scale bar. You
can see in one glance the thing that actually matters: **the network runs north-east from
Aldgate, so the search does too.** The south-west of the map is empty, and that emptiness is
information.

**Lesson: a hard constraint often forces you into a better design than free choice would have.**
The tile map was the lazy answer, and I'd have shipped it if I'd been allowed to.

### The two-map problem

There are now two maps, and it took building both to see why you need both. The geographic one
answers "where would we actually live?" The polar one — bearing is true, but radius is *minutes*
— answers "how far is that in the only unit that matters on a Tuesday morning?" Neither
substitutes for the other, because the whole premise of the project is that **distance and time
have come apart**. Upminster is far away and close. Chingford is close and far away.

So they're a toggle on one tab rather than two separate features. Same data, two projections, and
the difference between them *is* the insight.

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

### 5. The automation that wasn't allowed to automate

The deploy workflow does the obvious thing: build the site, push it to GitHub Pages. It failed
with `Get Pages site failed. Please verify that the repository has Pages enabled`.

The error helpfully names a fix — `actions/configure-pages` takes an `enablement: true` flag that
turns Pages on via the API. So I set it, pushed, and got a *different* failure:
`Create Pages site failed. Error: Resource not accessible by integration`.

The workflow's token simply isn't permitted to create a Pages site. No amount of cleverness in
the YAML changes that. The same wall showed up earlier in the session: creating the repository
itself returned `403 Resource not accessible by integration`, because the credentials in play
were scoped to specific repositories and did not include "make new ones".

Two lessons, and the second is the real one:

1. **An error that suggests a fix is not a promise the fix will work.** `enablement: true` is
   genuinely the right answer — for a token that has the permission. Reading the suggestion and
   trying it was correct; assuming it would work was not.
2. **When you hit a permissions boundary, stop and say so, rather than routing around it.**
   The tempting next moves — hunt for a different token, try the REST API directly, find some
   other host to deploy to — all amount to working around a boundary that exists on purpose. The
   right move was to do everything that *was* possible (repo, code, workflow, docs all pushed and
   correct), reduce the human's remaining work to a single switch, document that switch in both
   the workflow and the README, and hand it over honestly.

The failed `enablement: true` got reverted rather than left in place. Leaving behind a setting
that demonstrably cannot work — on the theory that it looks like it's trying — is worse than not
having it, because the next person to read the file will believe it.

---

## Real data, and what it cost me to find out

### The estimates were wrong, and the tool said so out loud

For three rounds this thing carried prices I had estimated by hand. They looked reasonable. Several
were badly wrong.

Wanstead: estimated £700k, actual median house price **£935k** across 143 sales. Shenfield:
estimated £675k, actual **£872k**. Those aren't rounding errors — they're the difference between an
area being inside a £750k budget and not. On my guesses, Wanstead ranked as the best all-rounder and
got recommended as such. On the real numbers it doesn't qualify at all.

The data is HM Land Registry **Price Paid Data**: every residential sale in England and Wales,
1,184,740 transactions for 2025–26, free, open, and recording what people *actually paid* rather
than what they hoped for. It took about ten minutes to wire in and it invalidated a recommendation
I'd given twice.

**Lesson: an estimate that never gets checked will quietly become a fact.** Mine sat in the codebase
looking like data, formatted like data, feeding a ranking like data — for three rounds. Nothing
about how they were *displayed* said "these are vibes." If you ship a placeholder, label it as one
loudly enough that future-you can't mistake it. Note what happened to the rent figures in the same
commit: they stayed estimates, because Land Registry publishes sales and not lettings — so now they
say "(estimate)" on the face of every card. That label is doing real work.

### Asking the data instead of my memory

Areas had to be mapped to postcode zones, and E11 is a trap: it covers both Wanstead and Leytonstone,
which are different places at different prices. My first instinct was to hand-assign postcode sectors
from memory. That would have been confident and unverifiable — the worst combination.

Instead: the Land Registry rows carry a **District** column, so E11 splits on the borough boundary
(Redbridge = Wanstead, Waltham Forest = Leytonstone) with no guessing at all. And where sectors were
genuinely needed, I looked up streets I *knew* — "does The Ridgeway land in E4 6 or E4 9?" — and let
the data answer. It came back clean: E4 9 is Highams Park, E4 6/7/8 is Chingford, CM15 8 is Shenfield.

**Lesson: when you have a dataset, interrogate it rather than your recollection.** The check cost one
query and turned a set of assertions into a set of findings. Where the data refused to give a clean
answer — E17's sectors genuinely interleave Walthamstow and Wood Street — the right move was to merge
those zones and *say so on the card*, not to force a split I couldn't defend.

### The chart where the colour turned out to be doing nothing

The price-against-commute scatter was going to colour each dot green/amber/red by whether it cleared
the 45-minute limit. Before shipping it I ran the palette through a contrast validator, and it failed:
red against amber is ΔE 3.9 under deuteranopia — effectively the same colour to a red-green colourblind
reader — and only 12.5 even with full colour vision.

The fix wasn't a better red. It was noticing that **commute time is already the x-axis**, so colouring
by a category derived from commute time was pure redundancy. The colour was decorative and I hadn't
noticed because it looked fine to me. So the dots became one quiet neutral, and the single accent got
spent on the thing that actually carries information: the **efficient frontier** — the areas where
nothing else is both cheaper and quicker.

**Two lessons, and the second is the one worth keeping:**

1. **Run the check, don't eyeball it.** I would have shipped that palette. It looked fine on my screen,
   which is exactly the problem — "looks fine to me" is not a colour-accessibility test.
2. **A failing check is often pointing at a design problem, not a colour problem.** The honest question
   wasn't "which red passes?" but "what is this colour telling anyone that the position doesn't?" The
   answer was nothing, and the chart got better for losing it.

### Knowing which door to knock on

The ask was "scrape Rightmove or Zoopla." That was the wrong tool for the job on three separate counts,
and it's worth separating them because only one is about rules:

- **Contractual:** both explicitly forbid automated collection.
- **Practical:** both run bot protection; a scraper is a maintenance liability that breaks on their
  schedule, not yours.
- **Substantive — and this is the real one:** asking prices are an *opening bid*. Land Registry records
  what changed hands. For "can we afford this area," sold prices are simply better evidence.

**Lesson: when a request names a method, solve for the goal.** "Scrape a portal" was a guess at how to
get property data; the actual goal was to compare real housing costs against the commute. Once you
separate those, an open, free, complete, legally unambiguous dataset is sitting right there. The
version that respects the constraint is also the version that's more accurate — that happens more often
than people expect, and it's worth looking for before assuming a trade-off exists.

### And a note on where recommendations come from

Somewhere in this, the question came up: what are people on Reddit actually saying? Reddit turned out to
be entirely unreachable from this environment, so what I could get was press and editorial coverage
instead — and I said so rather than letting search results stand in for community consensus they weren't.

It still earned its keep. It surfaced **Poplar**, which the rail geography alone hadn't flagged, and
which lands at 24 minutes and £602k — on the efficient frontier. It also surfaced a whole cluster of
inner-east-London areas (Bethnal Green, London Fields, Clapton, Hackney Wick, Leyton, Tottenham Hale)
that a model weighted toward parking had systematically skipped.

**Lesson: an optimiser only searches where you point it.** Mine was pointed at rail geography and
parking, so it kept finding suburbs — correctly, given its inputs, and incompletely. The fix for a
blind spot is rarely a better algorithm; it's a different source of candidates.

### The constraint that was hiding behind the constraint

Late on, one sentence changed the answer more than any dataset had: *we are not purchasing and just
renting.*

The obvious consequence was easy — swap the money axis from sold prices to rent, re-score, done. The
non-obvious consequence was the interesting one. **Many London developments are granted planning
permission on condition that residents can never hold an on-street parking permit.** Hackney, Tower
Hamlets, Newham, Southwark and Barking & Dagenham all run formal car-free registers. It binds tenants
exactly as it binds owners, and it is attached to the building permanently.

Which meant the "parking" score had quietly been measuring the wrong thing all along. It answered *is
there anywhere to put a car* when the question that actually decides it is *will your tenancy let you*.
Those come apart badly, and they come apart precisely where it hurts: the newest, best-connected,
best-value developments are the ones most likely to be car-free. Stratford, Custom House, Royal
Victoria, Canada Water, Poplar, Tottenham Hale — six of the strongest recommendations on the previous
scoring — all became "assume no car."

The fix was to split it into two scores and take the **lower** of the two, because both have to be true
at once.

**Lessons, and the first is the one I keep relearning:**

1. **A change in circumstances is not always a change of parameter.** "We're renting" sounds like it
   only touches the price column. It actually invalidated a *different* column, one nobody was looking
   at. When a premise shifts, re-derive rather than re-parameterise — ask what else was resting on the
   old premise, and go and check.
2. **A score can be well-calibrated and still be measuring the wrong quantity.** The parking numbers
   weren't inaccurate; they were answering a question adjacent to the one that mattered. That failure
   mode is much harder to catch than a wrong number, because nothing looks broken — it looks confident
   and precise, and it is quietly about something else.
3. **Some facts are yes/no, and those are the ones to check first.** Whether an address is in a
   permit-free development isn't a judgement call or a score out of five. The council will tell you.
   When a binary fact sits upstream of a whole ranking, it belongs at the top of the page — which is
   where it now is, in a callout, above everything else.

There is a pleasing coda. Wanstead had been dropped one round earlier for being unaffordable — a £935k
median house price against a £750k budget. Renting removes that barrier completely, and it walked
straight back onto the shortlist at about £1,900 a month. Two rounds, two reversals, both driven by
finding out something true rather than by thinking harder about what I already had.

### When the objective function changes shape

The last twist was the biggest: a second job, a second office, two days a week. My first instinct
was that this was a parameter change — add a column, weight it, re-rank. It wasn't. **It changed
which corridors are good at all.**

The two offices reward opposite geography, and you can see exactly why from one fact each:

- **Aldgate** is a 9-minute walk from Liverpool Street and a 5-minute walk from Fenchurch Street.
  That points east: Essex, c2c, the Central line.
- **3 Brandon Road, N7** is 620 yards from Caledonian Road & Barnsbury, which is on the **Mildmay
  line** — and the Mildmay line runs to **Stratford** and **Highbury & Islington**. That points at a
  completely different map.

So the thing that had been winning for four rounds died. **Upminster**: 39 minutes to Aldgate,
driveway with every house, inside budget — and **72 minutes** to N7, the worst on the list. That's
not a compromise you trade off against something. It's a veto.

What replaced it did so for a structural reason worth understanding rather than just recording: the
**Central line reaches Stratford in minutes and Liverpool Street directly**, which is the rare
double connection that serves both offices. Run the joint Pareto frontier over only the areas where
a tenancy would actually let you keep a car, and exactly one survives — Wanstead — because it beats
every other parkable area on *both* journeys simultaneously.

**Lessons:**

1. **Adding an objective is not the same as adding a column.** A single-destination optimiser
   doesn't become a two-destination one by averaging. The set of good answers changes, because
   "good" was defined relative to one geography and there are now two. When someone adds a
   constraint, check whether your *search space* is still the right one — not just your scoring.
2. **Pareto frontiers earn their keep the moment there are two objectives.** With one axis you can
   just sort. With two, "best" is genuinely ambiguous, and the frontier is the honest answer:
   here is the set nothing else beats on both; everything else is dominated. It also made the
   parking constraint legible — the unrestricted frontier is three areas you can't park at, and
   restricting it collapses to one.
3. **Model the uncertainty, don't resolve it.** The job isn't offered yet — it's a final-round
   interview. Baking it in would have been wrong, and ignoring it would have been wrong. So it's a
   switch, defaulted on, with the single-commute ranking one click away. Tools that plan around
   uncertain futures should let you *see both futures*, not pick one for you.

Which makes four reversals across this project, every one caused by learning something rather than
thinking harder: a terminus walk table, real sold prices, a renting-versus-buying premise, and a
second office. The tool got better each time it was told it was wrong — which is the only
real argument for building the thing as a model rather than as an opinion.

### The correction that cost the headline

The last input was one line: *both of us have two days a week, but I would prefer three.* I had been
assuming Tisha commuted five days. She commutes two.

The arithmetic consequence was pleasant — household travel for Wanstead fell from 9.6 hours a week to
7.3. The design consequence was bigger. At five days against two it was right to let Tisha's journey
outvote mine; at two against three they weigh almost the same, which means **an area that is excellent
for one person and grim for the other no longer averages out into looking fine.** So the score now
takes points off for a lopsided pair, and every card carries a balance reading measured against each
person's own ceiling rather than in raw minutes — 47 out of 60 is not the same burden as 47 out of 45.

The deeper consequence took longer to see. Five commuting days a week *between two people* means five
days a week where neither of them goes anywhere. They are choosing a place they'll spend far more time
in than travelling from. So the commute should set the boundary of what's possible and then largely get
out of the way — which is a different optimisation from the one I'd been running for six rounds.

That became a **Hybrid** button: commute weight down, walkability and green space up. And then it
embarrassed me.

I had already written, in the page, that Wanstead "comes top under every ceiling and both weightings."
I ran the check anyway, and under the hybrid weighting **Brentwood beat it, 83% to 81%** — cheaper,
and the only area scoring five out of five for walkability, green space and parking at once.

**Three lessons, and the last is the one that matters:**

1. **Verify claims about your own output, especially the flattering ones.** "Wanstead wins under every
   configuration" was a lovely sentence and I had no evidence for half of it. I nearly shipped it
   because it *felt* established after several rounds of Wanstead winning.
2. **A robustness claim is a testable claim.** "This answer doesn't move" is precisely the kind of
   statement you can check in one query, so there is no excuse for asserting it from impression.
3. **When two reasonable weightings disagree, that disagreement is the finding.** The temptation was
   to pick a weighting and present one answer. But the honest report is that Wanstead wins if the
   commute leads and Brentwood wins if the neighbourhood does — and **which of those is right depends
   on something the model cannot know**: whether five commuting days a week feels like a lot or a
   little to the people living it. A tool that hides that behind a single ranking is pretending to an
   authority it does not have. Surfacing it turns the tool from an oracle into an argument you can
   have properly.

---

## Things I chose not to build, and why

Naming what you deliberately left out is part of the deliverable. It's the difference between a
gap and an oversight.

- **No slippy map.** See above — the CSP forbids tile servers, and the hand-drawn map turned out
  to be the better answer anyway.
- **No live listings.** Sold prices, not what is on the market today. See above.
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
