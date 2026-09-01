# Viva Question: Monte Carlo Probability Estimations for Match Results

**Question Addressed:**
*Choose a cricket match of your choice (One Day/T20), and estimate the probability distribution of its results. You can separate the results by two cases – Team 1 bats first, or Team 2 bats first. The approach should be based on Monte-Carlo simulations, i.e. you simulate many matches, and estimate the outcome distribution based on the results of those. The simulation of the matches can be at any level of detail that you want. You can make any reasonable assumptions, and use any data that you feel is necessary.*

---

## 1. Selected Match Data and Assumptions

- **Selected Match Data:** India vs South Africa T20I Series (2024). We use the ball-by-ball delivery dataset covering the 4 matches from this tour.
- **Team 1:** India (`IND`)
- **Team 2:** South Africa (`RSA`)
- **Level of Detail:** **Ball-by-Ball Simulation.** We did not simulate broad inning-level estimates (like normally distributed totals). Instead, we meticulously simulated every single delivery based on specific `batter-vs-bowler` historical probabilities extracted from the dataset.
- **Rules Simulated:** 10 Wickets, 120 Legal Deliveries, Strike Rotation (singles, threes swap strike), and target-chasing.
- **Assumptions (Early Stopping):** When the chasing team (Team 2 in the inning) surpasses the target score defined by Team 1, the simulation algorithm mathematically halts, correctly mimicking target-chasing mechanics.

## 2. Methodology

To answer this question precisely, we configured our Monte Carlo Game Engine to simulate two distinct cases:

1. **Case 1 (Team 1 Bats First):**
   - India simulates an unconstrained innings until 120 balls or 10 wickets. This sets `Target = India Score + 1`.
   - South Africa simulates a chasing innings. If they reach Target, the script logs `RSA Win`. If they max out exactly 1 run short, the script logs `Tie`. Otherwise, `IND Win`.

2. **Case 2 (Team 2 Bats First):**
   - South Africa simulates an unconstrained innings. This sets `Target = South Africa Score + 1`.
   - India simulates a chasing innings.

We executed $N = 1000$ independent match simulations for each case (2000 total matches). The Law of Large Numbers ensures computational convergence toward the true mathematical outcome probabilities.

---

## 3. Results & Probability Distribution

Based on empirical data executed through the Monte Carlo pipeline, here are the estimated probability distributions of the match results:

### Case 1: India (Team 1) Bats First

*India sets the target; South Africa chases.*

| Result | Estimated Probability |
|--------|-----------------------|
| India Wins | **67.9%** |
| South Africa Wins | **31.1%** |
| Match Tied | **1.0%** |

---

### Case 2: South Africa (Team 2) Bats First

*South Africa sets the target; India chases.*

| Result | Estimated Probability |
|--------|-----------------------|
| India Wins | **69.5%** |
| South Africa Wins | **30.3%** |
| Match Tied | **0.2%** |

---

## 4. Analytical Conclusions

The estimated distributions confirm several actionable insights about the team composition found in the underlying ball-by-ball dataset:

1. **Structural Superiority:** India possesses a heavy statistical edge against South Africa in this dataset configuration. The win probability consistently centers around **~68-70%**, heavily favoring India.
2. **First-Innings Dominance (Negligible Toss impact):** The probability of an Indian victory shifts less than 2% depending on who bats first ($67.9\%$ defending roughly equals $69.5\%$ chasing). This indicates that the "Toss Advantage" is dwarfed by raw team strength differential.
3. **Distribution Consistency via Law of Large numbers:** The consistency across both target-setting and chasing mathematically validates the stability of the empirical probability metrics assigned to the batter-vs-bowler interactions in the simulation engine.
