# Aldgate Relocation & Commute Planner

A single-file tool for choosing where to live around a job at **Portsoken House, 155–157
Minories, London EC3N 1LJ** — the eastern edge of the City, by Aldgate.

Open `index.html` in any browser. No build, no server, no dependencies.

## What it does

Scores 44 London areas against four constraints at once:

- **Door-to-door commute** under 45 minutes, modelled properly — walk to the station, platform
  wait, time on the train, interchange penalty, and the walk from the arrival station to the desk.
- **Parking** — how realistic it is to keep a car there.
- **Walkability** and **green space** — the neighbourhood you'd actually live in.

Four views: a ranked list with each commute drawn on a shared 60-minute scale, a side-by-side
shortlist, a polar time map (bearing = compass direction, radius = minutes), and a written
method with the assumptions and caveats spelled out.

Everything is adjustable — the time ceiling, how far you'll walk to a station, the interchange
penalty, the rent cap, and how much each of the four priorities counts. Settings and shortlist
persist in `localStorage`.

## The finding that shaped it

The office is within walking distance of six stations, and the final walk ranges from 3 minutes
(Aldgate, where the Metropolitan line terminates) to 9 (Liverpool Street). That difference
re-ranks the whole map — it's why Upminster, 22 minutes out on the c2c into Fenchurch Street,
lands inside the limit while places that look far closer on a tube map don't.

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
