---
title: "How Much More Could This Power Line Carry?"
date: 2026-10-13
authors:
  - name: Paul Brunet
    url: https://github.com/p-brunet
description: >
  A sensorless dynamic line rating of the Ontario–Québec HVDC tie line near Ottawa,
  built only from open weather forecasts, LiDAR terrain and OpenStreetMap — and why
  the static rating it is compared against turns out to be optimistic almost a third
  of the time.
thumbnail: "https://raw.githubusercontent.com/p-brunet/dlr-demo/main/docs/figures/bottleneck_spans.png"
tags:
  - energy
  - visualization
  - data-engineering
keywords:
  - dynamic line rating
  - DLR
  - transmission
  - Hydro One
  - Hydro-Québec
  - IESO
  - HRDPS
  - LiDAR
  - CIGRE TB 601
  - linerate
  - weather forecasting
  - GeoLibre
---

<span class="badge badge-python">Python</span>
<span class="badge badge-gis">GIS</span>
<span class="badge badge-rasterio">rasterio</span>
<span class="badge badge-matplotlib">matplotlib</span>

# How Much More Could This Power Line Carry?

**GitHub**: [p-brunet/dlr-demo](https://github.com/p-brunet/dlr-demo)

## Overview

A power line's capacity is a temperature limit in disguise. Current heats the
conductor; sun heats it further; wind and cold air take the heat away. Utilities
usually rate a line once, for a bad-but-plausible day — a hot afternoon with
barely any wind — and then use that **static rating** all season.

**Dynamic line rating (DLR)** replaces that single number with one that follows
the weather. It is one of the cheapest ways to get more out of an existing grid,
and most deployments hang sensors on the wires. This project asks a simpler
question: *how far can you get with open data alone?*

The test case is **A41T/A42T**, the 230 kV double-circuit line that runs 34 km
from Hydro One's Hawthorne station in Ottawa, across the Ottawa River, to
Hydro-Québec's Outaouais converter station. It carries the main Ontario–Québec
tie: on 10 October 2026 it ran pegged at about 1,234 MW all day, right at the
converter's 1,250 MW limit.

One honest caveat up front: with both circuits in service, the **converter** —
not the line — caps that transfer. Thermal headroom on the line matters when one
circuit is out, which is where this kind of analysis would be used. I come back
to that below.

## Data & Pipeline

::::{grid} 1 1 3 3

:::{card} Line geometry — OpenStreetMap
The route, 111 towers and two substations, traced and validated from Overpass. Twice, the way that actually crosses the river carried no tags at all — so the pipeline now refuses any route that doesn't reach both substations.
:::

:::{card} Weather — ECCC HRDPS
Canada's 2.5 km, 0–48 h forecast model, read as cloud-native Zarr through [dynamical.org](https://dynamical.org/): every run since 9 July 2026, about 390 forecasts.
:::

:::{card} Terrain — NRCan HRDEM
1 m LiDAR terrain and surface models. They give each tower's altitude — and how many trees or buildings stand close enough to shelter each span from the wind.
:::

::::

Two more sources close the loop: hourly **station observations** from ECCC at
the Ottawa and Gatineau airports, used to check the forecasts, and **IESO's
intertie flow reports**, archived daily because IESO only keeps about three
months online.

The physics is the CIGRE TB 601 heat balance, through Statnett's open-source
[`linerate`](https://github.com/statnett/linerate) library. The project's tests
reproduce the brochure's worked examples before anything else runs. The real
conductor isn't public, so I assume a typical one — two ACSR *Drake* conductors
per phase — and report most results **relative to the static rating**, which
barely depends on that choice (swapping for a size up or down moves the median
gain by one percentage point).

## The Weakest Span Keeps Moving

A line is only as strong as its weakest span, so the rating is computed for all
110 spans every hour and the line takes the minimum.

::::{grid} 1 1 2 2

:::{card} Terrain and tower heights
![Span model](https://raw.githubusercontent.com/p-brunet/dlr-demo/main/docs/figures/span_model.png)
The LiDAR surface model even "sees" the towers: 26–87 m above ground, tallest at the river crossing.
:::

:::{card} Where the bottleneck sits
![Weakest-span map](https://raw.githubusercontent.com/p-brunet/dlr-demo/main/docs/figures/bottleneck_spans.png)
95 of the 110 spans are the limiting one at some hour.
:::

::::

The reason is geometry. This line doesn't run straight: it heads north-east,
crosses the river, then zig-zags north to the converter, so its spans point in
every direction. Whatever way the wind blows, it runs nearly **along** some
spans — and wind along a conductor cools it far less than wind across it.

## Headline Numbers

Over July to mid-October 2026 (2,316 hours):

| | |
|---|---|
| Static summer rating | ≈850 MW per circuit |
| Weather-based rating, median | **+8 %** over static |
| …if hourly wind is never treated as exactly parallel to a span | **+21 %** |
| Hours where the weather-based rating is **below** static | **30 %** |
| Ambient-temperature-only rating above the weather-based one | 99 % of hours |

The second-to-last line is the surprising one. The static rating assumes a gentle
4 km/h breeze **perpendicular to every span at once** — on a line that turns, that
almost never happens. Ambient-adjusted ratings, which only update the temperature,
inherit the same optimistic wind.

::::{grid} 1 1 2 2

:::{card} One week in August
![Ratings over a week](https://raw.githubusercontent.com/p-brunet/dlr-demo/main/docs/figures/ratings_week.png)
:::

:::{card} How often each rating beats static
![Distribution](https://raw.githubusercontent.com/p-brunet/dlr-demo/main/docs/figures/ratio_distribution.png)
:::

::::

The gap between +8 % and +21 % is the single biggest modelling choice in the
project: whether a 2.5 km model's hourly wind direction should be trusted exactly
at the scale of a 300 m span. Minute-by-minute wind from Environment Canada's
station at the Experimental Farm narrows it: within an hour the direction swings
by 8–15°, and averaging the cooling over those minutes behaves like a **9°**
minimum angle — a median gain of about **+12 %**. One station, four weeks and 9 km
from the line is evidence rather than proof; a couple of anemometers on the spans
that limit the line most often would settle it.

## Can You Forecast It a Day Ahead?

Operators need ratings before the hour, not after. Using every HRDPS forecast run
at leads of 1 to 48 hours:

- the forecast rating is off by **3 % at 7 hours and 5.5 % at 48 hours** (against
  the model's own short-range analysis), about 2.5× better than assuming tomorrow
  looks like now;
- a safety margin fitted on July–August (9 % at 7 h, 17 % day-ahead) keeps the
  resulting **"secure" rating above the true one in only 1 % of September–October
  hours** — the static rating did so 19 % of the time.

![Day-ahead forecasts](https://raw.githubusercontent.com/p-brunet/dlr-demo/main/docs/figures/secure_dayahead_week.png)

The secure rating isn't much higher than static on average (0.95× to 1.04×
depending on the wind-direction treatment). Its value is *when* it differs: it
drops exactly on the calm, along-the-line afternoons when static is unsafe, and
rises on cool windy days.

Checked against real observations, the picture holds up better than I first
thought. Hourly airport reports are two-minute snapshots, which exaggerate the
error; using the minute data instead, a virtual copy of the line placed at the
station needs a **16 %** margin (20 % in light wind) — in line with the
model-based day-ahead margin. One warning sign remains: in light wind the model
over-forecasts the wind by about 0.5 m/s, exactly when ratings bind.

## Explore the Map

[![Story map preview](https://raw.githubusercontent.com/p-brunet/dlr-demo/main/docs/figures/story_map_preview.png)](https://p-brunet.github.io/dlr-demo/)

The interactive version opens on a **live forecast for the next 48 hours**,
rebuilt automatically from every new Environment Canada weather run (about every
6 hours), with the secure rating, the weakest span hour by hour and the forecast
wind. A second view walks back through every hour since July in six short
chapters — hover any span to see the wind's angle to its wire. Like the solar map before it,
it's a link rather than an embed: [**open the story map →**](https://p-brunet.github.io/dlr-demo/)
— or the same data with a time slider in
[**GeoLibre →**](https://p-brunet.github.io/dlr-demo/geolibre.html).

## What About the N-1 Case?

If one of the two circuits trips, the other must carry the flow within its
emergency rating (127 °C in Ontario). With the assumed conductor, a single circuit
manages about 1,090 MW — below the tie's typical flow of about 1,165 MW, more
than half the time. Weather-based ratings would cover an extra 5 % of hours.

That mostly tells us the real conductor is larger than my assumption, or that the
converter runs back after a contingency — open data can't say which. It's the
right question for DLR on this line, but answering it needs the operator's
numbers.

## Limitations

- The conductor, its maximum temperature and the static assumptions are typical
  published Ontario values, not Hydro One's.
- 2.5 km weather can't resolve span-scale wind; the terrain sheltering index is a
  heuristic.
- Three months of summer and early fall. Winter — usually where DLR gains are
  largest — isn't in the archive yet.
- Ratings are steady-state; the conductor's thermal inertia would smooth short dips.
- Not for operational use.

## Stack

| Tool | Role |
|---|---|
| `linerate` | CIGRE TB 601 heat balance and steady-state ampacity |
| `dynamical-catalog`, `xarray`, `zarr` | HRDPS forecast archive, corridor extraction, span sampling |
| `rasterio` | NRCan HRDEM terrain windows over HTTP range requests |
| `geopandas`, `shapely`, `pyproj` | Route validation, span geometry, rotated-pole transforms |
| `pandas`, `httpx` | ECCC station observations, IESO flow archive |
| `matplotlib` | Static figures |
| MapLibre GL, GeoLibre | Interactive story map |
| GitHub Actions | Daily IESO archive, Pages publishing |

Source code, data and full method notes: [github.com/p-brunet/dlr-demo](https://github.com/p-brunet/dlr-demo).
