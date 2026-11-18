# Toronto Tempo – Data Engineer Technical Assessment

## Before You Begin

Thank you for taking the time to complete this assessment. The exercise mimics some of the most immediate and impactful work you would do in this role. It uses data from **Second Spectrum**, the official provider of game and tracking data for the WNBA, alongside the league's public stats data, to build the kind of database that our data scientists, coaches, and front office rely on.

**What we expect**

- **Time:** this should take **no more than 4 hours** in total across all three parts. Please do not spend longer. If you run out of time, stop and tell us what you would have done next; a clear list of gaps and trade-offs is a good answer.
- **Generative AI:** you are welcome to use AI tools, as you would on the job. Please note if, where, and how you used them.&#32;
- **Use your judgment:** feel free to take liberties based on basketball knowledge and intuition, and on data engineering best practices. For example, your tables do not need to exactly match the samples, and you may add columns you believe enhance the product or add context. Where something is ambiguous, make a reasonable assumption and document it.
- **Tools:** you may work in **Python or SQL** (or both).
- **Submission:** a zip file or GitHub repository sent to **lmanis@tempobasketball.com**.

**What we are looking for**

We are looking for **clear code and well-thought-out methods and processes**. We are not looking for perfection. We want to see that you understand the data structure, how to implement the pipeline, and how the data will be used.

---

## Part 1: Build the pipeline and the tables

### The scenario

Assume that for every WNBA game, the provider produces two JSON files:

| File | Contents |
| --- | --- |
| `metadata.json` | Teams, rosters (player ids and names), score, date, venue, officials |
| `basketball-markings.json` | A set of "marking" lists describing what happened in the game (`chances`, `possessions`, `shots`, `picks`, `touches`, `passes`, `rebounds`, and more). Records refer to each other by id. |

Game files are provided in the `data` folder.&#32;

### Your task

Multiple game files are in the data folder. **Y**our job is to process the files and **create** a relational database or set of tables **from scratch**.&#32;

Write a **pipeline** that reads the JSON files and creates the following tables:

1. **`mapping`**: one row per player or team, with its name and ID.
2. **`chances`**: one row per chance.
3. **`shots`**: one row per shot attempt.
4. **`picks`**: one row per pick action.

Required columns are listed under *Output tables* and sample rows are shown under *Sample output*. These tables form a **relational model**: they will be queried together, so **every table must be joinable to the others**. **How you define the tables is up to you**: keys, constraints, data types, indexes, and how you load them.

### Requirements

- **Database:** For purposes of this exercise, any relational database you can run locally at no cost (DuckDB, SQLite, Postgres, etc.), or a set of tables saved/loaded as CSV files is okay.
- **Reproducibili****ty****:** we should be able to run your pipeline from a clean checkout using any notes or instructions you provide.
- **Reliable & t****rustworthy data****:** think about what can go wrong when processing the files (a malformed file, a game file included twice, a run that stops halfway, etc.) and handle what is reasonable. Include checks that would catch a bad or incomplete load, and describe what should happen when one fails.

### Output tables

Think of the below as the minimum required columns. Everything not stated here (keys, constraints, types, indexes, extra helper tables) is your decision; please document it.&#32;

Note: All ids are strings. Player names are abbreviated to first initial and last name (for example `K. Iriafen`). Team names are the full team name from the metadata.

**`id_mapping`: one row per player id or team id**

| Column | Description |
| --- | --- |
| `id` | Player id or team id, as used throughout the markings.&#32; |
| `name` | Player name or team name. |
| `entity_type` | `player` or `team`. |

**`chances`: one row per chance**

| Column | Description |
| --- | --- |
| `id` | Chance id |
| `possessionId` | Possession the chance belongs to |
| `gameId` | Game id |
| `season` | Season year |
| `period` | Period (quarter) number |
| `offTeamId`, `offTeamName` | Team on offense, and its name |
| `defTeamId`, `defTeamName` | Team on defense, and its name |
| `outcome` | How the chance ended (provider code) |
| `ptsScored` | Points scored on the chance |

**`shots`: one row per shot attempt**

