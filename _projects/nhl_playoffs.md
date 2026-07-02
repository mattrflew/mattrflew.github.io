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

This project is a system with the objective to:

1. Simulate playoff bracket outcomes (Elo Rating & Monte Carlo Methods)
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
- Goal: 2 Points
- Assist: 1 Point

#### Goalies

- Win: 1 Point
- Assist: 1 Point
- Shutout: 2 Points

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

- [github.com/Zmalski/NHL-API-Reference](https://github.com/Zmalski/NHL-API-Reference)
- [gitlab.com/dword4/nhlapi/-/blob/master/new-api.md](https://gitlab.com/dword4/nhlapi/-/blob/master/new-api.md)

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

Examining the results of all regular season NHL games between 2010-2025 showed that the home team won 54.10% of the time, demonstrating a slight advantage to the home team. We can try to account for this in the Elo ratings by giving the home team a slight rating boost, $h_{adv}$.

If we have two teams of equal strength, $R_A=R_B$ but the home team gets a boost of $h_{adv}$, then the expected home advantage with probablity is (if $A$ is home team):
$$
E_H = \frac{1}{1 + 10 ^ {\frac{R_B - (R_A+h_{adv})}{400}}} = \frac{1}{1 + 10 ^ {\frac{-h_{adv}}{400}}},
$$

Given the above analysis, $E_H = 0.5410$, so we can solve for $h_{adv}$, giving:

$$
h_{adv} = -400\text{log}_{10}\left(\frac{1}{E_H} - 1 \right)
$$

$$
h_{adv} \approx 30.
$$

This value is used when calculating Elo ratings in regular season games. 

For playoff games over the same time frame the home team only won 51.31% of the time, corresponding to $h_{adv} \approx 11$. 

#### Goal Differential
If a particular game was a blowout for example, we want the rating changes to reflect this as it could indicate one team is significantly stronger than the other. The [World Football Elo Ratings](https://en.wikipedia.org/wiki/World_Football_Elo_Ratings#Number_of_goals) have something called a "Goal Index", $G$, where the number of goals between the winning and losing teams is taken into account. Football (soccer) scores are typically lower than hockey, so we'll modify their update logic slightly. 

For all regular season games in the dataset, I calculated the goal differential between the winning and losing teams.

![Goal differential of all regular season NHL games (2010-2025)](/assets/projects/nhl_playoff_pool/goal_differential.png)

We see that approximately 60% of games have a 1-2 goal differential, and up to 75% of values are up to a goal differential of 3. Goal differentials greater than 6 are comparitively rare. Based on this empirical distribution, we can implement some logic for our goal index, $G$, as a function of the absolute goal differential, $N$.

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

### Calculating Elo Ratings

At the start of a season, each team will be initialized to the same rating, $R=1500$. 

#### Results
The table below summarizes the ten highest-rated teams according to the Elo system. For comparison, the final regular season league ranking is also shown.

| Team | Elo Rating | Elo Rank | Regular Season Standing |
|:----:|-----------:|:--------:|:---------:|
| CAR | 1617.92 | 1 | 2 |
| BUF | 1613.01 | 2 | 4 |
| COL | 1603.33 | 3 | 1 |
| MTL | 1588.70 | 4 | 5 |
| OTT | 1576.24 | 5 | 9 |
| DAL | 1575.09 | 6 | 3 |
| TBL | 1572.35 | 7 | 5 |
| WSH | 1546.24 | 8 | 12 |
| MIN | 1544.06 | 9 | 7 |
| PHI | 1542.90 | 10 | 10 |

> Elo vs standings figure

> Elo vs points figure

## Simulating Playoff Brackets with Monte Carlo Methods

A notebook detailing the playoff simulation framework is available [here](https://github.com/mattrflew/nhl-playoff-pool-optimizer/blob/main/notebooks/05_Monte_Carlo_bracket_simulations.ipynb). 


We can begin this section by acknowledging how difficult it is to accurately predict playoff bracket winners due to the inherent uncertainty associated with each series. Even under the simple assumptions that every playoff series is a fair coin flip, there are 

- Round 1: 8 series
- Round 2: 4 series
- Round 3: 2 series
- Stanley Cup Playoffs: 1 series

for a total 15 series, giving $2^{15}= 32,768$ possible playoff brackets. 

Of course, playoff series are far from a coin flip-type result. Stronger teams are more likely to advance, and each possible bracket has a different probability of occuring based on the competing team strengths. Rather than attempting to predict a single "correct" bracket, this project instead simulates the playoffs thousands of times using the Elo derived win probabilities for each matchup. 

Aggregating the results of these simulations provides estimates of each team's probability of advancing through every playoff round, ultimately allowing us to estimate the expect number of playoff games played by every team.

### Monte Carlo Methods

At a high level, a Monte Carlo simulation is a computational algorithm that uses repeated random sampling to obtain the likelihood of a range of results of occuring. They are a way to model the probability of different outcomes of a process that can not be easily predicted.

### Simulating One Game
```python
import numpy as 
rng = np.random.default_rng(seed=123)

def elo_compute_win_probability(R_h, R_a, h_adv=30):
    '''
    Given the Elo ratings for the home and away teams (h and a), compute the probability team h wins. 
    h_adv is the home team advantage factor.
    '''
    return 1.0 / (1.0 + 10**((R_a - (R_h + h_adv))/400.0))
    
def simulate_game(R_h, R_a, rng, h_adv=30):
    '''
    Returns a boolean 1 if home team won, otherwise home team lost.
    '''
    
    # Calculate the win probability
    p_home_win = elo_compute_win_probability(R_h, R_a, h_adv=h_adv)
    
    # Random number
    u = rng.random()
    
    return u < p_home_win
```

### Simulating the Entire Playoffs

#### Updating Elo During Simulation
Instead of holding the ratings static, for a playoff simulation we can update Elo ratings (using the update rules explained in a [previous section](#putting-it-all-together)) based on the simulated outcomes on each game in the simulation. This way we can try and capture team strength throughout the playoffs, for example a team going on a postseason hot streak, rather than holding ratings static at the end of the regular season. For subsequent next playoff simulations, the ratings will reset to the team's base ratings to provide a fresh simulation which is independent of previous ones.

#### Results



# Estimating Player Value
Having estimated each team's expected number of playoff games, we can now estimate the expected fantasy value of every player. This completes the predictive component of the pipeline and provides the inputs required for roster optimization.

For simplicity, we assume that each player will maintain their regular season scoring rate throughout the playoffs. While individual performance can certainly improve or decline in the postseason, regular season statistics provide a reasonable basis for expected production. Only statistics from the current regular season are used to estimate player performance. In this project, the primary source of uncertainty is assumed to be the number of playoff games played rather than changes in individual player performance. These simplifying assumptions are considered sufficient for the purposes of this model.

Per the rules of the fantasy hockey pool, as explained in a previous [section](#the-optimization-problem), skaters and goalies have different point systems. 

## Skaters
The scoring for skaters (forwards and defencemen) is defined as:
- Goals: 2 points
- Assists: 1 point

The expected fantasy value, $\mathbb{E}[v_i]$, of skater $i$ is defined as

$$
\mathbb{E}[v_i] = 2\,\mathbb{E}[G_i] + \mathbb{E}[A_i],
$$

where

$$
\mathbb{E}[G_i] = \mathrm{GPG}_i \times \mathbb{E}[N_i],
$$

$$
\mathbb{E}[A_i] = \mathrm{APG}_i \times \mathbb{E}[N_i].
$$

Here, $\mathbb{E}[G_i]$ and $\mathbb{E}[A_i]$ denote the expected number of goals and assists scored by player $i$ in the playoffs, respectively. $\mathrm{GPG}_i$ and $\mathrm{APG}_i$ denote the player's regular season goals and assists per game. $\mathbb{E}[N_i]$ denotes the expected number of playoff games played by player $i$'s team.

## Goalies
The scoring for goalies is defined as:
- Wins: 1 point
- Assists: 1 point
- Shutouts: 2 points

Similarly, the expected fantasy value, $\mathbb{E}[v_i]$, of goalie $i$ is defined as

$$
\mathbb{E}[v_i] = 2\,\mathbb{E}[\mathrm{SO}_i] + \mathbb{E}[W_i] + \mathbb{E}[A_i],
$$

where

$$
\mathbb{E}[\mathrm{SO}_i] = \mathrm{SOPG}_i \times \mathbb{E}[N_i],
$$

$$
\mathbb{E}[W_i] = \mathrm{WPG}_i \times \mathbb{E}[N_i],
$$

$$
\mathbb{E}[A_i] = \mathrm{APG}_i \times \mathbb{E}[N_i].
$$

Here, $\mathbb{E}[\mathrm{SO}_i]$, $\mathbb{E}[W_i]$, and $\mathbb{E}[A_i]$ denote the expected number of shutouts, wins, and assists for goalie $i$ in the playoffs, respectively. $\mathrm{SOPG}_i$, $\mathrm{WPG}_i$, and $\mathrm{APG}_i$ denote the goalie's regular season shutouts, wins, and assists per game. $\mathbb{E}[N_i]$ denotes the expected number of playoff games played by goalie $i$'s team.

## Assumptions
One complication of this methodology is that players can be traded throughout the regular season and may accumulate statistics for multiple teams. For the purposes of this project, a player's regular season statistics are aggregated across all teams they played for, while their playoff expected values are determined using the team on whose playoff roster they ultimately appear. This assumes that a player's scoring ability is independent of the team they played for during the regular season.

# Optimizing the Roster
A notebook detailing the roster optimization implementation is available [here](https://github.com/mattrflew/nhl-playoff-pool-optimizer/blob/main/notebooks/03_baseline_roster_optimizer.ipynb). 

Using the expected player values derived in the previous section, we can formulate the fantasy roster selection as an optimization problem. Before doing so, one practical consideration must be addressed. Since player value is estimated on a per-game basis, players who appeared in a small number of regular season games could have inflated expected values due to small sample sizes. For example, a player who records two points in a single game would appear to have a fantastic scoring rate despite having little evidence that rate is sustainable. 

To reduce the influence of these outliers, only players who appeared in at least 15 regular season games are considered by the optimizer. This threshold is empirical, but it removes many of small-sample anomalies while retaining players who regularly play.  

## Formulating the Mixed-Integer Linear Program

With the player pool defined, the roster selection problem can now be formulated as a mixed-integer linear program.

As outlined in a previous [section](#the-optimization-problem), participants select a fixed roster of players at the beginning of the playoff consisting of:

- 15 skaters total
  - 9 forwards
  - 6 defencemen
- 2 goalies

We define a binary decision variable

$$
x_i =
\begin{cases}
1 & \text{if player } i \text{ is selected,} \\
0 & \text{otherwise.}
\end{cases}
$$

### Objective Function

The objective is simply to maximize the total expected fantasy points accumulated by the roster.

$$
\text{max}\sum_i v_ix_i.
$$

Let $v_i$ denote the expected fantasy value of player $i$, as defined in the [Estimating Player Value](#estimating-player-value) section.

### Constraints
The number of players in each position are our primary constraints, which can be written out as follows:

$$
\sum_i x_i = 17,
$$

$$
\sum_{i \in F} x_i = 9,
$$

$$
\sum_{i \in D} x_i = 6,
$$

$$
\sum_{i \in G} x_i = 2,
$$

where $F$, $D$, and $G$ denotes the respective sets of forwards, defencemen, and goalies.  

Additional strategic constraints can also be added into the optimization model. 

A maximum number of players per team was imposed to encourage a more balanced roster and reduce the impact of inccorect playoff bracket prediction.

$$
\sum_{i \in T} x_i \leq M_T \qquad \forall\ T,
$$

where $T$ denotes the set of players belonging to a given team, and $M_T$ is the maximum number of players permitted from that team. A modelling choice of $M_T=8$ was implemented.

Furthermore, an additional constraint was imposed to prevent the optimizer from selecting more than one goalie from the same team. 

$$
\sum_{i \in G_T} x_i \leq 1 \qquad \forall\ T,
$$

where $G_T$ denotes the set of goalies on team $T$.

Ideally, the goalie selected for the roster would be the starting goalie for that team.

### Solving the Optimization Problem
The optimization model was implemented using SciPy's `scipy.optimize.milp` solver. The expected player values define the objective function, while the roster requirements and strategic constraints are represented as linear equality and inequality constraints. Since `milp` is formulated as a minimization problem, the objective coefficients are simply negated to maximize the expected fantasy value. The complete implementation is available in the accompanying [notebook](https://github.com/mattrflew/nhl-playoff-pool-optimizer/blob/main/notebooks/03_baseline_roster_optimizer.ipynb).

## The Final Roster

Finally, with all of the components of the project combined, running the linear programming optimizer yielded the following roster.

| Position | Player | Team | Expected Games | Value/Game | Expected Points |
|:--------:|:-------|:----:|---------------:|----------:|----------------:|
| **FORWARD** | | | | | |
| F | Nathan MacKinnon | COL | 15.35 | 2.25 | **34.54** |
| F | Martin Necas | COL | 15.35 | 1.77 | **27.16** |
| F | Connor McDavid | EDM | 11.43 | 2.27 | **25.93** |
| F | Leaon Draisaitl | EDM | 11.43 | 2.03 | **23.21** |
| F | Nikita Kucherov | TBL | 10.08 | 2.29 | **23.09** |
| F | Jason Robertson | DAL | 11.80 | 1.72 | **20.30** |
| F | Tage Thomspon | BUF | 13.34 | 1.49 | **19.92** |
| F | Cole Caufield | MTL | 11.11 | 1.72 | **19.08** |
| F | Wyatt Johnston | DAL | 11.80 | 1.60 | **18.86** |
| **DEFENCE** | | | | | |
| D | Cale Makar | COL | 15.35 | 1.32 | **20.26** |
| D | Evan Bouchard | EDM | 11.43 | 1.41 | **16.17** |
| D | Rasmus Dahlin | BUF | 13.34 | 1.21 | **16.11** |
| D | Shayne Gostisbehere | CAR | 13.22 | 1.15 | **15.14** |
| D | Darren Raddysh | TBL | 10.08 | 1.26 | **12.71** |
| D | Lane Hutson | MTL | 11.11 | 1.10 | **12.20** |
| **GOALIE** | | | | | |
| G | Scott Wedgewood | COL | 15.35 | 0.89 | **13.65** |
| G | Brandon Bussi | CAR | 13.22 | 0.92 | **12.20** |
|  |  |  |  | | | 
|  |  |  |  | *Total Expected* | **330.52** |

> **Projected Team Total:** **330.52 fantasy points**

The roster composition by team is summarized below. 

| Team | Players Selected |
|:----:|-----------------:|
| COL | 4 |
| EDM | 3 |
| DAL | 2 |
| BUF | 2 |
| MTL | 2 |
| TBL | 2 |
| CAR | 2 |
| *Total* | **17 Players** |

# Results

With the 2026 Stanley Cup Playoffs complete, we can compare the model's projections against the actual fantasy points scored by the selected roster.

| Pos | Player | Team | Exp. Games | Actual Games | Exp. Points | Actual Points | Prediction Error |
|:---:|:-------|:----:|-----------:|-------------:|------------:|--------------:|---------:|
| **FORWARDS** | | | | | | | |
| F | Nathan MacKinnon | COL | 15.35 | 13 | 34.54 | 22 | -12.54 |
| F | Martin Necas | COL | 15.35 | 13 | 27.16 | 14 | -13.16 |
| F | Connor McDavid | EDM | 11.43 | 6 | 25.93 | 7 | -18.93 |
| F | Leon Draisaitl | EDM | 11.43 | 6 | 23.21 | 13 | -10.21 |
| F | Nikita Kucherov | TBL | 10.08 | 7 | 23.09 | 7 | -16.09 |
| F | Jason Robertson | DAL | 11.80 | 6 | 20.30 | 13 | -7.30 |
| F | Tage Thompson | BUF | 13.34 | 13 | 19.92 | 20 | +0.08 |
| F | Cole Caufield | MTL | 11.11 | 19 | 19.08 | 19 | -0.08 |
| F | Wyatt Johnston | DAL | 11.80 | 6 | 18.86 | 10 | -8.86 |
| **DEFENCE** | | | | | | | |
| D | Cale Makar | COL | 15.35 | 13 | 20.26 | 9 | -11.26 |
| D | Evan Bouchard | EDM | 11.43 | 6 | 16.17 | 8 | -8.17 |
| D | Rasmus Dahlin | BUF | 13.34 | 13 | 16.11 | 18 | +1.89 |
| D | Shayne Gostisbehere | CAR | 13.22 | 19 | 15.14 | 15 | -0.14 |
| D | Darren Raddysh | TBL | 10.08 | 7 | 12.71 | 3 | -9.71 |
| D | Lane Hutson | MTL | 11.11 | 19 | 12.20 | 19 | +6.80 |
| **GOALIES** | | | | | | | |
| G | Scott Wedgewood | COL | 15.35 | 13 | 13.65 | 7 | -6.65 |
| G | Brandon Bussi | CAR | 13.22 | 4* | 12.20 | 5 | -7.20 |
| | | | | **Total** | **330.52** | **209** | **-121.52** |

\* Brandon Bussi only played in 4 playoff games, while his team, the Carolina Hurricanes, played 19. Frederik Andersen turned out be that team's starting goalie for most of the playoffs. 



Summary table:

| Metric | Value |
|:-------|------:|
| Predicted Team Score | 330.52 |
| Actual Team Score | 209 |
| Prediction Error | 121.52 |
| Final Pool Rank | 13 / 38 |


> If I had selected Andersen as a goalie, I would have had a total of 223 points have tied for 3rd in the playoff pool


<figure class="figure">
  <img
    src="/assets/projects/nhl_playoff_pool/expected_vs_actual_playoff_games.svg"
    alt="Expected versus actual playoff games by team"
    style="max-width: 100%; height: auto;">
</figure>



<figure class="figure">
  <img
    src="/assets/projects/nhl_playoff_pool/expected_vs_actual_fantasy_points.svg"
    alt="Expected versus actual fantasy points by player"
    style="max-width: 100%; height: auto;">
</figure>



<figure class="figure">
  <img
    src="/assets/projects/nhl_playoff_pool/fantasy_prediction_error.svg"
    alt="Fantasy point prediction error by player"
    style="max-width: 100%; height: auto;">
</figure>



# Future Work and Limitations
There are two key areas that I think will improve my chances in the playoff pool:
1. Strategy around the optimization problem
2. Improving playoff bracket simulations

My optimized roster this year contained players from 7 distinct teams. This was a good to gain points from top players in early rounds, but as the playoffs progressed I lost large portions of my roster in each consecutive round. In contrast, half of the playoff pool's winning team's roster consisted of Carolina Hurricanes players, who had a very successful run and ultimately won. I think my strategy needs to be more assertive by adding a constraint that an optimized roster can only contain players from, let's say, 2-4 distinct teams. This way I can hopefully continue to gather more fantasy points just by nature of my players being in more games and opportunities to score points. In the scenario of only chosing players from two teams, I could extend constraint by requiring the two teams be from different conferences as the Stanley Cup Finals features a team from each conference.

Of course, this more aggressive strategy only works if my bracket simulations are good and I can predict which teams will have deep playoff runs. In development of the of the Elo rating system and Monte Carlo simulation framework, I had some ideas which would be interesting to try and see if it improves the predictions:

- Apply a grid-serach style of paramater optimization for $K$, $G$, and $M$ in the Elo rating system.
- Calculate Elo rating over a small number of seasons, acknowledging that teams do not start at the same strength at the beginning of each season. Teams are likely to carry over some of their momentum or strength season over season, and the Elo framework could represent that.
- Add per game performance variability as a randomness factor. Any sport is difficult to model and unexpected outcomes regularly occur, so if we added a fitted amount of noise to each game's simulation we could potentially improve the overall predictions.

And lastly, I want to make improvements to the player expected value predictions. A few preliminary tests using Machine Learning showed that a simple Linear Regression model with minimal feature engineering showed slight improvement over the logic of just using a player's regular season points per game as their value in the playoffs. One noteable limiation of the current method is that it assumes point scoring rates of each player is the same in the playoffs as it was in the regular season, which is unlikely to be true. More development time could lead to a more accurate prediction for playoff performance.  

However, I think the benefit gained from this will not be as significant as the playoff prediction portion. Let's say I have the constraint that I only want to pick players from two teams. In this case, a machine learning algorithm will probably just recommend to pick the players on the two teams which had the best regular season performance (about the same as what I implemented this year). The two methods would likely have different expected values per player, but if we constrain the optimization problem so much down to two teams it probably would end up with very similar rosters. At the very least, a more sophisticated expected value methodology is unlikely to be worse than what I did this year, so I will keep it as a secondary objective moving forward.  

In the end though, this is just a fun family competition and I hope they do not mind too much that I am borderline cheating, even if I am still losing.

<p>
  <a href="https://github.com/mattrflew/nhl-playoff-pool-optimizer" target="_blank" rel="noopener">
    <i class="fa-brands fa-github fa-2x"></i>
    &nbsp; View Code on GitHub
  </a>
</p>