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
/*ATP MATCHES DATA ANALYSIS FROM 1968 UNTIL 2026 */

-- SAMPLE SIZE (MATCHES) 
SELECT COUNT(*) FROM matches;

-- AFTER RESULTS, THE OUTPUT SHOWS 199.389 MATCHES OFFICIALLY PLAYED 
-- ----------------------------------------------------------------------------------

-- NUMBER OF PLAYERS ON DATABASE 
SELECT COUNT(DISTINCT player_id) AS player_count
FROM (
    SELECT winner_id AS player_id FROM matches
    UNION
    SELECT loser_id AS player_id FROM matches
) AS all_ids;
-- THE QUANTITY OF PLAYERS IS 7709 

-- ----------------------------------------------------------------------------------

-- HOW MANY GRAND SLAM TOURNAMENTS ARE THERE? 
SELECT DISTINCT (tourney_name) FROM matches
WHERE tourney_level = 'G';
-- BUG DETECTED, THERE ARE THREE OPTIONS REGARDING "AUSTRALIAN...", THE REAL ONE IS AUSTRALIAN OPEN.
-- RESULT:
/*
Tournaments name 
- Roland Garros
- Wimbledon
- Us Open
- Australian Open 
- Australian Chps.
- Australian Open-2
*/
-- Cleaning Code 
/*
UPDATE matches
SET tourney_name = 'Australian Open'
WHERE tourney_name IN ('Australian Chps.', 'Australian Open-2', 'Australian Open -2');
*/

-- After running the cleaning code, the wrong values were replaced. 
-- Result: 
/*
Tournaments name: 
- Roland Garros
- Wimbledon
- Us Open
- Australian Open 
*/
-- There are four Grand Slam Tournaments. The wrong values were fixed.

-- ----------------------------------------------------------------------------------
-- MATCHES PLAYED BY SURFACE, BY DECADE (SURFACE SHARE %)
-- NOTE: the "1960" bucket would only cover 1968-1969, so it's excluded for a fair comparison
SELECT 
    FLOOR(YEAR(tourney_date) / 10) * 10 AS decade,
    surface,
    COUNT(*) AS matches_played,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY FLOOR(YEAR(tourney_date) / 10) * 10), 1) AS pct_of_decade
FROM matches
WHERE surface IS NOT NULL AND YEAR(tourney_date) >= 1970
GROUP BY decade, surface
ORDER BY decade, pct_of_decade DESC;
-- RESULT: hard courts went from a minority surface (23.9% in the 1970s) to dominating
-- the tour (59.8% in the 2020s). Clay stayed fairly stable (~31-41%) across decades.
-- Carpet surface disappeared almost completely (17.5% in the 1970s to 0.2% in the 2010s,
-- gone by the 2020s) -- the ATP phased out carpet courts from the tour in the late 2000s.
-- Grass stayed a small, stable share (~9-17%), since only a few tournaments (led by
-- Wimbledon) still use it.

-- ----------------------------------------------------------------------------------
-- TOP 3 LONGEST MATCHES IN HISTORY
SELECT winner_name AS winner, loser_name AS loser, tourney_date AS match_date, 
       tourney_name AS tournament_name, minutes AS match_duration
FROM matches
WHERE minutes IS NOT NULL
ORDER BY minutes DESC
LIMIT 3;
-- BUG DETECTED, TOP RESULT SHOWED 1146 MINUTES (19 HOURS) ON A 2-SET MATCH, WHICH IS IMPOSSIBLE
-- CHECKED THE SCORE FIELD, MATCH WAS ONLY 2 SETS, REAL DURATION WAS 84 MINUTES
-- A SECOND CASE (987 MINUTES, 3 SETS) WAS ALSO FOUND AND SET TO NULL, NO REAL VALUE CONFIRMED
-- Cleaning Code
/*
UPDATE matches SET minutes = 84
WHERE winner_name = 'Gilles Muller' AND loser_name = 'Jeremy Chardy' AND tourney_date = '2016-01-11';

UPDATE matches SET minutes = NULL
WHERE winner_name = 'Hyeon Chung' AND loser_name = 'Martin Klizan' AND tourney_date = '2017-05-01';
*/
-- After the fix, the longest match is Isner vs Mahut, Wimbledon 2010 (665 minutes),
-- which is the real longest match in tennis history. This confirms the fix was correct.