| Column(s) | Description |
| --- | --- |
| `id` | Shot id |
| `possessionId` | Possession the shot belongs to |
| `chanceId` | Chance the shot belongs to |
| `gameId`, `season` | Game and season |
| `region` | Court region the shot was taken from |
| `shotClock` | Shot clock (seconds) at the time of the shot |
| `shooterId`, `shooterName` | Shooter id and name |
| `three` | Three-point attempt?&#32; |
| `outcome` | Shot made? |
| `fouled` | Shooter fouled on the attempt? |
| `assisted` | Shot assisted? |
| `distance` | Shot distance in feet |
| `shotType` | Shot category (layup, jumper, etc.) |
| `catchAndShoot` | Catch-and-shoot attempt? |
| `contested` | Shot contested? |

**`picks`: one row per pick action**

| Column(s) | Description |
| --- | --- |
| `id` | Pick id |
| `gameId`, `season` | Game and season |
| `chanceId`, `possessionId` | Chance and possession the pick occurred in |
| `direct` | Whether the pick is a *direct* pick (see definitions) |
| `offTeamId`, `offTeamName` | Team on offense, and its name |
| `defTeamId`, `defTeamName` | Team on defense, and its name |
| `ballhandlerId`, `ballhandlerName` | Player handling the ball |
| `screenerId`, `screenerName` | Player setting the screen |
| `ballhandlerDefId`, `ballhandlerDefName` | Defender of the ballhandler |
| `screenerDefId`, `screenerDefName` | Defender of the screener |
| `bhrDefType` | How the ballhandler was defended |
| `scrDefType` | How the screener was defended |
| `rollPop` | Screener's action after the pick |
| `bhrOutcomes` | What the ballhandler did as a result of the pick |

### Useful Definitions

- **Possession:** A possession is the entirety of when an offensive team has the ball. Possession only changes when the defensive team takes control of the ball. Possessions do not change on offensive rebounds or timeouts/dead balls/violations awarded to the offensive team.
- **Chance:** A chance is a section of a possession. A new chance occurs when the team with the ball changes, when there is a dead ball, or when a shot occurs. One possession may include more than one chance.
- **Shot:** When a ballhandler attempts to score a basket (field goal attempt). Does not include free throws.
- **Pick:** An offensive player, a screener, takes a stationary position to legally block the path of the ballhandler's defender to create attacking opportunities.
- **Direct Pick:** A pick is direct when a player associated with the play (ballhandler or screener) shoots, is fouled, turns the ball over or passes to a player who shoots within one dribble of receiving the ball, is fouled, or turns the ball over. This also includes examples when the ballhandler passes to the screener who then passes to a tertiary shooter.

### Sample output

These examples show what the finished tables might look like. Ids are shortened and some columns are condensed/omitted for readability.

**Table: Shots**

| id | possId | gameId | chanceId | season | region | shooter | shot Clock | three | outcome | foul | assist | dist | shotT ype |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ecba | 1f3f66 | 43f102 | 9918e91 | 2026 | key | K. Iriafen | 21.00 | FALSE | TRUE | FALSE | FALSE | 5.4 | layup |
| 7a63 | 30f162 | 43f102 | a6dc086 | 2026 | middle two | B. Sykes | 8.74 | FALSE | TRUE | FALSE | FALSE | 16.5 | jumper |
| 1d2f | b444d | 43f102 | 3cd5a97 | 2026 | ra | G. Amoore | 20.24 | FALSE | FALSE | TRUE | FALSE | 2.3 | layup |
| dc26 | 3023a | 43f102 | cf2d73d | 2026 | key | M. Mabrey | 18.72 | FALSE | FALSE | FALSE | FALSE | 4.8 | layup |

**Table: Chances**

| id | possId | gameId | season | qtr | offTeamName | defTeamName | outcome | ptsScored |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 9918e | 1f3f66e | 43f102 | 2026 | 1 | Washington Mystics | Toronto Tempo | FGM2 | 2 |
| a6dc7 | 30f162c | 43f102 | 2026 | 1 | Toronto Tempo | Washington Mystics | FGM2 | 2 |
| f4ab1 | e500ad | 43f102 | 2026 | 1 | Washington Mystics | Toronto Tempo | TO | 0 |
| 3cd51 | b444dc | 43f102 | 2026 | 1 | Toronto Tempo | Washington Mystics | FOU\_S | 2 |

