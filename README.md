# llm-determinism-kit

**Does the same input give the same risk score twice?**

A small, public study of how steadily large language models score the same evidence when asked more than once, and whether a rating built on them moves for the right reasons over a year.

## Status

Pre-registered, no data collected. The only substantive file in this repository is the protocol, which was committed before any code was written or any model was called. That ordering is the point: the hypotheses, the thresholds that decide them and the rules the study will follow are on record before a single result exists, so they cannot be adjusted to fit what comes back.

There is nothing to run yet.

## Why this exists

The study grew out of an internal evaluation of scoring models for a country-risk pipeline, run to decide whether a cheaper model could replace the one in production. That evaluation found something it was not looking for. Asked to score the same week's news several times under identical settings, a model would sometimes return a range of answers wide enough to change a trading signal, and it did so most on weeks where the news pointed nowhere in particular. A week containing an obvious crisis came back the same every time.

That is worth measuring properly, on a fixed design, across several vendors, rather than being left as an anecdote from one pipeline.

## What the protocol covers

Four hypotheses, each with a rule that decides it written down in advance:

1. Holding a model strictly to its answer format makes it steadier than holding it loosely.
2. Within one vendor, cheaper models vary more on identical input.
3. Longer inputs produce more variation, with everything else held fixed.
4. A rating built on these models holds still across a quiet year and moves across a year containing a currency collapse.

The protocol also fixes what is recorded for every call, how the models are chosen, the spending limit, and what the author commits not to do once results start arriving. Read it in full before reading anything else here.

## What is in the repository

- **PROTOCOL.md**: the pre-registration. Frozen; changes are appended as dated amendments and the original text is never edited.
- **LICENSE**: terms of use.
- **README.md**: this file.

## What comes next

In order, and each committed before the next begins: the frozen inputs and answer format the study will use, the code that runs one cell and records it, the code that turns raw records into the results tables and the hypothesis tests, a cost estimate by vendor, and only then the first call to a model.

Results will be published in this repository as raw records alongside the tables built from them, so every number can be reproduced by anyone with the same keys.

## Author

Eli Febres. Questions and disagreements with the protocol are welcome as issues; the protocol itself will not be edited in response, but an amendment can be.