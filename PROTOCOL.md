# Pre-registration: does the same input give the same risk score twice?

**Status:** committed before any data has been collected. Commit #1 of this repository.
**Author:** Eli Febres
**Date:** 2026-09-05
**Frozen:** nothing above the Amendments section is ever edited. Changes are appended there, dated.

---

## 1. The question

A language model asked to score the same evidence twice, with the same settings, does not always give the same answer. This study measures how often that happens, how far the answers move, and what drives it.
This study grew out of a head-to-head evaluation of scoring models for a country-risk pipeline. The evaluation was originally designed to decide whether a cheaper model could replace the one in production. That evaluation turned up something the evaluation was not looking for: a week with no clear story in the news could be scored anywhere across a range wide enough to change a trading signal, while a week containing an obvious crisis came back the same every time. That observation is the starting point, and the design below fixes how it gets tested before any of the testing happens.

### Terms used here

- **Repeat.** One of several identical calls: same model, same prompt, same input, same settings.
- **Spread.** The gap between the highest and lowest score across a set of repeats, in score points on a 0 to 100 scale.
- **Cell.** One model, one input size, one output mode. Every cell is run 20 times and reported as a unit.
- **Output mode.** How tightly the model is held to the required answer format. *Strict* means the format is enforced while the model writes, so it cannot produce anything that doesn't fit. *Loose* means the model is asked for the format in the prompt and the result is checked afterwards.
- **Main set.** The sixteen endpoints that every hypothesis is tested on.
- **Conditional arm.** The largest self-hosted model. It runs only if a condition written down in advance is met, its results are reported beside the main set, and it never decides a hypothesis.

## 2. Hypotheses

Each hypothesis below comes with a rule that decides if it passes or fails: a comparison in some cases, a count or a threshold in others, all written down before any data exists. A hypothesis is reported as **supported** when its rule is met and **not supported** when it is not. There is no third outcome, and no rule is loosened, swapped or reinterpreted once the results are in.

---

**H1. How the output is constrained matters more than how large the model is.**
The same model, held strictly to the answer format, will vary less on identical input than when it is held loosely.

*Prediction:* two things must both hold. Wherever a model can be run both ways, its spread under the strict mode is lower than under the loose mode on the same input. And across every cell in the study, the strict group has a lower median spread than the loose group. If only one holds, H1 is reported as not supported.

This is deliberately **not** a claim that strict mode brings spread to zero. Models exist that are held strictly to the format and still vary.

> **Where this came from.** The pipeline requires the scorer to answer inside a strict JSON schema, and one candidate model refused to accept the schema's nullable fields. Rewriting those fields into a form the candidate would accept was the same constraint in every way that matters, so it was tried on the production model first, and that model then stopped giving the same answer twice on repeated identical calls. Same model, same prompt, same temperature and seed; the only thing that moved was how tightly the schema limited the output.

---

**H2. Within one vendor, cheaper models vary more.**

Order a vendor's models by price and by spread, and the two orders run opposite to each other.

*Prediction:* those two orders run in reverse closely enough to give a rank correlation of −0.5 or stronger, measured on the middle-sized input. H2 holds only if that comes out in more vendors than it fails in, and an even split counts as a failure. Vendors are never compared against each other, since a price says as much about what a company charges as about the model behind it.

> **Where this came from.** The evaluation's third round put OpenAI's whole range through the same test, measuring each model against itself with ten repeated calls on identical input rather than against a reference. Ranked by how far those ten answers spread, the list came out close to reversed price, and four of the five candidates moved by more than the week-to-week change the series exists to detect. A later re-run on full-size inputs reversed the top two, so the ordering is worth testing rather than assuming.

---

**H3. Longer inputs produce more variation.**

Keep the model, the prompt and the answer format the same, and give the model more evidence to read. The more it has to read, the less it agrees with itself.

