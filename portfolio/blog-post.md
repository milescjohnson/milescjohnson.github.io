# Optimizing Email Open Rate With Thompson Sampling

## Overview

Knowhere News delivers a morning newsletter to almost 1 million subscribers every morning. The subscriber base has shown consistent and steady growth over the last quarter and is projected to continue doing so. Subscriber count, however, is only part of the equation. Consumer email products, in particular daily news and lifecycle newsletters, have an industry average of 28% open rate and an 8% click-through rate. Knowhere exceeds industry average with click through rate, which suggests that readers enjoy and engage with the content, but our open rate has historically been on-par with industry average. This data points to an immediate need to optimize open rate of our daily newsletters. 

We have learned that a major determining factor for open-rate success is the subject line and header we attach to the email. Accross every industy, it's well understood that certain copy variants outperform others, whether it's landing page copy, checkout page call-to-action copy, or user retention copy, the best companies know that finding the words that resonate most with their users is crucial to success. The conventional approach to finding the best performing copy is A/B testing: show different copy to your users and measure the relative performance of each over time to determine which gives the best result.

For example, a retail website trying to promote sales for a certain item may have a banner on their site dedicated to boosting sales for that item. They could then write 5 different versions of copy for that promotion and display each one to 20% of visitors for a month. After a week or a month, they can measure how effective each variant was at getting visitors to click on the banner to determine the highest performing variant to use moving forward.

Optimizing copy for a daily news newsletter presents a few major challenges in this regard.

1. Subject lines are not static over a long period of time; every day has different stories, and the subject line needs to be directly related to that morning's news.
2. Their is an expectation among readers to receive news in the morning. Stale news does not get read.

In other words, every morning we need to send out our newsletter to every reader in a timely manner while also trying to understand and optimize the subject lines and header text to maximize open rate. We don't have the luxury of time to measure a large dataset and converge on a maximally performing variant using the A/B testing approach outlined above. One might imagine testing all the copy variants early with a smaller cohort of subscribers, measuring their performance, and deciding on the best performing variant to use for the rest of the day, but the smaller the data set is, the more noisy and prone to false positives it is.**So how many users do we need to test copy on to feel confident in our decision to send optimally copy to the rest?** Instead of balancing the trade-off of un-optimal early deliveries with more-optimal later deliveries, it would be ideal instead to incrementally attempt the most optimal delivery we can and adjust in real time. That is where Thompson Sampling comes in.

## What is Thompson Sampling

The problem outlined above is what is known as a [multi-armed bandit]() problem in the data science world. A "one-armed bandit" is a colloquial term for a slot machine. From a statistics perspective, a slot machine can be thought of as a variable with an unknown chance of a positive outcome. If you spend $1000 playing that machine, you can come up with a pretty good estimate of the probability of a win. If, however, you have 10 slot machines, each with a different unknown win probability, you will need a strategy to choose which machines to play with your $1000 to maximize your chance of success. Optimizing our newsletter copy is much like the latter. We have a set number of emails we can send out, and our goal is to find the best copy to use for those emails to maximize our chance of a success (email open).

Thompson Sampling is an algorithm that balances known prior probability with exploration to achieve near-optimal distribution. So while early success of a single subject line might influence the algorithm to favor that subject line over others for future iterations, it won't collapse onto a single choice too early. Instead, it takes advantage of an understanding of the number of attempts remaining to explore other options to increase confidence in each choice in an attempt to converge on an optimal distribution eventually.

## Visualizing the Algorithm

To better understand howthe algorithm might approach optimizing subject line open rates, here are some tables representing the first few iterations of the algorithm. In this example, we assume 1000 recipients, 10 subject lines to test, and a batch size of 50 recipients.


## Thompson Sampling Example: Email Subject Line Optimization

This document visualizes how Thompson Sampling might allocate traffic and observe outcomes
when optimizing open rates across 10 email subject lines using batches of 50 recipients.

## Batch 1 — Pure Exploration

We have no prior knowledge of the performance of any of the subject lines, so we distribute them equally.

### Allocation

| Subject | Recipients |
|-------|------------|
| A | 5 |
| B | 5 |
| C | 5 |
| D | 5 |
| E | 5 |
| F | 5 |
| G | 5 |
| H | 5 |
| I | 5 |
| J | 5 |
| **Total** | **50** |


### Outcomes

