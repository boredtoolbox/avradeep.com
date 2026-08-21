---
date: '2026-08-21T11:47:43+08:00'
draft: false
title: 'Building FPL Analysis: The Data Model Behind Every Decision'
showtoc: true
tags: ["python", "football", "projects"]
---

*This is the first post in a series about a side project I am building: a self-hosted app that tells me who to play in Fantasy Premier League every week. Before I write a single line of code in later posts, I want to nail down the domain model. If you get the rules and the stats wrong, no amount of clever engineering saves you. So this post is the spec. It is the thing the code has to be faithful to.*

If you have never played Fantasy Premier League (FPL), here is the one-line version: you get a fake budget, you draft real Premier League players, and they score you points based on what they do in real matches. The whole game is a constrained optimization problem wearing a football shirt. That is exactly why it is fun to build software around.

This post will cover three things:

1. The rules the app will not be allowed to violate (the constraints).
2. The stats that actually predict points, and the target values I will filter on.
3. The projection formula that will turn all of it into a single number per player.

Let's get into it.

---

## Part 1: The Rules (the constraint layer)

Think of this section as the schema validation for the whole system. Every recommendation the app makes will have to satisfy every one of these, or it is an illegal move.

### Squad construction

- **Budget:** 100.0m pounds for 15 players.
- **Composition:** 2 goalkeepers, 5 defenders, 5 midfielders, 3 forwards.
- **Club limit:** maximum 3 players from any single club.
- **Starting XI:** a valid formation every gameweek (minimum 1 GK, 3 DEF, 2 MID, 1 FWD).

There is also a club limit that you have to abide by. It is the constraint that stops you from just buying the five best-value defenders if four of them play for the same team. The optimizer will have to carry this as a hard constraint.

### Transfers

- 1 free transfer per gameweek. Extra transfers cost 4 points each from your total points.
- You can bank up to 5 free transfers.
- Rule of thumb: only take a 4-point hit if the incoming player is projected to outscore the outgoing one by more than 4 points over your planning horizon.

A hit is not always "bad", it is a 4-point loan against future returns. Part of the app's job will be to work out whether the loan pays off.

### Captaincy

Your captain scores double. Your vice-captain steps in if the captain does not play. This is the single highest-variance decision you will make each week, and captaincy differences compound viciously over a 38-gameweek season. Two managers with identical squads can finish thousands of ranks apart purely on who they armband.

### Chips

You get eight chips across the season, one of each type per half:

| Chip | Effect |
| --- | --- |
| Wildcard | Unlimited permanent transfers for one gameweek |
| Free Hit | Unlimited transfers for one gameweek only, then the squad reverts |
| Triple Captain | Captain scores 3x instead of 2x |
| Bench Boost | Your bench players' points count toward your total |

The important operational details:

- The first set must be used before the Gameweek 19 deadline. Unused chips do not carry over.
- Only one chip can be active in a given gameweek.
- Free Hit is not available in Gameweek 1.
- Do not hoard chips out of tradition. Play them the moment you get something to gain.

### Scoring essentials

This is the reward function. Everything the projection engine estimates eventually maps back to these numbers:

- **Appearance:** 1 point for playing, 2 points for 60+ minutes.
- **Goals:** 6 for GK/DEF, 5 for MID, 4 for FWD.
- **Assists:** 3 points.
- **Clean sheets:** 4 for GK/DEF, 1 for MID.
- **Defensive Contributions (DefCon):** 2 points for a defender who racks up 10 combined clearances, blocks, interceptions and tackles (CBIT) in a match. Midfielders need 12, and theirs includes ball recoveries.
- **Bonus:** 1 to 3 points per match, allocated by the Bonus Points System (BPS).
- **Negatives:** yellow and red cards, own goals, missed penalties, and goals conceded (GK/DEF lose 1 point per 2 conceded).

### What actually changed for 2026/27

If you built an FPL model in a previous season, here is your migration guide:

- **BPS rebalance.** A player now earns 1 BPS (Bonus Points System) for every 3 clearances, blocks and interceptions, rather than every 2. This deliberately stops dominant centre-backs from double-dipping on defensive actions and narrows the bonus gap between them and attacking full-backs.
- **No more tackled penalty.** Players used to lose BPS every time they were dispossessed by a tackle. That is gone, which quietly helps high-volume dribblers.
- **Goalkeeper saves reworked.** Every save is now a flat value, with a bonus on top for close-range saves and a further bonus for saving a "big chance" (a one-on-one or a penalty). This rewards keepers who face and stop genuinely dangerous shots, not just shot-count merchants.
- **Later lockdown.** Scores now finalize at 09:00 the day after the last match of a gameweek, which gives Opta time to review the data. Practical implication for your code: do not treat live scores as final too early.
- Live points, ranks and mini-league standings now update in real time, and projected bonus appears after 20 minutes of each match.

---

## Part 2: The Stats That Predict Points

