---
layout: page
title: Building an NHL Playoff Pool Optimizer with Python
description: An end-to-end sports analytics project combining data engineering, statistical modelling, Monte Carlo simulation, and mixed-integer linear programming to optimize an NHL playoff pool roster.
img:
redirect:
importance: 1
category: ""
related_publications: false
toc: 
    sidebar: left
---

<!-- GitHub Repository -->
<p>
  <a href="https://github.com/mattrflew/nhl-playoff-pool-optimizer" target="_blank" rel="noopener">
    <i class="fa-brands fa-github fa-2x"></i>
    &nbsp; View Code on GitHub
  </a>
</p>

<!-- INTRO -->

# Designing an Overly Complex Solution to Win a Family Competition 

## Introduction

Every year, my family runs an NHL Stanley Cup playoff pool. Participants draft a fixed roster of players before the playoffs begin, and earn points based on their chosen player's performance throughout the postseason.

At first glance, choosing the best roster seems simple: pick the league's best players from the best teams. In reality, team performance is inherently hard to predict and individual player performance depends heavily on how long their teams survive in the playoffs. For example, a star player whose team is eliminated in the first round is worth less than a middle of the pack player who makes it to the finals. This makes roster selection an interesting problem involving prediction, uncertainty, and optimization.

Naturally, this called for an unnecessarily complicated solution.

This project is a system with the objective to:

1. Simulate playoff bracket outcomes (Elo Ranking & Monte Carlo Methods)
2. Estimate each player's expected playoff value (Statistical Modelling)
3. Select an optimal fantasy roster (Linear Programming/Optimization)
4. And most importantly, win the playoff pool (Bragging Rights).

While doing this might break the spirit of a fun family competition, it provides for an interesting project. I have also consistently done poorly in this playoff pool, so I clearly need to change my old strategy of hurriedly choosing players the day before the deadline.


## The Optimization Problem
The rules of the playoff pool are quite simple. Before the start of the Stanley Cup Playoffs, each participant selects a roster which cannot be changed for the remainder of the tournament. The players on the roster obtain fantasy points, which are summed together and the highest fantasy score wins the pool. 

### Roster Constraints
- 15 skaters total
  - 9 forwards
  - 6 defencemen
- 2 goalies

### Scoring

#### Skaters
- Goal: **2 Points**
- Assist: **1 Point**

#### Goalies

- Win: **1 Point**
- Assist: **1 Point**
- Shutout: **2 Points**

Maximizing fantasy points requires answering three questions:
1. Which teams are most likely to advance through the playoffs?
2. Given those playoff outcomes, how many fantasy points is each player expected to score?
3. Which combination of players maximizes the expected score while satisfying the roster constraints?

These questions naturally divide the project into major components which are discussed throughout this article.

## System Overview
The system transforms raw NHL data (player statistics, team records, game results, etc.) into an optimized fantasy roster, given the above objectives and constraints. Each stage builds on the previous one, creating end-to-end analytics pipeline. A high-level overview of the system is represented in the following diagram.

![System Overview](/assets/projects/nhl_playoff_pool/system_overview.svg)

# Data Collection, Engineering, & Processing

The NHL provides two public APIs to access player statistics, game results, standings, schedules, and other information. While these APIs are well suited for individual queries, they appeared less convenient for large scale historical analysis, where thousands of requests must be made in a reproducible manner. This motivated the development of a dedicated data collection pipeline. 

Fortunately, there are two useful community-maintained documentations of these APIs:

- https://github.com/Zmalski/NHL-API-Reference
- https://gitlab.com/dword4/nhlapi/-/blob/master/new-api.md

Rather than interacting with API endpoints directly throughout the project, I developed a small Python library to provide a clean interface for data collection. The library is responsible for:

- Constructing API endpoints
- Managing HTTP requests and sessions
- Automatic retries
- Caching raw API responses locally
- Returning structured JSON for downdtream processing

This allows the analysis code to remain concise and modular to fit the needs of the project. 

For example, retrieving the statistics for an entire team's roster requires only a few lines of code:

```python
from nhl_pool.dataset.api import NHLAPI

api = NHLAPI()

params = {
    "abbrev": "MTL",
    "season": 20252026,
    "season_type": 2
}

data = api.get_team_roster_stats(params)
```
The library first checks if the requested data already exists in a local cache. If so, the cached response is returned immediately. Otherwise, the request is sent to the NHL API, the response is stored as a compressed JSON file, and the downloaded data is returned. The caching system was created to avoid unnecessary network requests, as it can be timely collecting a large number of records.

The processing of data was handled by scripts, which take the raw JSON files and the nested API responses are flattened and standardized into tables such as:

- games.csv
- skaters.csv
- goalies.csv
- standings.csv

These processed datasets form the foundation for the remainder of the project. 