*Prediction:* both measures of that disagreement, the spread and the number of different scores returned, go up as the input grows. Neither one falls between 3k and 6k or between 6k and 12k, and at least one is higher at 12k than at 3k, in at least 11 of the 16 endpoints in the main set. The conditional arm, if it runs, is reported beside them and does not count toward that number. This is measured in strict mode, so that a size effect is not confused with a format effect; the loose-mode numbers are reported beside it but do not decide H3. A model that stays level the whole way, or steadies as the input grows, counts against H3.

> **Where this came from.** Every repeat measurement in that evaluation used a short hand-written test input, because that was the only one that existed at the time. Measured again on the inputs the pipeline actually assembles, several times longer and carrying twenty real article summaries and a full article body, the production model's worst-case spread grew by nearly ten times, and it no longer gave the same scored answer even once in ten.

---

**H4. The rating holds still in a quiet year and moves in a chaotic one.**
 
Two full years for one country: a quiet year, and a year containing a currency collapse. Every week of both years is scored once by each model on a shortlist fixed in advance. The quiet year is chosen by the author from that country's news volume and economic data, with no risk scorer involved, and the reasons for the choice are written down and committed before anything is scored.
 
This is not a repeat measurement. H1 to H3 ask the same question many times about one input. H4 asks each question once and follows the answer from one week to the next.
 
*Prediction:* for each model, the variance of its week-to-week score changes is smaller in the quiet year than in the crisis year. Variance here means the usual statistical one, taken over the fifty-one changes between consecutive weeks in each year. A rating that is doing its job should sit still in a year where little happened and move in a year where a great deal did.
 
> **Where this came from.** Across a full quiet year the production model returned only a handful of different scores over fifty-two weeks and leaned heavily on round numbers, against a prompt telling it not to. Across a year containing a currency collapse the same model moved with the events and reached for round numbers far less often. That looked like a rating behaving the way it should, but it was never checked against a year chosen in advance to be quiet, which is what this does.
 
---
 
H1 to H3 measure whether a model repeats itself. H4 asks a different question of the same rating: whether it moves for the right reasons over a year. The two belong together because a rating that cannot repeat itself on Tuesday cannot be trusted to have moved for a reason by Friday.
 
## 3. What gets measured
 
Recorded for **every call**, before anything is parsed or interpreted:
 
| Recorded | Why it is needed |
|---|---|
| the raw response, exactly as returned | the only thing that cannot be reconstructed later |
| the scores read out of that response | what the results tables are built from |
| a hash of the prompt, the format and the input | proves every repeat really did receive identical input |
| model name, vendor, output mode | identifies which cell the call belongs to |
| temperature, and the seed or a note that the vendor offers none | vendors differ here, and the results should say so rather than imply otherwise |
| tokens in, tokens out, and the actual cost | H2 needs the billed cost, not the advertised rate |
| how long the call took | reported for interest; no hypothesis rests on it |
| whether the answer fit the required format on the first try | a failure is a result, not something to retry away |
| the date and time, in UTC | a vendor can change how a model is served without changing its name, so every result is tied to a date |
| for self-hosted calls: the weights hash, the serving-software version, the container image digest, the GPU model and count, the driver version, and whether batch-invariant mode was on | a self-hosted result is only repeatable if every part of the stack that produced it is written down |
 
Reported for **every cell**:
 
- **Different scores returned.** How many distinct values came back across the 20 repeats.
- **Spread.** Highest minus lowest, in score points.
- **Field flip rate.** For each individual field in the answer, how often it took more than one value across the repeats. This is not the same as spread and is not covered by it: when that measurement was re-run, the overall score held perfectly steady across ten calls while a sovereign-stress flag flipped between true and false underneath it. Spread alone would have called that cell steady.
- **Exact-repeat rate.** The share of repeats whose whole answer matches the most common one, field for field.
- **Round-number share.** The share of scores landing on a multiple of five. In the earlier comparison one model did this 69% of the time and another 19% on the same inputs, against a prompt that tells them not to. Tracked as a sign of whether the model is following instructions or reaching for a default; no hypothesis rests on it.
- **First-try format failures.** How often the answer did not fit the required format before any retry. Retries never count as successes.
- **Week-to-week variance**, for the two-year run only. For each model and each year, the variance of the changes in its score from one week to the next.
- **Cost per call**, and **cost per usable call**. The second divides cost by the share of repeats that agreed with the most common answer, so a cheap model that has to be run several times is priced accordingly. For self-hosted models the cost is the GPU time the cell consumed, billed at the hourly rate actually paid.

