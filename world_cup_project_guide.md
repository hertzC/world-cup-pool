# ⚽ Family World Cup 2026 Prediction Pool — Project Documentation

Welcome to the official documentation for the **Family World Cup Prediction Pool App**. This document provides a detailed overview of the system architecture, custom rule mechanics, cloud infrastructure setup, and steps for managing the competition from kickoff to the final whistle.

---

## 📋 1. Project Objective & Core Concept

The goal of this project is to create a private, real-time web application tailored for a family of four (**Hertz, Joyce, Jeremy, and Winston**) plus any dynamic guests. The application allows family members to submit score predictions for all **72 Group Stage matches** of the 2026 World Cup directly from their personal devices (iPads and smartphones).

### Key Architectural Highlights
* **Zero-Server Infrastructure:** Built as a lightweight Single Page Application (SPA) using HTML5, JavaScript (ES6+), and styled entirely with Tailwind CSS.
* **Real-Time Data Layer:** Powered by a cloud-hosted Supabase database instances with instant state synchronization via client-side REST APIs.
* **Device Independence:** Optimised with strict viewport controls for touch targets, making it cross-compatible across modern iOS (Safari PWA native-wrapper) and Android ecosystems.

---

## 🧮 2. The Custom Multi-Tier Scoring System

Predicting the exact scoreline of high-stakes football matches is notoriously difficult. To keep the competition highly interactive, balanced, and engaging for children, a custom mathematical rule system was implemented to award fractional points based on different levels of accuracy.

When a parent closes a match inside the scorekeeper panel, the system evaluates all submissions using the following hierarchy:

### Tier 1: Exact Match Dominance (5 Points)
Awarded if and only if the predicted score matches the actual final score precisely.
$$\text{If } P_{\text{home}} = A_{\text{home}} \text{ and } P_{\text{away}} = A_{\text{away}} \implies \text{Points} = 5$$

### Tier 2: Definitive Match Outcome (1 Point)
Awarded if the participant correctly guessed the general direction of the match (Home Win, Away Win, or a Draw) but missed the exact goal count.
* **Home Win Criteria:** $A_{\text{home}} > A_{\text{away}}$ and $P_{\text{home}} > P_{\text{away}}$
* **Away Win Criteria:** $A_{\text{away}} > A_{\text{home}}$ and $P_{\text{away}} > P_{\text{home}}$
* **Draw Criteria:** $A_{\text{home}} = A_{\text{away}}$ and $P_{\text{home}} = P_{\text{away}}$

### Tier 3: Individual Team Accuracy Bonus (1 Point)
Awarded if the participant precisely nailed the exact number of goals scored by *at least one* of the competing teams, independent of the match outcome.
$$\text{If } P_{\text{home}} = A_{\text{home}} \text{ OR } P_{\text{away}} = A_{\text{away}} \implies \text{Points} = +1$$

### Point Evaluation Examples
The table below illustrates how different predictions are graded against a hypothetical actual match result of **Spain 2 - 1 Cape Verde**:

| Participant | Prediction | Tier 1 (Exact) | Tier 2 (Outcome) | Tier 3 (Team Score) | Total Points Given | Reason for Breakdown |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Hertz** | `2 - 1` | ✅ Match | — | — | **5 pts** | Maximized Tier 1. Caps at 5 points maximum. |
| **Joyce** | `3 - 1` | ❌ Miss | ✅ Win | ✅ Away Score (`1`) | **2 pts** | Correctly guessed a Spain Win (1pt) + Cape Verde's exact goals (1pt). |
| **Jeremy** | `2 - 0` | ❌ Miss | ✅ Win | ✅ Home Score (`2`) | **2 pts** | Correctly guessed a Spain Win (1pt) + Spain's exact goals (1pt). |
| **Winston** | `0 - 1` | ❌ Miss | ❌ Miss | ✅ Away Score (`1`) | **1 pt** | Missed outcome entirely, but gets a safety point for Cape Verde's single goal. |
| **Guest** | `0 - 0` | ❌ Miss | ❌ Miss | ❌ Miss | **0 pts** | No parameters met. |

---

## 📇 3. Database Schema Structure (Supabase DDL)

The data model uses a standard relational scheme consisting of three strongly decoupled tables linked via foreign key cascades. Row Level Security (RLS) is explicitly disabled via schema instructions to allow seamless client-side writes during local execution.

```
       +--------------------+            +-------------------+
       |   FAMILY_MEMBERS   |            |   MATCH_RESULTS   |
       +--------------------+            +-------------------+
       | name (PK, Unique)  |            | match_label (PK)  |
       | total_score        |            | home_score        |
       +---------+----------+            | away_score        |
                 |                       | is_finished       |
                 |                       +---------+---------+
                 |                                 |
                 +---------------+    +------------+
                                 |    |
                            +----+----+----+
                            | PREDICTIONS  |
                            +--------------+
                            | player_name  |
                            | match_label  |
                            | pred_home    |
                            | pred_away    |
                            +--------------+
```

### Table Definitions & Constraints

#### 1. `family_members`
Tracks the global state of participants and their current running tournament score.
* `id` (Serial, Primary Key)
* `name` (Text, Unique Constraint): Acts as the identification token.
* `total_score` (Integer, Defaults to `0`).

#### 2. `match_results`
Stores the definitive master index of all 72 Group Stage matches.
* `id` (Serial, Primary Key)
* `match_label` (Text, Unique Constraint): Format string (e.g., `Spain vs. Cape Verde`).
* `home_score` / `away_score` (Integer, Nullable): Populated only post-match.
* `is_finished` (Boolean, Default `false`): Flags visibility in dropdown arrays.

