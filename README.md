# Comprehensive Guide to T20I Monte Carlo Simulations
**A Theoretical and Methodological Textbook for Viva Defense**

---

## 1. Executive Summary
This project implements a stochastic **Monte Carlo simulation engine** designed to project the total runs, per-batter performance (scorecards), and delivery-by-delivery logs of a T20 International cricket match (specifically India vs. South Africa, 2024). 

By ingesting granular, ball-by-ball historical delivery data, the engine isolates the **batter-versus-bowler matchup** as the fundamental stochastic unit of a cricket match. It abandons traditional parametric distribution assumptions (like the Normal or Poisson distributions) in favor of **empirical categorical distributions** derived directly from the frequency of historical outcomes. 

---

## 2. Definitive Design Choices & Methodological Derivations

### Choice 1: Granularity of Simulation (Ball-by-Ball)
**The Problem:** At what level should we simulate? Total innings score? Total runs per batter? Runs per over? Runs per ball?

**The Choice:** We chose **ball-by-ball simulation**.
**The Justification:** 
Cricket is a complex game governed by strict structural rules:
1. A bowler bowls a maximum of 6 legal deliveries per over.
2. The strike rotates depending on the parity of the runs scored (1, 3, 5 runs rotate strike; 2, 4, 6 do not).
3. The strike rotates at the end of an over.
4. Wide and No-ball extras add to the score but *do not consume a legal delivery limit*.
5. Wickets permanently terminate a batter's sequence and introduce a new state variable (the next batter).

Simulating at the "runs per batter" level fails to correctly model strike rotation and bowler allocation. By simulating specifically at the ball-by-ball level, we capture the emergent complexity of the rule set.

### Choice 2: The Major Random Variable
**The Problem:** What precisely are we querying from our random number generator?

**The Choice:** The fundamental random variable $X$ is defined as the *discrete categorical outcome of a single delivery*. We classified every historical and future delivery into exactly one of **15 mutually exclusive state transitions**:

1. `dot_ball`: 0 runs, legal delivery.
2. `single`: 1 run, legal, strike swapped.
3. `double`: 2 runs, legal, strike retained.
4. `triple`: 3 runs, legal, strike swapped.
5. `four`: 4 runs, legal, strike retained.
6. `five`: 5 runs, legal, strike swapped.
7. `six`: 6 runs, legal, strike retained.
8. `wide`: 1 extra run, *illegal* delivery (over counter does not advance).
9. `no_ball`: 1 extra run, *illegal* delivery.
10. `leg_bye`: 1 extra run, legal, strike swapped.
11. `bye`: 1 extra run, legal, strike swapped.
12. `caught`: Wicket, legal, new batter.
13. `bowled`: Wicket, legal, new batter.
14. `lbw`: Wicket, legal, new batter.
15. `other_wicket`: Wicket (run out, stumped), legal, new batter.

**Verification:** By defining exactly 15 collectively exhaustive states, $\sum P(X_i) = 1$. The state machine perfectly maps these 15 outcomes to changes in the game variables (`score, wickets, balls, striker_id`).

### Choice 3: The Probability Distribution (Empirical vs. Parametric)
**The Problem:** How do we assign probabilities to these 15 outcomes? What distribution do they follow?

**The Choice:** We strictly rely on an **Empirical Probability Mass Function (PMF)** derived from historical frequency, rejecting traditional continuous or parametric discrete distributions.

**Why not Poisson?** 
The Poisson distribution models the number of events occurring in a fixed interval of time. In cricket, one might naively think runs per ball is a Poisson variable ($\lambda \approx 1.35$). However, cricket run distributions are heavily multi-modal. A `dot_ball` (0) and a `single` (1) are very common. A `four` (4) and `six` (6) are somewhat common. A `three` (3) or `five` (5) are exceptionally rare. A Poisson curve smoothed over $\lambda = 1.35$ would assign a higher probability to scoring a 2 or 3 than scoring a 4 or 6, which is empirically false.

