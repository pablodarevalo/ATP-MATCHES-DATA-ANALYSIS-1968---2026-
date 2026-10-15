# ATP Tennis Match Analysis (1968–2026)

SQL-based data analysis project using ~199,000 ATP men's tour matches, built in MySQL. The goal is to clean, model, and query real historical tennis data to answer questions about surface trends, player performance, rivalries, and how well ranking predicts match outcomes.

## Data Source

Match data compiled by [Jeff Sackmann](https://github.com/JeffSackmann/tennis_atp), covering ATP tour-level matches from 1968 to 2026.

Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) — Attribution: Jeff Sackmann / Tennis Abstract. Used here for non-commercial, educational purposes.

## Tech Stack

- **MySQL 8.0** (MySQL Workbench) — storage, modeling, querying
- **Python (pandas)** — merging and cleaning the 59 yearly source CSVs into one file before import
- **Power BI** *(planned)* — dashboard built on top of the SQL data

## Data Model

| Table | Description | Status |
|---|---|---|
| `matches` | ~199,000 matches, 1968–2026: tournament, surface, date, winner/loser, score, and match stats (aces, double faults, serve points, etc.) | ✅ Loaded & cleaned |
| `players` | Player bio data: name, hand, date of birth, country, height | ⬜ Planned |
| `rankings` | Weekly ATP ranking snapshots per player | ⬜ Planned |

`matches.winner_id` / `matches.loser_id` are built to link to `players.player_id` once that table is added.

## Data Cleaning

Raw data isn't taken at face value — every fix below was found, verified, and corrected with SQL, not assumed:

- **Duplicate tournament names.** "Australian Open" appeared under three different spellings (`Australian Chps.`, `Australian Open-2`, plus the correct name), which would have inflated the Grand Slam count to 6 instead of 4. Fixed with a targeted `UPDATE`.
- **Impossible match durations.** Two matches showed durations of 1146 and 987 minutes (19+ hours) with 2–3 set scorelines — physically impossible. Checked the `score` field for each: one was corrected to its real duration (84 minutes, based on external verification), the other set to `NULL` since no reliable source was found. After the fix, the longest match in the dataset correctly resolves to Isner–Mahut, Wimbledon 2010 (665 minutes) — the actual longest match in tennis history, confirming the fix.
- **Win streak over-count.** A first pass at the longest win streak gave Djokovic 47 consecutive wins (2010–2011), but the official ATP record is 43. Investigated match-by-match and found the gap came from Davis Cup matches and round-robin wins at the 2010 ATP Finals — neither counts toward official streaks since they aren't single-elimination. Excluding both brought the result to exactly 43, matching the official record.
- **Upset ranking direction.** An early version of the "biggest upsets" query had the ranking subtraction backwards, surfacing *expected* results (top players beating lower-ranked ones) instead of actual upsets. Also excluded walkovers and retirements (`W/O`, `RET`), since those matches weren't fully played and don't reflect a real competitive upset.
- **Missing match stats in early years.** Columns like `w_ace`, `w_svpt`, and `winner_rank` are largely `NULL` before the 1990s, since those stats weren't recorded yet. Analyses involving these fields are scoped to the years where data is available.

## Key Findings

- **Hard courts took over the tour.** Hard surface went from a minority (23.9% of matches in the 1970s) to dominating the game (59.8% in the 2020s). Clay stayed fairly stable (~31–41%) across every decade. Carpet courts disappeared almost completely — from 17.5% in the 1970s to effectively 0% by the 2020s, matching the ATP's phase-out of carpet in the late 2000s.
- **The game got more aggressive, then leveled off.** Average aces per match grew from 9.9 (1990s) to 12.5 (2010s–2020s), with the sharpest jump in the 1990s–2000s (likely racket technology), before flattening out — consistent with the ATP slowing down court surfaces in the 2010s to reduce serve-dominated matches.
- **Ranking predicts outcomes better at Grand Slams.** The favorite (lower-ranked player) wins 70.4% of matches at Grand Slams vs. 65.0% elsewhere — a 5.4 point gap likely explained by the best-of-5 format, which gives the stronger player more time to assert an edge compared to best-of-3 events.
- **Djokovic's 2011 run is the real deal.** After correctly excluding Davis Cup and round-robin matches, his 43-match win streak (Jan–May 2011) matches the officially recognized ATP record exactly.
- **Djokovic vs. Nadal is tennis' most played rivalry** (60 matches), ahead of Djokovic vs. Federer (51) and Nadal vs. Federer (41).

## Questions This Project Answers

- How many matches, players, and Grand Slam tournaments are in the dataset?
- How has surface usage shifted across decades?
- What are the longest matches in ATP history?
- Who are the oldest players to compete in a Grand Slam?
- How has the average age of Grand Slam champions changed over time?
- How has the average number of aces per match evolved?
- Who are tennis' most-played rivalries, and what does the Nadal–Djokovic head-to-head look like?
- What is the longest win streak in ATP history, and does it match the official record?
- What are the biggest ranking upsets in tennis history?
- Does ranking predict match outcomes better at Grand Slams than at other tournaments?

## Repository Structure

```
/sql
  atp_analysis.sql       -- all queries, cleaning steps, and findings, in order
/python
  merge_csvs.py           -- combines yearly source files into one CSV
README.md
```

## Setup

1. Download the source CSVs from [JeffSackmann/tennis_atp](https://github.com/JeffSackmann/tennis_atp).
2. Run `python/merge_csvs.py` to combine yearly files into `atp_matches_all.csv`.
3. Create the `matches` table (schema included at the top of `atp_analysis.sql`).
4. Load the merged CSV with `LOAD DATA LOCAL INFILE`.
5. Run the cleaning `UPDATE` statements included in `atp_analysis.sql`.
6. Run the analysis queries.

## Status

🚧 In progress — `matches` table loaded, cleaned, and fully analyzed (10 queries covering surface trends, data cleaning, rivalries, win streaks, upsets, and ranking predictiveness). `players` and `rankings` tables and a Power BI dashboard are planned next.