For more detail on the implementation, the source code is available on [GitHub](https://github.com/mattrflew/nhl-playoff-pool-optimizer/tree/main/nhl_pool/dataset).


# Predicting the Playoffs

The next challenge is estimating how the Stanley Cup Playoffs are likely to unfold. Since players can only accumulate fantasy points while their team remains in contention, predicting team performance is a critical component of estimating each player's expected value. 

To simulate the playoffs, we require a measure of team strength. While regular season standings provide a rough indication of performance, they are not suitable for predicting individual games. Instead, this project uses the Elo rating system, a method for estimating relative strengths of competing teams based on historical game results. These ratings can then be converted into win probabilities when comparing two teams, providing the foundation for simulating playoff brackets and estimating each team's likelihood of advancing through the tournament.

## Measuring Team Strength via Elo Rating

A notebook detailing the Elo implementation is available [here](https://github.com/mattrflew/nhl-playoff-pool-optimizer/blob/main/notebooks/04_Elo_rating.ipynb). 

### The Elo Rating System

The Elo rating system is a method for calculating the skill or strength of a player/team relative to their competitors. The difference in ratings between two teams can be a predictor of the outcome of the game. Two teams with equal rating playing each other are expected to have an equal number of wins. A team with a higher rating is more likely to win against a lower rated opponent.

At the start of a season, each team will be initialized to the same rating, $R$. 

#### Algorithms
The basic Elo system is defined by two algorithms.

The expected probability for team A, $E_A$, to win given the ratings of teams $R_A$ and $R_B$ is:

$$
E_A = \frac{1}{1 + 10 ^ {\frac{R_B - R_A}{400}}},
$$

and similar for $E_B$. The factor of 400 is arbitrary and standard in ELO systems.

Given the actual outcome for team $A$, $S_A$
$$
S_A = \begin{cases} 
1 & \text{if Player } A \text{ wins}  \\
0 & \text{if Player } A \text{ loses}  \\
\end{cases},
$$

and  
$$
S_B = 1 - S_A,
$$

the rating of each team can be updated as:
$$
R_A \gets R_A + K(S_A - E_A),
$$

and similarly for team $B$. The K-factor $K$ is discussed below.

#### K-Factor
The K-factor can be thought of as the sensitivity of the updates. A greater K-factor will lead to bigger swings in ratings and the opposite for smaller K-factors. In some systems you could have a variable K-factor if more than a certain number of games are played, since at the beginning we want a team's rating to stabilize quickly. For our purposes, it is probably satisfactory to just use a constant $K$. It is somewhat arbitrary, but let's use $K=20$.

### Extensions on the Elo System
The formulation above outlines the basic Elo system, but a number of additions can be made to improve it.

#### Home Team Advantage
In sports there is said to be a benefit to the home team when they play at their home location. 

Examining the results of all regular season NHL games between 2010-2025 showed that the home team won 54.10% of the time, demonstrating a slight advantage to the home team. We can try to account for this in the Elo ranking by giving the home team a slight rating boost, $h_{adv}$.

If we have two teams of equal strength, $R_A=R_B$ but the home team gets a boost of $h_{adv}$, then the expected home advantage with probablity is (if $A$ is home team):
$$
E_H = \frac{1}{1 + 10 ^ {\frac{R_B - (R_A+h_{adv})}{400}}} = \frac{1}{1 + 10 ^ {\frac{-h_{adv}}{400}}},
$$

Given the above analysis, $E_H = 0.5423$, so we can solve for $h_{adv}$, giving:

$$
h_{adv} = -400\text{log}_{10}\left(\frac{1}{E_H} - 1 \right)
$$

$$
h_{adv} \approx 30.
$$

This value is used when calculating Elo ratings in regular season games. 

For playoff games over the same time frame the home team only won 51.31% of the time, corresponding to $h_{adv} \approx 11$. 

#### Goal Differential
If a particular game was a blowout for example, we want the rating changes to reflect this as it could indicate one team is significantly stronger than the other. The World Football Elo Ratings have something called a "Goal Index", $G$, where the number of goals is taken into account. Football (soccer) scores are typically lower than hockey, so we'll modify their update logic slightly. 

For all regular season games in the dataset, I calculated the goal differential between the winning and losing teams.

![Goal differential of all regular season NHL games (2010-2025)](/assets/projects/nhl_playoff_pool/goal_differential.png)

We see that approximately 60% of games have a 1-2 goal differential, and up to 75% of values are up to a goal differential of 3. We also see that outliers begin after a goal differential of 6. Considering this analysis, we can implement some logic for our goal index, $G$ with goal differential represented as $N = |\text{goalDiff}|$.

$$
G = \begin{cases} 
1 & \text{if } N \leq 2  \\
1.1 & \text{if } N =3  \\
1.25 & \text{if } 4 \leq N \leq 5  \\
1.5 & \text{if }  N \geq 6  \\
\end{cases}
$$


#### Overtime and Shootout Wins/Losses

A game going to overtime or shootout are indicators that is was a close game, therefore the updates should be smaller in magnitude to reflect this.

We will use a simple factor of $M$ to implement this.

$$
M = \begin{cases} 
1 & \text{if game finish in regular time}  \\
0.75 & \text{if game finish in overtime} \\
0.5 & \text{if game finish in shootout}  \\
\end{cases}
$$

### Putting it All Together

Combining the modifications of the previous sections yields the final Elo formulation used throughout this project. Let $H$ denote the home team, and $A$ the away team.


The expected probabilities are: 
$$
E_H = \frac{1}{1 + 10 ^ {\frac{R_A - (R_H+h_{\text{adv}})}{400}}} \quad \text{and} \quad E_A = 1-E_H,
$$

The game outcomes are represented by:

$$
S_H = \begin{cases} 
1 & \text{if team } H \text{ wins}  \\
0 & \text{if team } H \text{ loses}  \\
\end{cases} \quad \text{and} \quad S_A = 1 - S_H.
$$

The rating updates are:

$$
R_H \gets R_H + KGM(S_H - E_H),
$$

$$
R_A \gets R_A + KGM(S_A - E_A),
$$

where $h_{adv}$, $G$, and $M$ are the home-ice, goal differential, and overtime/shootout modifiers defined previously.

### Calculating Elo ratings


## Simulating Playoff Brackets with Monte Carlo Methods

# Estimating Player Value

# Optimizing the Roster

# Results

# Future Work