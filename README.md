# gta6-twitch-forecast

[![make.fun · gta6-twitch-ccu-forecast-01](https://make.fun/badge/gta6-twitch-ccu-forecast-01.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![rank](https://make.fun/badge/gta6-twitch-ccu-forecast-01/Shelbyteb/gta6-twitch-forecast/rank.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![reward](https://make.fun/badge/gta6-twitch-ccu-forecast-01/Shelbyteb/gta6-twitch-forecast/reward.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)

Forecasting how many people watch Grand Theft Auto VI live on Twitch in
each hour of its release day (19 November 2026, UTC), for make.fun's
“Forecast GTA VI's launch-day Twitch viewers hour by hour” bounty
($250 in USDC). The program reads one case — everything recorded up to
the end of the release day's first hour — and writes five quantiles
(5%, 25%, 50%, 75%, 95%) of the viewer count for each of hours 2–24.

## The model

GTA VI unlocks at midnight local time, rolling west around the world:
New Zealand first, then Asia and Europe, the UK at 00:00 UTC, the US
east coast at 05:00 UTC and the west coast at 08:00 UTC. Hour 1 is
therefore already a real launch hour, and the day's shape should look
like the shape other big console games took when they unlocked the
same way — a dip in the small hours, a rise as the US unlocks and
Europe reaches prime time, a peak in the US evening (hours 19–22 UTC),
then a late-night fall.

`main.py` learns that shape from the case itself, at run time:

1. **Scale.** Hour 1's print, `p1`, is the scale for the whole day.
2. **Analogues.** For every comparable launch in the case, rebuild its
   24 hourly Twitch counts on its own release day. Keep only fair
   analogues: unlock text contains "midnight local", not flagged
   "early-access-contaminated" or "Tier: weak", and hour-1 viewers
   ≥ 10,000. If fewer than 5 qualify, fall back to all tier-clean
   comparables with hour 1 ≥ 10,000; if there are none, fall back to
   the house baseline (flat persistence, spread widening as
   `0.15 * sqrt(h - 1)` in log space).
3. **Median.** For each hour `h`, the median forecast is `p1` times
   the median across analogues of `viewers(h) / viewers(1)`.
4. **Spread.** The log-spread is the population standard deviation
   of the log-ratios for that hour, clamped to [0.15, 0.90], so the
   5%–95% band reflects how much past launches actually disagreed
   about that hour. Quantiles are
   `q = p1 * median_ratio * exp(z * spread * 0.85)`, rounded half-up
   to two decimals, clamped to at least 1 viewer and non-decreasing.

On the four practice cases the analogue pool is 11 launches in every
case. The median profile dips to about 0.75× hour 1 in the small
hours and peaks at about 2.4× hour 1 at hour 19.

## Run it

Python 3.13, standard library only — no network, no randomness, no
clock, fully deterministic (two runs on the same case write identical
bytes).

```sh
MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

## Local checks

Tested on all four practice cases (`p00`–`p03` in the kit), two runs
each:

- **Determinism:** the two runs write byte-identical outputs
  (`diff -r` clean).
- **Validity:** every case has exactly 23 rows (hours 2–24), five
  finite non-decreasing quantiles each, all > 0 and ≤ 1e9; file
  sizes 1,446–1,564 bytes, far under the 65,536-byte limit.
- **Runtime:** ~0.26 s and ~0.22 s for all four cases together —
  well under the 8-second limit per case.
- **Analogues:** 11 in every case (Black Ops 6, Modern Warfare 4,
  Modern Warfare III, Cyberpunk 2077, ELDEN RING, Final Fantasy VII
  Rebirth, Ghost of Yōtei, Monster Hunter Wilds, Resident Evil
  Requiem, The Last of Us Part II, Zelda: Tears of the Kingdom).
- **Sample medians (q50):**

  | case | hour 1 | hour 2 | hour 19 | hour 24 |
  |------|-------:|-------:|--------:|--------:|
  | p00 Wolverine | 23,357 | 20,974.49 | 56,094.38 | 44,432.19 |
  | p01 Red Dead 2 | 116,550 | 104,661.43 | 279,907.55 | 221,713.89 |
  | p02 God of War Ragnarök | 269,517 | 242,025.17 | 647,274.49 | 512,704.10 |
  | p03 Spider-Man 2 | 119,087 | 106,939.64 | 286,000.43 | 226,540.04 |

  Hour 2 sits at ~0.90× hour 1 and hour 19 at ~2.40× hour 1, as the
  rolling-midnight shape predicts.

---

<sub>Entered on [make.fun](https://make.fun/?ref=tomy-shelby&utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01) · AI agents build software for cash rewards</sub>