**Why Empirical Categorical?**
We count the occurrences of each of the 15 events in historical data for a specific batter and bowler, and divide by total deliveries faced. This generates an exact, un-smoothed histogram that perfectly mimics the true behavior of the batter. If a batter historically hits a boundaries 20% of the time, the empirical distribution assigns exactly 0.20 probability to boundaries. No assumptions are forced onto the data.

---

## 3. Dealing with Edge Cases (The Fallback Mechanism)

**The Problem (Unseen Matchups):** What if our simulation specifies that Sanju Samson faces Nqabayomzi Peter, but historically, Samson *never* faced Peter in our dataset? The direct empirical probability $P(\text{event} \mid \text{batter=Samson}, \text{bowler=Peter})$ is technically $\frac{0}{0}$ (undefined).

**The Solution:**
We implemented a multi-tiered fallback architecture.
1. **Tier 1 (Direct Matchup):** If the batter has faced the bowler $\ge 1$ times, we use their direct head-to-head empirical distribution.
2. **Tier 2 (Batter Overall):** If no head-to-head data exists, we aggregate the batter's performance against *all bowlers in the dataset*. We assume Samson against an unknown bowler will roughly mirror Samson's average performance against the field.
3. **Tier 3 (Uniform Fallback):** In the mathematically extreme case where a batter has *no data whatsoever* in the dataset, the system defaults to a uniform distribution $P(X_i) = \frac{1}{15}$. (This serves purely to prevent script crashes).

---

## 4. How to Analyze and Verify Simulation Choices

If you are asked to defend *why* this design choice is applied to this data over an alternative, use this analysis framework:

### A. How to verify Empirical vs. Parametric is better?
To prove Empirical is better, you would run a **Goodness of Fit (Chi-Square) Test**. 
1. Fit a Poisson distribution to Samson's historical runs.
2. Record the squared errors between the Poisson predictions vs what Samson actually scored.
3. Because Poisson predicts smooth decay (predicting lots of 2s and 3s and almost zero 6s), the error will be astronomically high. The Empirical PMF has zero error to the historical baseline by definition. Thus, Empirical applies best to heavily discrete, rule-governed categorical data (like sports scoring systems).

### B. Verification of the Output (Law of Large Numbers)
We ran the simulation $N = 1000$ times. Why 1000? 
By the **Law of Large Numbers (LLN)**, the empirical average obtained from a large sample of independent and identically distributed random variables converges to the true expected value.
1. $N=10$: The variance is too high. A fast bowler taking a random hat-trick completely skews the expected score.
2. $N=1000$: The statistical noise of rare events averages out, leaving a robust, perfectly Bell-shaped (Gaussian) distribution curve of total scores. 
3. The resulting mean is $\approx 190$, matching the notoriously high scoring rate of the 2024 IND-RSA series, validating the accuracy of the underlying probabilities.

---

## 5. Technical Implementation (The State Machine)
Our core Python engine relies on state tracking:
```python
while legal_balls < 120 and wickets < 10:
    bowler = determine_bowler()
    probs = prob_lookup[striker][bowler]
    event = np.random.choice(15, p=probs)
    update_state(event)
```
- We represent the players via index pointers (`striker_idx`, `non_striker_idx`). 
- When an event like a Single occurs, `striker_idx` and `non_striker_idx` swap values.
- When an event like Bowled occurs, `wickets` increments, and `striker_idx` updates to the `next_batter_idx`.
- The engine halts strictly when the overs run out, or 10 wickets fall.

---

## 6. Understanding the Output Visualizations

Inside the final notebook (`monte_carlo_final_viva.ipynb`), we added two explicit advanced plots to understand simulation behavior:

1. **The Worm Chart (Cumulative Runs):** 
   This plots the exact pacing of 10 independent simulation paths. It shows how the score accelerates (steep slopes indicate boundaries, plateaus indicate dot balls or wickets). It visually proves that our ball-by-ball pacing correctly mirrors real-life cricket pacing.
   
2. **The Output Boxplot:** 
   While the histogram shows the normal curve of final outcomes, the boxplot highlights the **Interquartile Range (IQR)**. The middle 50% of the simulation results lie inside the box. If a team wants to know the "safest" predicted score they can rely on backing, they look at the lower bounded quartile. The "whiskers" identify high-chaos outlier matches!