Here is the core insight that the entire app will be built on: **goals and assists are noisy, but the underlying numbers that generate them are stable.** A striker who scored 5 from 2.1 expected goals is a regression candidate. A striker on 1 goal from 4.0 expected goals is a buy. If you chase last week's points, you are always a week late. If you chase the underlying stats, you are early.

So the app will not rank players by points. It will rank them by the inputs that produce points. Below is every stat it will track, what it means, and the threshold I will filter on.

### Attacking stats

**xG (Expected Goals).** Each shot gets a scoring probability based on location, angle, body part and assist type. It is the best single predictor of future goals precisely because it strips out finishing luck.

Targets per 90 minutes:
- Premium forward: 0.55 to 0.60+ (elite is 0.8+, which is Haaland-shaped).
- Mid-price forward or attacking mid: 0.35 to 0.50.
- Below 0.25 for any attacker is a red flag, unless assists carry them.

**xA (Expected Assists).** The probability that a player's pass becomes an assist, based on the quality of the chance created. It surfaces creators whose teammates keep missing.

Targets per 90:
- Genuine creative threat: 0.25+.
- Elite: 0.35+.
- Set-piece takers inflate this, which is a feature, not a bug. Reliable dead-ball delivery is bankable output.

**xGI (Expected Goal Involvement = xG + xA).** The best single attacking number for FPL, because the game pays for both halves of it. Rank players by xGI per million of price to find value.

Targets per 90:
- Premium picks: 0.7+.
- Value picks: 0.45+.

**Shots in the box and big chances.** Volume matters as much as quality. A player taking 3+ shots in the box per 90 will eventually score even through a cold streak. "Big chances" (situations where a player is expected to score) are the strongest subset. Target 2+ big chances every 3 matches.

**npxG (non-penalty xG).** Strip penalties out when comparing raw threat, then add penalty duty back as a separate flag. A penalty taker gets roughly 0.15 to 0.2 xG per 90 for free, so the app will maintain a set-piece taker table as its own data source.

### Defensive stats

**xGC (Expected Goals Conceded).** A team-level number that predicts clean sheet probability far better than actual clean sheets, which are lucky in small samples.

Targets per match:
- Under 1.0: clean-sheet machine, back their defenders.
- Over 1.5: avoid the defense regardless of price.

**Clean sheet probability per fixture.** Derived from team xGC versus opponent xG.
- 35%+ makes a defender's fixture attractive.
- 50%+ is captaincy-adjacent territory for attacking defenders.

**Defensive Contributions (DefCon).** Since DefCon points survived into 2026/27 unchanged, the app will track each player's per-match CBIT average.
- A defender averaging 10+ CBIT per match is banking a near-guaranteed 2 points every week. That is a floor-raiser worth roughly 0.5m of value.
- Midfielders need 12+ (CBIT plus recoveries), which points you at the defensive-mid archetype.
- The metric that actually matters is **DefCon reliability**, the share of matches in which a player clears the threshold. Above 60% is genuinely bankable.

**Attacking threat from defenders.** The holy grail is a defender with both clean sheet upside and attacking output. A defender posting xGI per 90 above 0.20 is excellent (overlapping full-backs, set-piece target centre-backs). Given the 2026/27 BPS tweaks, the app will weigh attacking full-backs slightly above centre-backs.

### Goalkeeper stats

- Saves per 90: 3+ is high volume.
- Save percentage: 70%+ is elite.
- Save profile now matters because of the rule change: close-range and big-chance saves earn extra.

The value logic is counterintuitive. The best budget keeper is a good shot-stopper behind a mediocre defense, because volume of saves generates points. The best premium keeper sits behind an elite defense and banks clean sheets. The trap is a keeper behind a bad defense who does not even face many shots: no clean sheets and no save volume.

### Availability and usage

**Minutes and starts.** The most underrated stat in the game. A nailed-on starter at 5 points a game beats a rotation risk at 6 points a start, because the rotation risk keeps handing you a zero. The app will only build the XI around players with 80%+ start probability, and it flags anyone averaging under 70 minutes per start (early substitutions cost you the 60-minute bonus and any late-goal upside).

**Form versus fixtures.** FPL's "form" stat is just points per game over the last 30 days. It is backward-looking, so the app will always cross-reference it against underlying xGI to separate a sustainable level from a hot streak that is about to end.

**FDR (Fixture Difficulty Rating).** FPL ships a 1 to 5 rating, but the app will build its own from team strength models (attack strength times opponent defense strength, adjusted for home and away). It will look 4 to 6 gameweeks ahead, because you want to buy a player before a good run of fixtures, not in the middle of it when everyone else already has.

**Price, value and ownership.**
- Points per million: target 15+ over a season for outfield players.
- Price change momentum: net transfers in and out predict rises and falls, and buying before a rise builds your team value over time.
- Effective Ownership (EO): high-EO players (50%+) are shields, where the risk is in not owning them. Low-EO players (under 10%) with strong underlying stats are differentials that win you rank when they return.

**ICT Index.** FPL's own composite of Influence, Creativity and Threat. It is a fine tiebreaker, but xGI-based metrics predict better, so the app will treat ICT as a secondary signal only.