## 4. Design
 
### 4.1 One cell
 
A cell is one model, one input size and one output mode, run **20 times** on input that is identical down to the byte.
 
- **Input sizes:** roughly 3k, 6k and 12k tokens. Three real evidence bundles taken from the risk pipeline's development database, saved into the repository before collection starts and never regenerated afterwards.
- **Prompt and format:** one scoring prompt and one answer format, saved into the repository before collection starts. Every vendor receives the same bytes.
- **Settings:** temperature 0 throughout. A fixed seed wherever the vendor accepts one, recorded as unavailable where they don't. No other sampling settings are touched; the vendor's defaults are used and written down.
- **Order:** the 20 repeats in a cell run back to back, and the cells run in an order written down before collection starts, so no ordering decision is ever made after seeing a result.

### 4.2 Models
 
Four vendors and two self-hosted models, sixteen endpoints in the main set, plus one conditional arm. Every vendor contributes at least three, spanning that vendor's range from cheapest to most expensive, because H2 needs three points inside a vendor before a ranking means anything.
 
| Vendor | How many | Output modes available |
|---|---|---|
| OpenAI | 4, cheapest to most expensive | strict; loose |
| Anthropic | 4, cheapest to most expensive | schema through the tool interface; plain prompting |
| Google | 3, cheapest to most expensive | strict; loose |
| Qwen (hosted) | 3, cheapest to most expensive | whichever the chosen route offers |
| Self-hosted, main set | 2, from one model family: Qwen3-4B and a model of roughly 23B parameters | strict, enforced locally; loose |
| Self-hosted, conditional arm | 1, from the same family, 100B parameters or more | strict, enforced locally; loose |
 
**The self-hosted models.** All three come from the same model family and are served by the same version of vLLM, pinned by container image digest, with batch-invariant mode switched on. The two in the main set each run on one GPU with no quantization, in bf16: the 4B on one RTX 4090, the 23B on one 80 GB card. Because everything except the parameter count is held the same between them, the difference between those two is a test of model size and nothing else.
 
The conditional arm is different in kind. A model of 100B parameters or more does not fit on one card in bf16, so it must be either quantized or split across several GPUs, and each of those changes the arithmetic that produces the answer. Its result is therefore not a size comparison against the other two; it is a test of whether a scorer at that scale, served the way it would actually have to be served, repeats itself. It runs only after every main-set cell is complete and its records are committed, and only if the 4B and the 23B disagree with each other on spread, in strict mode, on at least one input size by more than the smaller of the two spreads. If they do not, the arm is not run, and that is recorded as a result rather than a gap. The amendment that opens the arm states the exact model, the quantization if any, and the number of GPUs and how the model is split across them, before its first call.
 
Exact model names and the access route for each are written down and committed on the day the first cell runs. Any vendor that cannot be reached by the time collection starts is listed under Exclusions, with the reason, before its first call would have been made.
 
*Still to fill in before the first run: the exact model names, the route used for hosted Qwen, and the exact 23B checkpoint. Closed by an amendment, not by editing this line.*
 
### 4.3 The two-year run behind H4
 
One country, two calendar years: one containing a currency collapse, one chosen as quiet. One hundred and four weekly evidence bundles, built by the same pipeline that produced the three saved above. Each week is scored **once** by a shortlist of models fixed in advance: the cheapest and most expensive endpoint from each vendor, plus the two self-hosted models in the main set. The conditional arm is not part of the two-year run. The size of each week's input is recorded with its results.
 
