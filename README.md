# AI Engineering Path

## Read This First

AI engineering is not 'prompt engineering'. Writing better prompts is a skill, but it is not engineering. If your mental model of AI engineering is about connecting to the OpenAI API and sending prompts, you're in the wrong place.

This path is about building systems that use AI models as components. The model is one part. The data pipeline, the evaluation loop, the calibration layer, the human-in-the-loop design are the other parts. Most of what separates a working AI system from a broken one has nothing to do with the model.

The project is a football match prediction system. Football has a property that most learning projects lack: the ground truth arrives on a schedule, it's public, it's binary, and it doesn't care how confident you were. The market is the best teacher of overconfidence you'll find.

You'll use an LLM. You'll also implement Elo ratings, train a logistic regression model, design a backtesting framework, and make architectural decisions about how these components talk to each other. When you're done, you'll understand why sports bettors consistently lose money even when their models are technically sound. That failure mode shows up in every AI system you'll ever build.

A model that is 70% accurate in backtest and 55% in production is not a betting problem. It is a systems problem. You'll build that system and watch it happen.

Prerequisites are probability and statistics, linear algebra at the level of matrix multiplication, and Python fundamentals. If you finished secondary school with a pass in mathematics and can define a variable, loop, function, and class in Python, you have enough to start. Read the resources section first to fill any gaps, then come back here.

It is okay to quit. Knowing when to cut your losses is something this project will teach you. Apply it to the project too if you need to.

---

## Project: [match-engine](https://github.com/benx421/match-engine)

**What you'll learn:**

- Data pipeline design: fetching, normalizing, and caching structured data from external APIs
- Elo rating systems: the algorithm behind FIFA rankings, chess ratings, and League of Legends matchmaking
- Logistic regression: features, training, and what the output probability actually means
- Walk-forward validation: why train/test splits fail for time-series data
- Model calibration: the difference between a model that outputs 0.72 and one that is actually right 72% of the time
- Prompt engineering for structured output: getting an LLM to return machine-readable JSON
- Hybrid system design: combining a statistical model with an LLM reasoning layer
- Human-in-the-loop pipeline design: what to automate and what not to
- Evaluation against real ground truth: metrics that mean something when results come in on Saturday

**Core task:** Build a pipeline that fetches historical football results, computes Elo ratings, trains a logistic regression model, uses an LLM to score match context from team news, combines the signals into a calibrated probability estimate, and backtests the system against historical bookmaker odds. At the end, answer one question in your `TRADEOFFS.md`: does your system have a genuine edge after bookmaker margin, and how do you know?

---

## What You Are Building

The system has six components. Build them in order. Don't skip ahead.

### 1. Data Pipeline