#### 3. `predictions`
The central transaction matrix storing the predictions.
* `id` (Serial, Primary Key)
* `player_name` (Text, Foreign Key references `family_members.name` on cascade delete)
* `match_label` (Text, Foreign Key references `match_results.match_label` on cascade delete)
* `pred_home` / `pred_away` (Integer): The values submitted by the player.
* **Unique Composite Constraint:** `UNIQUE(player_name, match_label)` — This protects database sanity by executing an `UPSERT` if a player attempts to change their prediction before a match begins, overwriting their old entry instead of bloating the database with duplicate rows.

---

## ⚙️ 4. Functional Code Capabilities

The front-end website engine (`index.html`) leverages modular JavaScript organised into a four-tab SPA shell. All data is fetched once into in-memory caches on load and re-fetched after any mutation, so every tab stays consistent without additional round-trips.

### Tab Structure

| Tab | Icon | Purpose |
| :--- | :---: | :--- |
| Standings | 🏆 | Leaderboard with medal rankings and a scoring rules reference |
| Matches | ⚽ | Full group-stage match grid with filter pills (All / Upcoming / Finished) and per-player prediction chips |
| Predict | 🔮 | Score submission form; team name labels update dynamically when a match is selected |
| Admin | 🛠️ | Parent scorekeeper console — settle matches and see a per-player point breakdown |

### A. The Data Cache & Render Engine
On application mount (`window.onload → initApp`), the system fires three parallel Supabase queries:
* `family_members` → leaderboard cards, user select dropdown
* `match_results` → match cards (all tabs) and the open-match dropdowns for Predict and Admin
* `predictions` → prediction chips rendered inside every match card

Results are stored in module-level arrays (`allMatchesCache`, `allPredictionsCache`, `allPlayersCache`). Any write operation (submit prediction, settle match, add player) awaits `fetchAllData()` to refresh all caches and re-render the UI atomically.

### B. The Matches Tab
Each match card displays:
* Match label parsed into **Home vs Away** team names.
* A status badge — `🕐 Upcoming` or `✅ H – A` (final score) once settled.
* A chip row showing every family member's prediction. If a prediction exists for a finished match, the chip also shows points earned in the appropriate colour (amber = 5 pts, green = partial, grey = 0 pts).

Filter pills (`All` / `Upcoming` / `Finished`) refilter the local cache client-side with no additional DB calls.

### C. The Client-Side Scoring Pipeline
When the Parent Scorekeeper settles a match, the Admin tab:
1. Marks the match as finished in `match_results` (locks it from the prediction dropdowns).
2. Fetches all predictions for that `match_label` from `predictions`.
3. Runs the Tier 1/2/3 point calculation (`calcPoints`) per player.
4. Updates each player's `total_score` in `family_members` via a read-then-increment pattern.
5. Displays a per-player summary line (e.g. `Hertz: 2–1 → 5pts`) inline in the Admin panel — no generic alert.

If any step fails (e.g. Row Level Security blocking a table), the exact Supabase error message is surfaced in the status box rather than swallowed silently.

### D. Amicable Pair Verification System (Security Wiping)
To prevent accidental database clearance by curious kids tapping around on the iPads, a security wall guards the system initialization routine. It prompts the user for a mathematical password based on **Amicable Numbers**.

#### The Math Behind the Password
Amicable numbers are pairs of orthogonal integers where the sum of the proper divisors of each number is exactly equal to the other number. 
* The smallest known non-trivial amicable pair discovered by ancient mathematicians is **220** and **284**.
* **Divisors of 220:** $1, 2, 4, 5, 10, 11, 20, 22, 44, 55, \text{ and } 110 \implies \text{Sum} = 284$
* **Divisors of 284:** $1, 2, 4, 71, \text{ and } 142 \implies \text{Sum} = 220$

By requiring the validation phrase string **`220284`**, the system unlocks the global admin command parameters. This triggers a total cascade wipe, setting all player scores back to 0, purging the prediction history matrix, and resetting all 72 games back to a fresh unplayed state for complete replayability.

---

## 🚀 5. Maintenance Operations & Deployment

### Quick Deployment Checklist
1. **Database:** Ensure the master creation script has been completely processed within your Supabase SQL window. Verify that the table layouts match without Row Level Security conflicts.
2. **Row Level Security:** Disable RLS on all three tables so the anon key can read and write freely from the browser:
   ```sql
   ALTER TABLE family_members  DISABLE ROW LEVEL SECURITY;
   ALTER TABLE match_results   DISABLE ROW LEVEL SECURITY;
   ALTER TABLE predictions     DISABLE ROW LEVEL SECURITY;
   ```
3. **Configuration Inject:** Confirm your unique `SUPABASE_URL` and alphanumeric public `anon` API token strings are wrapped tightly inside the initialization block of your local code file.
4. **GitHub Pipeline:** Push the production-ready script into a clean public GitHub code repository. Navigate directly to **Settings -> Pages**, route the build trigger to target your active master branch root, and click publish.

### Advancing to the Knockout Stages
Once the Group Stage wraps up on June 27, the remaining teams will enter the Round of 32. Because these match titles cannot be anticipated ahead of time, you can add them dynamically without updating your website code. 

Simply log into your cloud dashboard, open the SQL window, and run a quick injection command to make them visible to your family's iPads instantly:

```sql
INSERT INTO match_results (match_label) VALUES 
('Spain vs. Argentina - Round of 32'),
('Portugal vs. Netherlands - Round of 32');
```

Your system is now ready to run smoothly for the entire month of the tournament! Have fun tracking the predictions with the family!