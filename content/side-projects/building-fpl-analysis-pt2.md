---
title: "Building FPL Analysis Part 2: How the Algorithm Actually Thinks"
date: 2026-09-03T10:00:00+08:00
draft: false
tags: ["python", "football", "projects"]
showToc: true
---

*This is the second post in a series about a side project I am building: a self-hosted app that tells me who to play in Fantasy Premier League every week. The [first post](https://avradeep.com/side-projects/building-fpl-analysis-pt1/) was the spec: the rules, the stats, and a rough formula. This one is the algorithm as it actually exists in the code, explained from the ground up.*

If you skipped part one, here is all you need: in Fantasy Premier League you pick 15 real footballers with a fake budget, and they earn you points for the things they do in real matches. Goals, assists, clean sheets, bonus. My app's job is to tell me which 15 to own, which 11 to start, and who to captain.

Underneath all of that, the app is trying to answer exactly one question, over and over, for every player in the league:

**How many points will this player score in each of the next six gameweeks?**

Once it has that number for everyone, choosing a team is just arithmetic. So nearly all of the machinery is about producing that number well. I am going to walk through how it gets there in five steps, and I am going to name every variable as it appears, because by the end I want you to be able to read the full formula and know what every piece of it means.

Let's get into it.

---

## Step 1: How good is each team?

Before you can guess how many points a player will score, you need to know how many goals his team is likely to score and concede in a given match. If Haaland plays for a team that creates 20 chances a game, he is going to score more than if he played for a team that creates 5.

The app measures this using **expected goals**, written xG. If you have not met this stat before, here is the idea. Every shot in a match is given a value between 0 and 1 based on how often shots like it go in. A tap-in from two yards might be worth 0.8. A hopeful hit from 30 yards might be worth 0.03. Add up every shot a team took and you get the number of goals they *deserved*, which is a far more honest picture in my opinion.

For each team, the app works out two averages from their match history:

- **Expected goals scored per match** (how many chances they create).
- **Expected goals conceded per match** (how many chances they give up).

Then it compares each to the league average. Say the league average is about 1.4 expected goals per team per match. I call that number **`league_xg`**. It is the yardstick everything else is measured against.

If Liverpool average 2.0 expected goals scored, their **`attack`** rating is 2.0 divided by 1.4, which is about 1.43. In plain words: "43% better than average at creating chances." If a promoted side concede 1.8 per match, their **`defence`** rating is 1.8 divided by 1.4, about 1.29. In plain words: "29% leakier than average." "Leaky" is football slang for a defence that lets in a lot of goals. A leaky defence leaks goals the way a leaky bucket leaks water. So "29% leakier than average" just means "gives up 29% more chances than an average team does.". Defence rating works backwards from what you might expect. A higher number is worse. A team with a defence rating of 1.29 is not 29% better at defending, it is conceding 1.29 times the league average, so it is 29% worse. I clamp both ratings between 0.5 and 1.8 so that one freak result cannot produce a silly number.

One small wrinkle. Promoted teams have no Premier League data yet, so I give them a prior guess instead: `attack` of 0.82 and `defence` of 1.18. I have kept it deliberately a bit worse than average, because promoted teams usually are. Keyword is "usually", although I know this is an issue as everyone following EPL knows, Hull City has blown through that assumption. But let's move on.

Now, to predict how many goals a team will score in a specific fixture, the app multiplies four things together:

```
team_lambda = league_xg * attack (of the scoring team) * defence (of the conceding team) * venue_factor
```

**`venue_factor`** is the home advantage. Home teams get 1.10 (a 10% boost) and away teams get 0.92 (a small penalty).

**`team_lambda`** is the result: the number of goals this team is expected to score in this specific fixture. Lambda is just the conventional letter used for "the average of a count,". So Liverpool at home to that promoted side comes out at 1.4 × 1.43 × 1.29 × 1.10, which is roughly 2.8 expected goals. There is a floor value in place which is set at 0.15 so no team is ever predicted to score literally nothing.

This one number is what the whole rest of the model leans on. The scoring team's `team_lambda` drives goals and assists. The opposing team's `team_lambda` drives clean sheets.

### Why the app keeps old seasons around

At the start of a season there are only one or two matches of data. That is nowhere near enough. One good afternoon would make a mid-table side look like champions. So the app borrows from the previous two seasons, and slowly shifts its trust from the old data to the new as the season goes on.

The variable that controls this is **`current_weight`**, which is simply the number of finished gameweeks divided by 10, capped at 1. Before a ball is kicked it is 0, so the model is 100% history. After five gameweeks it is 0.5, so history and this season count equally. By gameweek 10 it is 1, and the old seasons are dropped entirely and we start giving current season data more weight.w

---

## Step 2: How productive is each player?

Now every player gets what I think of as a rate card: for every 90 minutes he is on the pitch, how much of each point-scoring thing does he do? Each of these is worked out from his match history using the same `current_weight` blend as the teams.

- **`xG90`**: expected goals he personally produces per 90 minutes. This is his own shots, not his team's.
- **`xA90`**: expected assists per 90 minutes. The chances he creates for others.
- **`saves90`**: saves per 90 minutes. Goalkeepers only.
- **`bonus90`**: bonus points per 90 minutes.
- **`yellow90`** and **`red90`**: cards per 90 minutes.
- **`defcon_hit_rate`**: how often he clears the defensive contribution bar. *(If you missed part one, defensive contribution is a scoring rule where defenders get 2 points for reaching 10 tackles, blocks, interceptions and clearances in a match, and midfielders and forwards get 2 points for reaching 12 (with ball recoveries counted too). The app measures this as the share of his 60-plus-minute appearances where he cleared the bar. This stat only exists from 2025/26 onward, so older seasons are excluded from it entirely, otherwise they would drag every player's rate down to zero.)*

There is one adjustment on top. Some players consistently score more than their expected goals say they should, and some consistently score less. I capture that with a **`conversion`** factor: the ratio of his actual goals to his expected goals, valued between 0.85 and 1.15, and only switched on once he has at least 5 expected goals of history. So the best finisher in the league gets at most a 15% bonus on his goal projection. That is deliberately small. Finishing luck is one of the noisiest things in football and I do not want the model chasing it.

A new signing with no match-level history falls back to FPL's own published per-90 numbers. It is worse data, but it is better than nothing.

---

## Step 3: Will he actually play?

This is the step that ruins most fantasy weeks. It tracks two separate probabilities:

- **`p_available`**: the chance he is fit and in the matchday squad at all.
- **`p_start`**: the chance he is in the starting eleven.

`p_available` is always at least as high as `p_start`, because you cannot start if you are not in the squad. The gap between them is the chance he comes off the bench for a short cameo. A fit squad rotation player might have a `p_available` of 0.95 but a `p_start` of only 0.4.

Where do the numbers come from? If the official FPL site marks a player injured, suspended or unavailable, both go to zero. If it says "75% chance of playing," both are capped at 0.75. Otherwise `p_start` comes from how many of his last five matches he started. A brand new player with no history at all gets `p_start` capped at 0.5, because I have no idea if he is a regular and I would rather assume rotation risk than be burned by it.

From these two, the app derives a handful of minutes-related numbers that feed the final formula:

- **`start_minutes`**: how long he typically plays when he starts, from his history, clamped between 45 and 90.
- **`p_cameo`**: the chance of a substitute appearance. This is the gap between `p_available` and `p_start`, scaled down by how often he actually gets used off the bench (capped at 60%).
- **`expected_minutes`**: `p_start × start_minutes + p_cameo × 18`. If he starts he plays his usual minutes, if he comes on it is about 18 minutes, weight each by its probability.
- **`minutes_share`**: `expected_minutes ÷ 90`. What fraction of a full match he is likely to be on the pitch for. This is the number that scales every per-90 rate from Step 2 down to what he will actually do in this match.
- **`p60`**: the chance he plays 60 or more minutes, which is the threshold for the second appearance point and for clean sheet points. Currently `p_start × (start_minutes − 25) ÷ 65`, clamped between 0 and 1. Keep that formula in mind, it comes back in the shortcomings section.
- **`p_any`**: the chance he plays at all, which is `p_start + p_cameo`, capped at 1.

---

## Step 4: The final formula

Now everything gets combined into a points estimate for one player in one fixture. It is built up piece by piece, in exactly the order FPL awards points, so that I can open any recommendation and see which line drove it. Here is the whole thing:

```
xPts = 2 * p60 + 1 * (p_any - p60)                                   # appearance
     + xG90 * minutes_share * attack_scale * conversion * h2h * GOAL_PTS    # goals
     + xA90 * minutes_share * attack_scale * h2h * 3                  # assists
     + exp(-opponent_lambda) * CS_PTS * p60                           # clean sheet
     + defcon_hit_rate * 2 * p60                                      # defensive contribution
     + saves90 * minutes_share / 3                                    # saves (GK only)
     + bonus90 * minutes_share                                        # bonus
     - E[floor(conceded / 2)] * p60                                   # goals conceded (GK/DEF only)
     - yellow90 * minutes_share - 3 * red90 * minutes_share           # cards
```

Let me take it line by line.

**Appearance.** You get 2 points for 60-plus minutes and 1 point for anything less. So: 2 times the chance of reaching 60, plus 1 times the chance of playing but not reaching 60.

**Goals.** Start with his personal goal rate `xG90`. Scale it by `minutes_share` because he only scores while he is on the pitch. Then scale it by **`attack_scale`**, which is this fixture's `team_lambda` divided by `league_xg`. In words: "how good is this fixture for scoring compared to an average fixture?" A strong team at home to a leaky defence might get 1.6, a weak team away to a good defence might get 0.6. Multiply by his `conversion` factor, by a tiny head-to-head factor I will explain in a second, and finally by **`GOAL_PTS`**, the points a goal is worth for his position: 4 for a forward, 5 for a midfielder, 6 for a defender or keeper.

**Assists.** Same shape, using `xA90`, and an assist is worth 3 for everyone.

**Clean sheet.** The chance the opponent scores zero. If the opposing team's expected goals in this fixture (**`opponent_lambda`**, which is just their `team_lambda`) is 0.8, then the maths of goal counts says they will blank about 45% of the time. That `exp(-opponent_lambda)` is the formula for exactly that: it turns an expected number of goals into a probability of zero goals. Multiply by **`CS_PTS`** (4 for keepers and defenders, 1 for midfielders, 0 for forwards) and by `p60`, because you only get clean sheet points if you played 60 minutes.

**Defensive contribution.** His hit rate times 2 points, times `p60`, since you almost never clear the bar in a short cameo.

**Saves.** Keepers get 1 point per 3 saves.

**Bonus.** His bonus rate scaled by minutes.

**Goals conceded.** Keepers and defenders lose 1 point for every 2 goals conceded. The app works out the expected size of that penalty from the opponent's `team_lambda`.

**Cards.** Minus 1 per yellow, minus 3 per red.

The whole thing is floored at zero, because you cannot project negative points for a player who might not play.

### The head-to-head factor

**`h2h`** deserves its own paragraph because it is the one term I put in for humans rather than for the maths. It compares a player's average points against *this specific opponent* over the last two seasons to his position's average, and nudges the projection by at most 5% either way. It needs at least 3 past meetings, and even then it is shrunk towards 1.0 unless there are 8 or more. It is in there because every FPL manager I know looks at "he always scores against Spurs," and I wanted the app to acknowledge that instinct without ever leaning on it. Three matches is nowhere near enough evidence to move a number more than a whisker.

### Quick reference

| Variable | What it means | Where it comes from |
| --- | --- | --- |
| `league_xg` | Average expected goals per team per match across the league | Team match history |
| `attack` | How good a team is at creating chances vs average (1.0 is average) | Team xG scored ÷ `league_xg` |
| `defence` | How leaky a team is vs average (above 1.0 is leaky) | Team xG conceded ÷ `league_xg` |
| `venue_factor` | Home advantage: 1.10 at home, 0.92 away | Fixed |
| `team_lambda` | Expected goals for a team in one specific fixture | `league_xg × attack × defence × venue_factor` |
| `opponent_lambda` | The other team's `team_lambda` in the same fixture | As above |
| `current_weight` | How much to trust this season vs the last two (0 to 1) | Finished gameweeks ÷ 10 |
| `xG90`, `xA90` | Player's own expected goals and assists per 90 minutes | Player match history |
| `saves90`, `bonus90` | Saves and bonus points per 90 minutes | Player match history |
| `yellow90`, `red90` | Cards per 90 minutes | Player match history |
| `defcon_hit_rate` | Share of 60+ minute appearances that cleared the DefCon bar | Player match history (2025/26 onward only) |
| `conversion` | Finishing skill adjustment, 0.85 to 1.15 | Goals ÷ xG, damped |
| `p_available` | Chance of being fit and in the squad | Official FPL status, crowd layer |
| `p_start` | Chance of starting | Last 5 matches, capped by official status |
| `start_minutes` | Typical minutes when he starts | Player match history |
| `p_cameo` | Chance of a substitute appearance | Gap between `p_available` and `p_start` |
| `expected_minutes` | Likely minutes on the pitch | `p_start × start_minutes + p_cameo × 18` |
| `minutes_share` | Fraction of a full match he will play | `expected_minutes ÷ 90` |
| `p60` | Chance of playing 60+ minutes | Derived from `p_start` and `start_minutes` |
| `p_any` | Chance of playing at all | `p_start + p_cameo` |
| `attack_scale` | How good this fixture is for scoring vs an average one | `team_lambda ÷ league_xg` |
| `h2h` | Tiny nudge for past record vs this opponent, 0.95 to 1.05 | Last 2 seasons vs this team |
| `GOAL_PTS` | Points per goal by position: GK 6, DEF 6, MID 5, FWD 4 | FPL rules |
| `CS_PTS` | Points per clean sheet by position: GK/DEF 4, MID 1, FWD 0 | FPL rules |

---

## Step 5: Six weeks ahead, then let the optimiser choose

Step 4 gives one number per player per fixture. The app runs it for the next six gameweeks. If a team plays twice in a gameweek, both matches are added. A gameweek that has already started is excluded, because you cannot buy into it.

Weeks further away count for a bit less. Each week out, the projection is multiplied by 0.9 again, so next week counts fully, the week after at 90%, the one after that at 81%, and so on. A projection six weeks out is less reliable, and there is a good chance I will have transferred the player out by then anyway. Add the six weighted weeks up and you get each player's **horizon value**.

Now a piece of code called an **optimiser** takes over. If you have never met one, here is the idea: you give it a set of rules that cannot be broken (15 players, budget, at most 3 per club, valid formation, 2 keepers, 5 defenders, 5 midfielders, 3 forwards) and a score for every option, and it finds the combination with the highest total score. It does this without literally trying every combination, which would take longer than the season. Think of it as a very fast, very thorough assistant that never forgets a rule.

The app uses it three times a week:

**Best eleven.** From the 15 I own, pick the 11 with the highest next-gameweek `xPts` that form a legal formation. Bench order is the reserve keeper first, then the outfield players by `xPts`.

**Captain.** In part one I argued you captain the ceiling, not the average, because doubling rewards variance. So the app calculates `ceiling = xPts + points_std`, where **`points_std`** is how much the player's per-match points have bounced around historically. It shows both the highest-average and highest-ceiling picks and flags when they disagree.

**Transfers.** For 0, 1 and 2 transfers, find the best legal 15 that differs from my current 15 by exactly that many players, maximising horizon value. Bench players count at 12% weight, because bench points are worth something (an injury to a starter brings them on) but not much. The money rule is the real one: whatever I buy has to cost no more than my bank plus what I get for selling. A paid transfer costs 4 points, and the app only recommends one when the horizon gain minus 4 is still positive. Otherwise it shows the best hit-free plan.

Free transfers are not published by the FPL API, so the app reconstructs them from my transfer history: plus one per gameweek, banked to a maximum of five, minus the ones I actually used.

Before the first deadline of the season, when nobody owns anyone yet, the same optimiser runs with a 100.0m budget and every player marked as a "buy." So it drafts a full 15 from scratch, then switches to real transfer planning once my picks exist.

### Wisdom Of The Crowd

This is the idea that the average or collective answer of a large group of people is often more accurate than the prediction of any single expert. Alongside all of the calculation, once a day the algorithm gathers injury flags, news articles and YouTube transcripts from five FPL creators I have ranked by how much I trust them, and asks a language model to read them and fill in a fixed form per player mentioned: fit, doubtful, injured or suspended, a guess at starting chance, and how positive or negative the chatter is.

The language model **never picks a team**. Its answers can only *lower* a player's numbers. An injury or suspension report sets `p_start` and `p_available` to zero. A doubt caps them at the lowest of the model's value, the crowd's guess and the official percentage. Negative chatter can shave up to 10% off `xPts`, but only if two independent sources agree. Positive chatter cannot raise anything. Every adjustment is stored with its reason, so the app can show me "1.45 → 0.00 because source X reported he came off at half time with a hamstring."

### Checking itself

After every gameweek finishes, the app compares what it predicted for my eleven against what actually happened, and records the average error and whether its captain pick was in fact my top scorer.

---

## But it is not working!

I said in part one that I would rather write my assumptions down and be wrong in public than hide them inside the code. So here it is. The overall design I think is good, building points up piece by piece from expected goals, minutes and fixtures is the same approach the best public FPL models take. But there is one outright mistake, and several places where my thinking is not correct than the rest. In order of how much I think they matter.

### 1. Team attacking strength is counted twice

Look at the goals line in the formula. A player's goal points are his `xG90`, times `minutes_share`, times `attack_scale`. And `attack_scale` is `team_lambda ÷ league_xg`, which contains his own team's `attack` rating.

Here is the problem. His `xG90` was measured *while playing for that team*. A striker at Liverpool already has a high `xG90` precisely because Liverpool create lots of chances and feed him. So his team's attacking strength is already inside his rate card. Multiplying by his team's `attack` rating on top of that counts the same thing twice.

The effect is that attackers at strong teams get inflated and attackers at weak teams get deflated, beyond what is fair. And since transfer recommendations are driven by these numbers, the model has been quietly nudging me towards big-club attackers all along.

I think the fix is to make `attack_scale` capture only what *changes* from one fixture to the next for this player: the opponent's `defence` rating and the `venue_factor`. Leave his own team's `attack` out, because it is already in his rate. The one exception is a player who has just changed clubs. His old `xG90` came from his old team, so for him, the ratio of new team's `attack` to old team's `attack` is a legitimate adjustment.

### 2. Minutes prediction is the thinnest

Every serious FPL model finds the same thing: predicting whether someone plays, and for how long and this causes more error than predicting goals. My Step 3 uses only "how many of the last five did he start" and a rough formula for `p60`.

But I think that `p60` formula is the weaker of the two. It says a player who averages 68 minutes per start only reaches 60 minutes about two thirds of the time. In reality such a player almost always reaches 60. He is being subbed on 68, not on 55. I believe the better approach will be to count it directly from history: of the matches he started, what share did he actually reach 60 minutes in?

The other cheap win is fixture congestion. The single most predictable cause of rotation is a midweek European or cup match. Any league fixture that follows one within three days should knock `p_start` down for squad players at those clubs. The fixtures data is already in the database, I just have not used it for this.

### 3. Team ratings do not account for who the team has played

A team's expected goals average looks great if their first five matches were all against weak sides. My `attack` and `defence` ratings do not adjust for that.

The standard fix is not complicated. Calculate the ratings once, then go back through each match and re-weight that match's expected goals by the opponent's rating (discount a big xG total against a weak defence, credit one against a strong defence), recalculate the ratings, and repeat a few times until the numbers stop moving. It also gives me a principled way to set the promoted-side priors of 0.82 and 1.18, by looking at how promoted teams have actually performed in the historical data instead of picking numbers I liked the look of.

### 4. Bonus points correlation

Bonus points are awarded by a system that gives heavy credit for goals, assists and clean sheets. So a player in a fixture where he is likely to score is also likely to pick up bonus. The two rise and fall together. My `bonus90 × minutes_share` line gives every player the same bonus rate regardless of fixture, so it undercounts bonus in good fixtures and overcounts it in bad ones.

The fix: from the history data, work out how much bonus a goal, an assist and a clean sheet are typically worth for each position, and add that on top of a smaller flat rate. Then bonus scales with the fixture the way it does in real life.

### 5. The self-check is measuring the wrong thing

"How far off was the prediction for my eleven?" is dominated by luck. One unexpected hat-trick swamps everything. It cannot tell me whether the model is any good.

Two better checks. First, across *all* players each gameweek, did the ones the model ranked highly actually score more than the ones it ranked low? That is a rank comparison and it is much less sensitive to one big result. Second, does the model beat a dumb baseline? FPL publishes its own expected points number for every player. If my model does not outperform that, or a plain "average of the last six weeks," then all of this complexity is not earning its place. I want to know that before I spend any more evenings tuning the hand-picked constants.

*Speaking of which: the bench weight of 0.12, the 0.9 horizon discount, the 5% cap on `h2h`, the 0.5 cap for unknown players, the promoted-side priors. None of these were fitted to anything. They are all numbers I thought sounded right.*

### 6. One source can zero a player

Sentiment needs two independent sources to move `xPts` by even 10%. But one creator saying "injured" zeroes a player outright, even when the official flag says 75%. That happened with Doku early this season: FPL had him at 75%, one source said he came off at half time and was out for weeks, and the app took him from 1.45 to 0.00. It was almost certainly right. But it did that on one source, and the asymmetry bothers me.

The clean fix is to make it a probability rather than a switch. One source saying "injured" should set `p_available` to the lower of the official figure and one minus that source's confidence, not to zero. Only two sources agreeing, or my top-ranked creator alone, should go all the way to zero.

I am also reconsidering the rule that positive information can never raise a number. A trusted creator's predicted lineup from a press conference is the most valuable thing the crowd layer could contribute, and right now the layer is only ever a brake. Letting lineup news raise `p_start` up to a cap, from the top tier of sources only, is on the list.

---

## What's Next

The fix for all of the above is basically to stop collapsing the afternoons into an average and instead play the gameweek out in app a few thousand times, using the model's own probabilities, and keep every result. Once you have the spread, the captaincy question answers itself. If Bruno is the top scorer in 48% of runs and Haaland in 37%, the app should say so and let me choose, rather than pretending it knows. The same applies to transfers. Three plans that come out top 35%, 33% and 32% of the time are a menu, not a recommendation.

But I am not sure how I implement this.

If you are building anything similar, or you think I have got point 1 wrong, reply and let me know.

---

## References

- [Building FPL Analysis Part 1: The Data Model Behind Every Decision](https://avradeep.com/side-projects/building-fpl-analysis-pt1/) (part one of this series)
- [vaastav/Fantasy-Premier-League](https://github.com/vaastav/Fantasy-Premier-League) for the historical match-level data
- [PuLP](https://coin-or.github.io/pulp/) and the CBC solver, which is the optimiser the app uses