-- ----------------------------------------------------------------------------------
-- TOP 5 OLDEST PLAYERS PLAYING A GRAND SLAM IN HISTORY
SELECT player_name, MAX(player_age) AS max_age, tourney_name, tourney_date
FROM (
    SELECT winner_id AS player_id, winner_name AS player_name, winner_age AS player_age, tourney_name, tourney_level, tourney_date 
    FROM matches
    UNION
    SELECT loser_id AS player_id, loser_name AS player_name, loser_age AS player_age, tourney_name, tourney_level, tourney_date 
    FROM matches
) AS all_matches
WHERE tourney_level = 'G'
GROUP BY player_id, player_name, tourney_name, tourney_date
ORDER BY max_age DESC
LIMIT 5;

-- ----------------------------------------------------------------------------------
-- PHYSICAL EVOLUTION AND LONGEVITY (AGE AND GAME STYLE VS TIME)

-- HOW DID THE AVERAGE AGE OF GRAND SLAM CHAMPIONS CHANGE BY DECADE (1970 VS 2000 VS 2020)?
SELECT 
    FLOOR(YEAR(tourney_date) / 10) * 10 AS decade,
    ROUND(AVG(winner_age), 1) AS average_age_champ,
    COUNT(*) AS finals
FROM matches
WHERE tourney_level = 'G' 
  AND round = 'F'
GROUP BY decade
HAVING decade IN (1970, 2000, 2020)
ORDER BY decade;
-- BUG DETECTED FIRST TRY: tourney_date is a DATE type, not a number, so dividing it directly
-- gave wrong results. Fixed by using YEAR(tourney_date) first, then grouping by decade.

-- HOW DID THE AVERAGE NUMBER OF ACES PER MATCH CHANGE OVER TIME?
SELECT 
    FLOOR(YEAR(tourney_date) / 10) * 10 AS decade,
    ROUND(AVG(w_ace + l_ace), 1) AS avg_aces_per_match
FROM matches
WHERE w_ace IS NOT NULL AND l_ace IS NOT NULL
GROUP BY decade
ORDER BY decade;
-- NOTE: w_ace/l_ace are only recorded from the 1990s onward, earlier decades will show NULL
-- RESULT: average aces per match grew from 9.9 (1990s) to 12.5 (2010s-2020s)
-- biggest jump is 1990s-2000s, likely due to racket technology improvements
-- numbers level off after 2010, which matches the ATP slowing down court surfaces 
-- around that time to reduce serve-dominated matches

-- ----------------------------------------------------------------------------------
-- HEAD-TO-HEAD: NADAL VS DJOKOVIC
SELECT 
    tourney_date,
    tourney_name,
    round,
    winner_name,
    loser_name,
    score
FROM matches
WHERE (winner_name = 'Rafael Nadal' AND loser_name = 'Novak Djokovic')
   OR (winner_name = 'Novak Djokovic' AND loser_name = 'Rafael Nadal')
ORDER BY tourney_date;

-- TOP 10 RIVALRIES WITH MOST MATCHES PLAYED (any pair of players)
SELECT 
    LEAST(winner_name, loser_name) AS player_a,
    GREATEST(winner_name, loser_name) AS player_b,
    COUNT(*) AS matches_played
FROM matches
WHERE winner_name IS NOT NULL AND loser_name IS NOT NULL
GROUP BY player_a, player_b
HAVING matches_played >= 20
ORDER BY matches_played DESC
LIMIT 10;
-- USED LEAST() AND GREATEST() SO "NADAL VS DJOKOVIC" AND "DJOKOVIC VS NADAL" COUNT AS ONE PAIR,
-- NOT TWO SEPARATE GROUPS