Fetch historical match results from [football-data.co.uk](https://www.football-data.co.uk/data.php). It's free, no API key, and gives you CSV files per season for major European leagues going back to the early 1990s. You can also use any datasource you deem fit. You want date, home team, away team, home goals, away goals, and the bookmaker odds columns. Store everything locally. A CSV per season is fine. A SQLite database is also fine. The choice is yours and you'll defend it.

Your pipeline must be resumable. If it fails halfway through a fetch, it shouldn't re-fetch what it already has. This is not optional. Production data pipelines fail. You'll learn this by watching yours fail.

### 2. Elo Rating System

Implement Elo from scratch. You're not installing a library. The algorithm is simple: each team has a rating, the expected outcome is derived from the rating gap, and both ratings update after every match based on the actual result. Process every match in chronological order. Never let future data touch past ratings.

When you're done, you'll have a rating for every team at every point in history. Verify it makes sense. The best teams in your dataset should have the highest ratings. If they don't, your implementation is wrong.

### 3. Logistic Regression

You can use scikit-learn for the model. The engineering challenge is the feature set and the validation methodology.

Start with these features: Elo difference between home and away team, a home advantage constant, each team's average goals scored and conceded over their last five matches, and days since each team last played. These are your baseline. You can add more. You'll need to justify every addition.

Don't use a random train/test split. Your data is time-ordered. Splitting randomly leaks future information into your training set and your backtest will look better than it is. Use walk-forward validation: train on seasons 1 through N, test on season N+1. Repeat for every season. If the model doesn't hold up across folds, it doesn't hold up.

Evaluate calibration, not just accuracy. A model that outputs 0.70 for every prediction will be accurate 70% of the time if the home team wins 70% of those games. That model has learned nothing. Plot a reliability diagram: bucket your predictions by confidence and measure actual outcome rates per bucket. They should match.

### 4. LLM Research Layer

Your statistical model knows nothing that happened in the last 72 hours. It doesn't know a key player is suspended, a manager was just sacked, or a team played in UEFA Champions League three days ago. That is what the LLM layer is for.

For each upcoming fixture, fetch recent news for both teams. RSS feeds from major sports outlets, a news search API, web scrapping or the LLM's web search tool can handle this. Pick one and make it reliable.

Write a prompt that takes the news and asks the LLM to score the context impact. The output must be structured JSON. Define the schema before you write the prompt. Something like: home context score (integer, -3 to +3), away score (same), one-sentence reasoning. This is a signal, not a probability override. You're collecting an adjustment, not replacing the model.

Your prompt must produce consistent, parseable output. Test it on edge cases: no news available, news in a foreign language, contradictory signals. Handle failures gracefully. If the LLM layer fails, the rest of the pipeline should not fail with it.

### 5. Prediction Pipeline

Combine the components. Given a list of upcoming fixtures, the pipeline should:

1. Look up current Elo ratings for both teams
2. Compute base probabilities from logistic regression
3. Fetch and score match context from the LLM layer
4. Adjust probabilities by the context score
5. Output a ranked list: fixture, probability, confidence tier (HIGH / MEDIUM / LOW), context summary

Define your confidence tiers before you run the pipeline. HIGH, MEDIUM, and LOW should correspond to specific probability ranges. Don't adjust them after you see the output.

### 6. Backtesting and Evaluation

Don't skip this component.

Using historical odds from football-data.co.uk or any other datasource you choose, simulate placing a fixed stake on every prediction your model would have made in your test set. Calculate ROI, hit rate, and average odds per league. Not every league will be equally predictable. Some will have positive ROI. Most won't.

Then apply the bookmaker margin. Bookmakers don't offer fair odds. They build in roughly 5-10%. Subtract that from your expected value calculation. How many of your leagues survive? How many predictions survive? That number is your real edge. If it's zero or negative, your model has no edge. That's the correct outcome for a first version. Understanding why is the point. Maybe you will understand why "the house always win in the long run".

---

## The Uncomfortable Part

When you run your backtest, it will look good. You'll see 70% hit rates and positive ROI. Then you'll deploy it live and watch the numbers drop. This is not bad luck. It is overfitting, look-ahead bias, or both. The project is designed to make this happen so you can diagnose it.

This is not unique to sports betting. An LLM-assisted fraud detection model scores 94% AUC on your evaluation set and 61% in production. A content ranking model improves click-through in A/B test and degrades 90-day retention. You evaluated on the wrong thing, or your evaluation data didn't represent the real world. Pick one.

A betting model forces this lesson early because the feedback loop is short and the result is not negotiable. You predicted 0.74. The match happened. You were right or wrong. There's no way to redefine success after the fact.

The goal of this learning path is to introduce you to AI Engineering, not sports betting. I do hope you become profitable, but it is unlikely. However, you will pick up very important skills in the process. Even though basic math and python fundamentals are the prerequisites, prior software engineering, or more specifically, backend engineering experience will make your learning journey less frustrating.

---

## Resources

Read the relevant section before each phase, then build. The prerequisites are the exception: fill those gaps before you start anything.

### Prerequisites

Fill these in before you start.

1. **Python:**
   - [Exercism Python track](https://exercism.org/tracks/python): Work through this until variables, functions, classes, loops, and file I/O feel natural.
   - [Automate the Boring Stuff with Python](https://automatetheboringstuff.com): Free. Read chapters 1-9 if you need a gentler on-ramp.

2. **Probability and statistics:**
   - [Khan Academy: Statistics and Probability](https://www.khanacademy.org/math/statistics-probability): Free. Cover distributions, expected value, and conditional probability at minimum.
   - [StatQuest with Josh Starmer](https://www.youtube.com/@statquest): YouTube. Watch the probability, Bayes' theorem, and logistic regression playlists. Clear without being shallow.

3. **Linear algebra:**
   - [3Blue1Brown: Essence of Linear Algebra](https://www.youtube.com/playlist?list=PLZHQObOWTQDPD3MizzM2xVFitgF8hE_ab): YouTube. You need vectors, matrices, and dot products. This covers them visually in under two hours.

### While you build

1. **Data pipelines and API integration:**
   - [football-data.co.uk](https://www.football-data.co.uk/data.php): Read the column key (notes.txt on the same site) before Phase 1. The odds columns are what you need for backtesting.
   - [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/): Read when your pipeline starts failing intermittently.

2. **Elo ratings:**
   - [Arpad Elo, "The Rating of Chess Players" (Chapter 8)](https://www.gwern.net/docs/statistics/1978-elo-theratingofchessplayers.pdf): Read the algorithm section, not the whole book. The maths takes twenty minutes.
   - [FiveThirtyEight's Elo explainer](https://fivethirtyeight.com/features/how-we-calculate-nba-elo-ratings/): Read when you want a practical implementation reference.

3. **Logistic regression and feature engineering:**
   - [An Introduction to Statistical Learning, Chapter 4](https://www.statlearning.com): Read before Phase 3. Covers logistic regression, odds ratios, and the classifier evaluation you need.
   - [Scikit-learn user guide: Calibration](https://scikit-learn.org/stable/modules/calibration.html): Read when you're working on calibration. The reliability diagram section specifically.

4. **Walk-forward backtesting:**
   - [Advances in Financial Machine Learning, Chapter 7, Marcos Lopez de Prado](https://www.wiley.com/en-us/Advances+in+Financial+Machine+Learning-p-9781119482086): Read when your validation results look suspiciously good. The financial context is incidental, but the methodology is universal.

5. **LLM integration and structured output:**

   Pick one provider and stick with it. The structured output pattern is the same across all of them.

   - Claude: [Anthropic API documentation, tool use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
   - OpenAI: [Structured outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
   - Gemini: [Controlled generation](https://ai.google.dev/gemini-api/docs/structured-output)
   - Ollama (local, free): [API documentation](https://github.com/ollama/ollama/blob/main/docs/api.md). Run models locally so you're not burning credits while you iterate.
   - [Building LLM applications for production, Chip Huyen](https://huyenchip.com/2023/04/11/llm-engineering.html): Read when your LLM layer starts producing inconsistent output.

6. **Calibration and probabilistic evaluation:**
   - [Probabilistic Machine Learning: An Introduction, Chapter 3, Kevin Murphy](https://probml.github.io/pml-book/book1.html): Read when your model is predicting probabilities that feel wrong. Covers calibration and Brier score.

7. **System design and human-in-the-loop:**
   - [Designing Machine Learning Systems, Chapter 7, Chip Huyen](https://www.oreilly.com/library/view/designing-machine-learning/9781098107963/): Read after you have a working pipeline and want to understand what production would look like.

---

## Submitting Work

Submit with a `TRADEOFFS.md` that answers these questions:

1. What data source did you use and what are its limitations?
2. What features did you include in your logistic regression model and why? What did you try that didn't work?
3. What does your walk-forward backtest show, per league? Where does the model have edge and where doesn't it?
4. What does your calibration curve look like? Is the model overconfident?
5. How does the LLM layer affect predictions? Can you measure its contribution?
6. After bookmaker margin, what's your model's expected value per bet? Is it positive?
7. What would you build next if this were a production system?

I'll review your code and your `TRADEOFFS.md`. I'm more interested in your reasoning than your implementation. A well-reasoned answer to "this didn't work and here is why" is more valuable than a working system you can't explain.

Create a pull request and request a code review when you're done.

Good luck.