| Subject | Opens | Attempts | Open Rate |
|-------|------|----------|-----------|
| A | 1 | 5 | 20% |
| B | 0 | 5 | 0% |
| C | 2 | 5 | 40% |
| D | 1 | 5 | 20% |
| E | 0 | 5 | 0% |
| F | 1 | 5 | 20% |
| G | 3 | 5 | 60% |
| H | 1 | 5 | 20% |
| I | 0 | 5 | 0% |
| J | 2 | 5 | 40% |



## Batch 2 — Light Bias Toward Early Winners

We now have data suggesting some variants perform better than others. We slightly adjust the distribution in favor of the winners without overcommiting due to low confidence.

### Allocation

| Subject | Recipients |
|-------|------------|
| A | 4 |
| B | 3 |
| C | 6 |
| D | 4 |
| E | 3 |
| F | 4 |
| G | 9 |
| H | 4 |
| I | 3 |
| J | 10 |
| **Total** | **50** |


### Outcomes

| Subject | Opens | Attempts | Open Rate |
|-------|------|----------|-----------|
| A | 1 | 4 | 25% |
| B | 0 | 3 | 0% |
| C | 2 | 6 | 33% |
| D | 1 | 4 | 25% |
| E | 0 | 3 | 0% |
| F | 1 | 4 | 25% |
| G | 5 | 9 | 56% |
| H | 1 | 4 | 25% |
| I | 0 | 3 | 0% |
| J | 6 | 10 | 60% |



## Batch 3 — Exploitation Begins

As confidence grows, we can weight the distribution further in favor of higher performing variants while still exploring. Weights for future iterations are still influence by performance of both dominant and non-dominant variants.

### Allocation

| Subject | Recipients |
|-------|------------|
| A | 3 |
| B | 2 |
| C | 5 |
| D | 3 |
| E | 2 |
| F | 3 |
| G | 12 |
| H | 3 |
| I | 2 |
| J | 17 |
| **Total** | **50** |


### Outcomes

| Subject | Opens | Attempts | Open Rate |
|-------|------|----------|-----------|
| A | 0 | 3 | 0% |
| B | 0 | 2 | 0% |
| C | 1 | 5 | 20% |
| D | 1 | 3 | 33% |
| E | 0 | 2 | 0% |
| F | 1 | 3 | 33% |
| G | 7 | 12 | 58% |
| H | 0 | 3 | 0% |
| I | 0 | 2 | 0% |
| J | 11 | 17 | 65% |



## Batch 4 — Dominant Variants Emerging

As confidence increases and dominant variants emerge, the distribution begins to converge in favor of them.

### Allocation

| Subject | Recipients |
|-------|------------|
| A | 2 |
| B | 1 |
| C | 3 |
| D | 2 |
| E | 1 |
| F | 2 |
| G | 14 |
| H | 2 |
| I | 1 |
| J | 22 |
| **Total** | **50** |


### Outcomes

| Subject | Opens | Attempts | Open Rate |
|-------|------|----------|-----------|
| A | 0 | 2 | 0% |
| B | 0 | 1 | 0% |
| C | 1 | 3 | 33% |
| D | 0 | 2 | 0% |
| E | 0 | 1 | 0% |
| F | 0 | 2 | 0% |
| G | 8 | 14 | 57% |
| H | 0 | 2 | 0% |
| I | 0 | 1 | 0% |
| J | 15 | 22 | 68% |



## Batch 5 — Near-Convergence Behavior

Eventually, we get a near convergent distribution of dominant variants. Depending on the remaining iterations we can parameterize the algorithm to continue exploration of weak performers or converge entirely on the winners.

### Allocation

| Subject | Recipients |
|-------|------------|
| A | 1 |
| B | 1 |
| C | 2 |
| D | 1 |
| E | 1 |
| F | 1 |
| G | 16 |
| H | 1 |
| I | 1 |
| J | 24 |
| **Total** | **50** |


### Outcomes

| Subject | Opens | Attempts | Open Rate |
|-------|------|----------|-----------|
| A | 0 | 1 | 0% |
| B | 0 | 1 | 0% |
| C | 0 | 2 | 0% |
| D | 0 | 1 | 0% |
| E | 0 | 1 | 0% |
| F | 0 | 1 | 0% |
| G | 9 | 16 | 56% |
| H | 0 | 1 | 0% |
| I | 0 | 1 | 0% |
| J | 17 | 24 | 71% |

---

## Conclusion

By leveraging user timezones, we can begin a batch-delivery process every morning starting with smaller cohorts on the East Coast, and use Thompson Sampling to iterate on open rate data from early readers to converge on optimal subject lines throughout the day. This ensures that all readers receive their newsletter in a timely manner in the morning, while also maximizing the performance of our emails.

