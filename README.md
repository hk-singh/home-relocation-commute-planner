# Aldgate Relocation & Commute Planner

A single-file tool for choosing where to live around a job at **Portsoken House, 155–157
Minories, London EC3N 1LJ** — the eastern edge of the City, by Aldgate.

Open `index.html` in any browser. No build, no server, no dependencies.

## What it does

Scores 65 London areas against four constraints at once:

- **Door-to-door commute** under 45 minutes, modelled properly — walk to the station, platform
  wait, time on the train, interchange penalty, and the walk from the arrival station to the desk.
- **Parking, in two parts** — whether there is anywhere to put a car, *and* whether a tenancy
  actually entitles you to use it. Many London developments are permit-free by planning condition,
  so tenants can never hold a resident permit. Areas are scored on the lower of the two.
- **Walkability** and **green space** — the neighbourhood you'd actually live in.

Five views: a ranked list with each commute drawn on a shared 60-minute scale, a side-by-side
shortlist, a polar time map (bearing = compass direction, radius = minutes), and a written
method with the assumptions and caveats spelled out, plus a price-against-commute chart that picks
out the **efficient frontier** — the areas nothing else beats on both cost and journey time.

## Rents and prices

Every price comes from **HM Land Registry Price Paid Data** for 2025–26: 1,184,740 recorded
residential sales, filtered to these postcode zones and to standard full-value transactions, under
the Open Government Licence. These are prices actually paid, not asking prices. Sample sizes are
shown against every figure.

This is deliberately *not* scraped from Rightmove or Zoopla. Both prohibit automated collection in
their terms, both run bot protection that breaks scrapers quickly, and sold prices are better
evidence than asking prices anyway. The trade-off is that there are no live listings here — run
that search yourself and bring addresses back to check against the commute.

Rents come from the **ONS Price Index of Private Rents**, the official monthly measure by local
authority — shown on every card with the month it was measured. The per-area 2-bed figures are
estimates calibrated against those borough anchors, and are labelled as estimates everywhere they
appear: they are the only unmeasured numbers in the app.

Sold prices are kept as secondary context ("if you were buying") — not what a renter pays, but a
good signal of how an area is regarded.

Everything is adjustable — the time ceiling, how far you'll walk to a station, the interchange
penalty, the rent cap, and how much each of the four priorities counts. Settings and shortlist
persist in `localStorage`.

## The finding that shaped it

The office is within walking distance of six stations, and the final walk ranges from 3 minutes
(Aldgate, where the Metropolitan line terminates) to 9 (Liverpool Street). That difference
re-ranks the whole map — it's why Upminster, 22 minutes out on the c2c into Fenchurch Street,
lands inside the limit while places that look far closer on a tube map don't.

Read the other way, it also explains the gaps. Cannon Street is a 14-minute walk, so the entire
Southeastern corridor out through Bexley loses ten minutes at the end of every journey no matter
how fast the train — which is why Bexleyheath, with cheap houses and a direct train, still comes
out fifteen minutes over. Areas that lose are included and shown with their numbers rather than
left out, so an absence never has to be guessed at.

## Deploying

`.github/workflows/pages.yml` publishes `index.html` to GitHub Pages on every push to `main`.

It needs one manual step first, once per repository: **Settings → Pages → Build and deployment →
Source: _GitHub Actions_**. A workflow token is not allowed to turn Pages on by itself, so the
first run fails with `Get Pages site failed` until that switch is flipped. After that, re-run the
workflow (Actions → Deploy to GitHub Pages → Run workflow) and every later push deploys
automatically.

## Caveats

Journey times are built from scheduled services and typical peak frequencies, not a live API.
Area scores are hand-researched judgement calls, and parking in particular varies street by
street. This is a tool for building a shortlist — do the journey once at 8am on a Tuesday before
offering on anything.

Full write-up, design decisions and post-mortem: **[FORHARSH.md](FORHARSH.md)**.