---

## Part 3: Turning It Into One Number

All of the above is just features. The point of the app will be to collapse them into a single projected score per player, per gameweek, so the optimizer has something to maximize.

Here is the projection formula it will use:

```
xPts = (P(start) * minutes_points)
     + (xG_per90  * goal_points_for_position * finishing_adjustment)
     + (xA_per90  * 3)
     + (P(clean_sheet) * clean_sheet_points_for_position)
     + (P(DefCon_threshold) * 2)
     + expected_bonus
     + expected_save_points        # goalkeepers only
     - expected_negatives          # cards, goals conceded
```

Every term will then be adjusted for opponent strength and home versus away. A striker's xG term gets scaled down against a strong defense and up at home against a weak one. The clean sheet probability falls off a cliff against a high-xG opponent. I am keeping the formula deliberately additive and inspectable, because I want to be able to open any recommendation and see exactly which term drove it.

On top of the projection will sit the optimization layer:

- **XI selection:** maximize total projected points subject to the formation constraints.
- **Captaincy:** here is a subtle one. You do not captain the highest mean projection, you captain the highest *ceiling*. Captaincy doubles your points, so it rewards variance. A safe 6-point floor is a worse captain than a volatile player who projects the same mean but can explode for 20.
- **Transfers:** optimize under budget, the club limit, and free-transfer rules, only taking a 4-point hit when it is net positive across the planning horizon.

And then the chip heuristics, which are basically pre-baked strategy rules:

- **Triple Captain:** a premium attacker at home to weak opposition. Haaland hosting all three promoted clubs in the first half of the season is the textbook window.
- **Bench Boost:** ideally the week right after a Wildcard, when your full 15 is at its strongest.
- **Free Hit:** best saved for blank or double gameweeks, though those are rare before Gameweek 19.
- **Wildcard:** early (Gameweek 3 to 4) to react to new-season information, or after an international break when injuries and price changes have shaken out.

---

## What's Next

That is the domain model. It is opinionated on purpose, because a projection engine is only as good as the assumptions baked into it, and I would rather write those assumptions down and be wrong in public than hide them inside the code.

In the next post I will get into the architecture: a Raspberry Pi running Python, SQLite for local storage, the official FPL API plus a historical dataset for the numbers, and a hybrid setup where an AI model handles the messy natural-language side (injury news, press conferences, what the FPL YouTubers are saying) while Python owns all the hard constraints. That crowd-intelligence layer is the part I am most excited to write about, because getting an AI to *inform* decisions without letting it *make* them turned out to be the interesting engineering problem.

If you are building anything similar, or you just want to argue about whether xGI is really better than ICT, reply and let me know. This series is as much about comparing notes as it is about showing off a side project.

---

## References

The rules, scoring and 2026/27 changes above were checked against the following sources:

- [Premier League: All you need to know about changes to FPL for 2026/27](https://www.premierleague.com/en/news/4679873/all-you-need-to-know-about-changes-to-fpl-for-202627)
- [Premier League: What's happening with FPL chips in 2026/27](https://www.premierleague.com/en/news/4679879/whats-happening-with-fpl-chips-in-202627)
- [Premier League: How and when to use your chips in 2026/27 Fantasy](https://www.premierleague.com/en/news/4362085)
- [Fantasy Football Scout: FPL 2026/27, 5 rule changes and new features announced](https://www.fantasyfootballscout.co.uk/2026/07/20/fpl-2026-27-5-rule-changes-new-features-announced)
- [Fantasy Football Fix: FPL 2026/27 new rules](https://www.fantasyfootballfix.com/blog-index/fpl-2026-27-new-rules/)
- [Fantasy Football Fix: FPL chip strategy 2026/27](https://www.fantasyfootballfix.com/blog-index/fpl-chip-strategy/)
- [FPL Oracle: FPL 2026/27 rule changes explained](https://fploracle.team/blog/fpl-2026-27-rule-changes-explained)
- [All About FPL: 2026/27 season new rules and changes](https://allaboutfpl.com/2026/07/2026-27-fpl-season-new-rules-changes-whats-new-in-fpl/)
- [RotoWire: FPL 2026/27 rule changes, player prices and position changes explained](https://www.rotowire.com/soccer/article/fpl-2026-27-rule-changes-player-prices-and-position-changes-explained-124721)
- [Sporting Tribune: What to know about new rules, features and changes for FPL 2026/27](https://sportingtribune.com/what-to-know-about-new-rules-features-and-changes-for-fpl-2026-27/)
- [FPL India: FPL rules 2026/27 complete guide](https://fpl11india.com/fpl-rules/)
- [Wikipedia: 2026/27 Premier League](https://en.wikipedia.org/wiki/2026%E2%80%9327_Premier_League)

*Historical player data for the model comes from the community dataset at [vaastav/Fantasy-Premier-League](https://github.com/vaastav/Fantasy-Premier-League), which I will cover properly in a later post.*