The quiet year is picked by the author from that country's news volume and economic data alone, with no model in the loop. The reasons for the pick, the data they rest on and the time the choice was made are written down and committed before any scoring begins.
 
*Still to fill in before the run: the country and both years. Closed by an amendment.*
 
### 4.4 Spending limit
 
The design comes to roughly 1,920 calls for the main set of cells, being 16 endpoints across three input sizes and two output modes at 20 repeats each, and fewer where a vendor offers only one mode. The conditional arm, if it runs, adds 120. The two-year run adds roughly 1,040, being ten models across one hundred and four weeks, scored once each. A cost estimate by vendor is written down and committed before anything runs, and the self-hosted estimate is stated in GPU-hours at the rate actually paid, with the conditional arm priced separately. If the real bill runs more than 50% over that estimate partway through, collection stops and the partial result is published as partial, with the stopping point named. It is not quietly trimmed to fit.
 
## 5. How the results will be analysed
 
- The code that turns raw records into the results tables and the four hypothesis tests is written and committed **before** a single result exists, and runs with no manual step in between.
- Every number in the published tables can be reproduced by anyone who runs that code against the raw records.
- The main table is ordered by spread on the middle-sized input in strict mode, lowest first, with ties broken by cost per usable call.
- The hypotheses are tested exactly as written above. No alternative threshold is introduced after the fact.
- The comparison between the two main-set self-hosted models, and the conditional arm's results if it runs, are reported in their own table. No hypothesis in this study rests on them; they are reported as exploratory.
- Anything else worth reporting that is not one of H1 to H4 is reported in a section marked as exploratory and is not presented as a result of this study.

## 6. Commitments
 
- No model is dropped once its results are known. A model that fails every call is reported as failing every call.
- No repeat is re-run for looking wrong. A call is repeated only when it never completed, meaning a timeout or a server error, and every such repeat is recorded as one, with the original failure kept.
- No cell is added once collection has started, unless it was declared in an amendment dated before that cell's first call. The conditional arm is declared here, and the amendment that opens it must be dated before its first call and after the last main-set record is committed.
- No change to the prompt or the answer format after the first call. If one becomes necessary, the study restarts and the earlier results are kept in the repository, set aside and marked as superseded, with a note explaining why.
- No number in the write-up comes from anywhere except the raw records the code produced.

## 7. Exclusions
 
*(none yet. Filled in by amendment before collection starts.)*
 
## 8. Limits of this study, stated in advance
 
- One prompt, one answer format, one kind of task. This measures how steady these models are **on this job**, not how steady language models are in general.
- Twenty repeats sets a floor on what can be detected. A model that wavers once in two hundred calls will look perfectly steady here, and the reports say so.
- Vendors can change how a model is served without changing its name. Every result is true of that model **as served on that date**, which is why the kit is built to be re-run rather than cited forever.
- H4 scores each week once, so it cannot tell a rating that sat still because nothing happened from one that sat still because the model reached for the same default every week. H1 to H3 measure that second behaviour directly, and the round-number share is reported for both years so a reader can see whether the quiet year was flat for the wrong reason.
- The quiet year is chosen by judgement, from news volume and economic data. The reasons are written down in advance so they can be argued with, but a different author could pick a different year.
- The self-hosted models are there to test the output-mode question in H1 and to give one clean size comparison, not to argue that self-hosting matches a frontier model on quality. Two sizes from one family on one serving stack is a narrow window on size; the result says what happened in that family on that stack, not what size does in general.
- Every self-hosted cell runs with nothing else on the GPU. That is the quietest condition a server can be in, and it is not the condition a shared endpoint runs under, so the self-hosted numbers are a floor for that stack rather than a prediction of how it would behave under load.

## 9. Amendments
 
*(append only, each entry dated; nothing above this line is edited)*