-- ----------------------------------------------------------------------------------
-- LONGEST WIN STREAK IN HISTORY (using window functions, "islands and gaps" technique)
WITH player_matches AS (
    SELECT winner_id AS player_id, winner_name AS player_name, tourney_date, 1 AS is_win
    FROM matches WHERE tourney_level != 'D' AND round != 'RR'
    UNION ALL
    SELECT loser_id AS player_id, loser_name AS player_name, tourney_date, 0 AS is_win
    FROM matches WHERE tourney_level != 'D' AND round != 'RR'
),
ordered_matches AS (
    SELECT 
        player_id, player_name, tourney_date, is_win,
        ROW_NUMBER() OVER (PARTITION BY player_id ORDER BY tourney_date) AS match_seq,
        ROW_NUMBER() OVER (PARTITION BY player_id, is_win ORDER BY tourney_date) AS win_seq
    FROM player_matches
),
streak_groups AS (
    -- match_seq - win_seq stays the same number while the win streak is not broken
    SELECT 
        player_id, player_name, tourney_date, is_win,
        (CAST(match_seq AS SIGNED) - CAST(win_seq AS SIGNED)) AS streak_group
    FROM ordered_matches 
    WHERE is_win = 1
)
SELECT player_name, MIN(tourney_date) AS streak_start, MAX(tourney_date) AS streak_end, COUNT(*) AS win_streak
FROM streak_groups
GROUP BY player_id, player_name, streak_group
ORDER BY win_streak DESC
LIMIT 10;
-- FIRST TRY INCLUDED DAVIS CUP AND ROUND ROBIN MATCHES, GIVING DJOKOVIC A 47-MATCH STREAK
-- (OFFICIAL ATP RECORD IS 43). CHECKED THE MATCHES ONE BY ONE AND FOUND:
--   - DAVIS CUP MATCHES DON'T COUNT ON THE OFFICIAL STREAK (REMOVED WITH tourney_level != 'D')
--   - 2 ROUND ROBIN WINS AT THE 2010 ATP FINALS ALSO DON'T COUNT (REMOVED WITH round != 'RR')
-- AFTER BOTH FIXES, DJOKOVIC'S STREAK IS EXACTLY 43 MATCHES, MATCHING THE OFFICIAL ATP RECORD

-- ----------------------------------------------------------------------------------
-- BIGGEST UPSETS IN HISTORY (lower-ranked player beating a much higher-ranked player)
SELECT 
    tourney_date,
    tourney_name,
    tourney_level,
    round,
    winner_name,
    winner_rank,
    loser_name,
    loser_rank,
    (winner_rank - loser_rank) AS rank_diff,
    score
FROM matches
WHERE winner_rank IS NOT NULL AND loser_rank IS NOT NULL
AND tourney_level != 'D'
AND score NOT LIKE '%W/O%'
AND score NOT LIKE '%RET%'
ORDER BY rank_diff DESC
LIMIT 15;
-- FIRST TRY: USED (loser_rank - winner_rank), WHICH ACTUALLY SHOWED THE OPPOSITE 
-- (BETTER-RANKED PLAYER BEATING A WORSE-RANKED ONE, WHICH IS NORMAL, NOT AN UPSET)
-- FIXED BY FLIPPING TO (winner_rank - loser_rank)
-- ALSO REMOVED DAVIS CUP MATCHES (WEAK TEAMS SKEW THE RANKING GAP) AND WALKOVERS/RETIREMENTS
-- (score LIKE 'W/O' OR 'RET'), SINCE THOSE MATCHES WEREN'T FULLY PLAYED AND AREN'T REAL UPSETS

-- ----------------------------------------------------------------------------------
-- DOES RANKING PREDICT RESULTS BETTER IN GRAND SLAMS THAN IN OTHER TOURNAMENTS?
SELECT
    CASE WHEN tourney_level = 'G' THEN 'Grand Slam' ELSE 'Other tournaments' END AS tournament_type,
    COUNT(*) AS total_matches,
    SUM(CASE WHEN winner_rank < loser_rank THEN 1 ELSE 0 END) AS favorite_won,
    ROUND(100.0 * SUM(CASE WHEN winner_rank < loser_rank THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct_favorite_won
FROM matches
WHERE winner_rank IS NOT NULL AND loser_rank IS NOT NULL
GROUP BY tournament_type;
-- RESULT: THE FAVORITE WINS 70.4% OF MATCHES AT GRAND SLAMS VS 65.0% ELSEWHERE
-- LIKELY EXPLAINED BY THE BEST-OF-5 FORMAT: MORE SETS GIVE THE STRONGER PLAYER MORE TIME
-- TO SHOW THE DIFFERENCE, WHILE BEST-OF-3 LEAVES MORE ROOM FOR UPSETS
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