**Table: Picks**

| id | possId | gameId | chanceId | season | direct | ballhandler | screener | ballhandlerDef | screenerDef | bhrDefType | scrDefType | rollPop | bhrOutcomes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 9bca | 30f16 | 43f102 | a6dc086 | 2026 | TRUE | B. Sykes | N. Sabally | S. Citron&#32; | K. Iriafen | over | soft | roll | \[FGM, FGM2\] |
| 07d3 | 9d43b | 43f102 | b9b56ae | 2026 | TRUE | B. Sykes | N. Sabally | A. Florez | K. Iriafen | switch | switch | roll | \[PASS, AST0\] |
| 72bf | db57c | 43f102 | 6a652c3 | 2026 | FALSE | K. Rice | S. Austin | B. Sykes | N. Sabally | over | soft | pop | \[PASS\] |
| ec1b | 04b07 | 43f102 | 89bce9a | 2026 | TRUE | S. Citron | K. Iriafen | K. Rice | T. Key | over | none | other | \[TO\] |

---

## Part 2: Add a Second Data Source

### Task 1: Create a new table

We now want to add a table alongside your existing tables, from a **different source**: the WNBA stats API. You will pull data from it using the `nba_api` Python package. This can be treated as a one time load. Here is some code to get you started:

```python
# Import packages (you may need to install nba_api)
import pandas as pd
import nba_api as nba
from nba_api.stats.endpoints import LeagueDashPlayerStats

# Pull 2026 regular season player box score stats 
players = LeagueDashPlayerStats(
    league_id_nullable="10",
    season="2026",
    season_type_all_star="Regular Season",
    measure_type_detailed_defense="Base",
    per_mode_detailed="PerGame",
    timeout=30,
)

# Store result as a dataframe
players_2026 = players.get_data_frames()[0]

# Clean up column names 
players_2026.columns = players_2026.columns.str.lower()

# Select relevant columns 
players_2026 = players_2026[
    ["player_id", "player_name", "age", "gp", "min", "oreb", 
"dreb", "reb", "ast", "tov", "stl", "blk", "pf", "pfd", "pts", "plus_minus"]]
```

*\*If you prefer, or if you run into API issues, you may instead collect a sample of **the data manually from the WNBA stats website. Data source: <https://stats.wnba.com/players/traditional/?sort=PTS&dir=-1>*

### Task 2: Join the data to the Second Spectrum data

Analysts want to combine the box score data with the Second Spectrum event data.

**How can you do this?** After loading the player box score data into a table, use any method or logic you choose to make it joinable to the Second Spectrum data, where player names are in different formats. Then show proof that your method works by running any join between your box score data and the Second Spectrum tables, such as one with `chances`, `shots` or `picks`.&#32;

---

## Part 3: Answer questions with your data

Write **SQL or Python** to answer the following questions using your tables. You do not need to provide the answers, only the code that produces them. State any assumptions you make.

1. Which WNBA team had the highest points per chance in 2026?
2. Who was Toronto's most efficient pick duo (ballhandler + screener) by points per chance (total points scored divided by total chances)?
3. How many points did Toronto score off catch-and-shoot attempts in game id 43f102d6?

---

## What to submit

1. **Code files** (`.py`, `.ipynb`, `.sql` ):
   - **Part 1:** the code that processes the JSON files and creates your database or tables.
   - **Part 2:** the code that loads the box score data, prepares it for the join, and demonstrates a join.
   - **Part 3:** the queries or Python commands that answer the questions.
2. **Your data:** the populated database file **or** the table CSV files, whichever route you took.&#32;
3. **Optional: a README or text file** with your methods, assumptions, decisions, notes and thoughts. It is welcome but not required. If you write one, these are useful to cover:
   - setup and run instructions;
   - the primary key of each table, and how the tables **join** (an ER diagram or a clear list of relationships);
   - the data checks you built and how failures are surfaced;
   - your assumptions, how you used generative AI, and what you would do next with more time
