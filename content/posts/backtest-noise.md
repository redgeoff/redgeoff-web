---
title: "Changing only the random seed made my model look twice as good"
date: 2026-10-12T08:00:00-07:00
draft: true
description: "Eleven seeds, one model, and 14 of 21 research decisions inside the noise. How I measure noise, write the gate before the build, and check a rebuild against reality."
images:
  - /posts/backtest-noise/seeds-hero.jpg
categories:
  - Programming
tags:
  - machine learning
  - backtesting
  - statistics
  - reproducibility
  - ai
  - claude
---

{{< figure src="/posts/backtest-noise/seeds-hero.jpg" alt="Eleven thin grey lines wandering right from a single bright starting point on a dark background, spreading apart as they go, with one glowing red line among them ending highest" align="center" attr="Image credit: author, drawn with code" >}}

In September 2026, I ran the same research model eleven times. Same code, same data,
same settings, same everything except one number: the random seed. The best of those
eleven runs scored about twice as well as the worst.

That would be a curiosity if I hadn't spent months making decisions by
comparing exactly one run against exactly one other run. When we went back through the
research log for that model, 14 of the 21 decisions it recorded, to keep a feature or to
throw one out, rested on a difference smaller than what the seed alone could produce.

If you're new to this, a quick bit of background. I build models that pick stocks, and
I can't test one by waiting ten years to see how it does. So, I backtest it: replay
history, let the model make its picks as if it were living through each date, and score
those picks against what actually happened next. Then I run hyperparameter searches,
which backtest hundreds or thousands of versions of the model with different settings
and compare them. Over the last couple of years, that has added up to a lot of models.

The problem with all of that is that the dangerous bugs don't crash. They produce a
convincing result. In [my last post](https://redgeoff.com/posts/trading-model-leakage/)
I wrote about one way that happens, leakage, where information from the future sneaks
into the replay. This post is about the other two ways: noise, and my own choices.
They're harder to test for, because neither one lives in the code.

## A Number That Wouldn't Reproduce

It started with a rerun. Earlier that year I'd measured what my live trades actually
cost to execute, and the answer was close to nothing. That made me wonder whether a few
research lines I had already closed deserved a second look, since some of them had been
judged with a cost assumption that was now clearly too pessimistic. So, an agent reran
some of the old experiments.

When I reviewed the agent's work, I found that the conclusions were sound but several
of the supporting numbers were wrong. Fixing those meant actually running a check that
had been skipped, and that check reran a configuration from the old log with the
identical command. The result had been recorded as a solid one. The rerun came back
negative.

The first question was whether the engine was random at all. It wasn't. Two runs with
the same seed on the same machine produced equity curves that were identical down to
the byte. So, whatever had changed between the old run and the new one, it wasn't the
dice, and the data changes since the original run became their own investigation.

However, it raised a question that I'd never asked. If the engine is deterministic for a
given seed, how much does the result move when the seed changes?

## How Much Does a Result Move When Nothing Changes?

Some background on why a seed matters at all. The models in that research line were
gradient-boosted trees, and like a lot of tree models, each tree is trained on a random
subset of the features and a random subset of the rows. The seed decides which subsets.
Change it and you get a different set of trees from the same data, which is supposed to
be a detail.

So, we held one configuration fixed and ran it with eleven different seeds over the
real four-year research window. The best seed scored about twice as well as the worst.
From that spread you can work out the smallest difference between two single runs
that you could trust, i.e. the difference you'd only expect to see by chance about one
time in twenty.

Then came the unpleasant part. The research log for that program recorded 21 decisions
that had been made by comparing one run against one other run. Fourteen of them rested
on a smaller difference than that. Some were features I'd thrown out as harmful. Four
were features I'd kept because they helped. In other words, the log had been flipping
coins in both directions, and writing down the result as a finding.

{{< figure src="/posts/backtest-noise/decisions-vs-noise.png" alt="Horizontal bar chart of 21 keep-or-drop decisions from a research log, sorted by the size of the difference each rested on, as a multiple of what changing the random seed alone can produce. A red line marks 1x. Fourteen bars, including four decisions to keep a feature, end left of the line, inside the noise; seven extend past it." caption="Each bar is one of the 21 keep-or-drop decisions in the research log, measured by the difference it rested on, as a multiple of what the random seed alone can produce. Fourteen fall short of the line." >}}

One of those four came back to confirm it. A later rerun, done properly this time with
the seed held the same on both sides, found that a feature the log had recorded as
helping was actually hurting. That's exactly what "inside the noise" predicts: the sign
can go either way.

None of those features were in production, so nothing needed reverting. But the same
habit was also in my more active research lines, which feed decisions that do matter.
When we measured the noise for the daily research engine the same way, one recorded
decision stood out. In a long/short research line, the time of day to trade had been
chosen on a difference between 17 and 29 times smaller than that engine's noise. The
backtest couldn't separate the candidate times at all. The same measurement showed that
the worst loss from peak to trough was even more sensitive than the score: it more than
doubled between the gentlest seed and the harshest.

For what it's worth, my live system trades at 10 AM, and I'd defend that time without
any backtest. The first stretch after the market opens is the most volatile part of the
day, which makes it hard to hit a price target or to model the price you'll actually
get. That's an argument about how the market works, and it doesn't depend on a number
that a different seed could flip.

The rules that came out of this are short:

- No feature is declared harmful, or kept, on a difference smaller than the measured
  noise.
- Comparisons are either seed-matched, with both sides on the same seed, or run over
  several seeds per side and compared on the average.
- The seed, the thread count and the data's provenance are stamped into every recorded
  result, so that a number in the log can always be reproduced.

There was one more surprise in the details. The library we used doesn't pick a random
seed when you don't set one. It uses fixed defaults, so every "unseeded" run in the log
had been perfectly reproducible all along, sitting on one arbitrary draw that nobody had
recorded or varied. Setting the seed to 0 doesn't reproduce those runs either, because
it overrides the defaults with different values. An agent assumed it would, and a test
now pins the fact that it doesn't.

Once the coin flips came to light, I kept pushing to see if there was a way to improve
the statistical significance of the results. That question, how long it takes before
you can tell a real edge from luck, ends up running through the rest of this post.

## Write the Test Before You Build

A couple of weeks later I wanted a fair comparison for my own model: something cheap
and simple that I could hold it up against, over long stretches of history that
included real bear markets. The obvious candidate was a public momentum index fund,
which buys the stocks that have risen the most over the past year. The problem was
that the fund only goes back to 2015, and its index's history before that is the index
provider's own backtest.

So, the plan was to rebuild the index from its published rules, using my own data, and
run it back through history myself.

That plan has a trap in it. A rebuilt index has dozens of choices in it, e.g. how far
back to measure momentum, how to size each position, when to rebalance, and every one
of them is a knob. If I build the benchmark, look at how my model compares, and then
adjust the benchmark, I've tuned my own yardstick. So, the spec was written first and
merged before any construction code existed, along with a gate the rebuild had to
pass: over the years where the real fund exists, its annual return had to land within
three percentage points of the fund's, its risk-adjusted return within a fixed margin,
and its period-by-period returns had to be at least 0.8 correlated. All three, or it
was disqualified. The spec also says why it was written first: every parameter is a
researcher degree of freedom, and my research log already had four results on record
that had been retracted after a closer look.

This wasn't a new habit for me. When working with LLMs, I've found that you have to
have very clear acceptance criteria, especially if you want an agent to run unsupervised
for long periods of time. Without one, the agent stops wherever the result looks good.
I set the initial rules and then refined them with the models.

There was one more guard, and it's my favourite. The benchmark was deliberately not
registered as one of my models. Registered models can be fed to the hyperparameter
search, which means somebody, perhaps an agent, could have swept the benchmark's
own parameters and kept the best-scoring version. Changing it requires editing the code,
which shows up in the history.

The first build failed the gate by a wide margin. The leading suspect was that it
weighted every stock equally, while the real index weights by company size. So, the
second build pulled historical share counts from the SEC's filing database and weighted
by size. It closed about a tenth of the gap, and it failed again.

That second failure was useful in its own right. A pass would have justified buying a
commercial dataset to reach further back in time. A tenth of the gap said that better
share counts weren't the problem, so the data wasn't worth buying, and I found that out
for free.

## Was That Moving the Goalposts?

This is the part I've thought hardest about how to tell honestly.

The spec said that if the gate failed, the work stopped. It failed twice, and I didn't
stop.

What changed was the diagnosis. The benchmark had been built to rebalance on my model's
dates, roughly once a month, so that a comparison between the two would only differ in
how they picked stocks. Meanwhile, the gate asked it to track a fund that rebalances
twice a year. Those two requirements contradict each other, and the gate was close to
impossible to pass by construction. Both failures had to be read in that light.

So, the work split into two builds with different jobs. A faithful replica, on the
fund's own twice-a-year schedule, has one purpose: pass the gate, and prove that the
index can be recreated at all from my data. The monthly build, which is the one my
model gets compared against, is then justified by the replica passing. The thresholds
never moved. Each new attempt was written up as its own issue before it was run, with
the reason it might close the gap. On the fund's own schedule, and weighted by the
shares available to public investors (the free float, which is what the index uses,
rather than every share a company has issued), the replica passed.

{{< figure src="/posts/backtest-noise/gap-to-the-real-fund.png" alt="Bar chart of the gap in annual return between the rebuilt index and the real fund over six attempts: 7.41 and 6.66 percentage points on my model's monthly dates, then 5.42 on the fund's own schedule, all above a dashed limit of 3 set before the first build; then 1.93 with free-float weights, 1.50 with the reference-date lag and daily volatility found from the fund's holdings, and 0.86 with the provider's published weight caps and buffer rule, all below it." caption="The gap between the rebuilt index and the real fund's annual return, attempt by attempt, against the pre-registered limit of three percentage points. The first two attempts were scored on my model's monthly schedule, the rest on the fund's." >}}

That being said, I don't think "the thresholds never moved" settles it. A gate you're
allowed to keep attempting is a search of its own, and with enough attempts, something
eventually passes. What convinced me that this one hadn't just been searched into
passing was the next step.

## Fit to Holdings, Not Returns

A replica that matches the fund's return could still be holding the wrong stocks. Lots
of different portfolios produce similar returns over a few years, especially when
they're all tilted the same way. But the fund has to file its actual holdings with the
SEC, so there's a much harder target available: the list of roughly 100 companies it
held at each rebalance, and how much of each.

So, with the gate already passed, an agent compared the replica against those filings
instead. The first result said that where the replica held the same company, it sized
it almost exactly right, but about a third of the companies were different. Two bugs in
the comparison came out along the way. The comparison had been pitting a freshly
rebalanced replica against the fund's holdings months after its last rebalance, and the
code that matched company names to tickers had been quietly dropping as much as a third
of the fund's holdings, e.g. it turned "Marsh & McLennan Cos." into something that matched
nothing. The finding survived the second fix, which is what made me trust it.

That left the ranking itself as the problem, and parts of how the index ranks stocks
were still guesses. The agent couldn't read the index provider's methodology document,
because the site blocked automated downloads. So, it swept the uncertain parameters and
scored each setting by how many of the fund's actual holdings the replica matched,
never by return.

The first sweep, over how many weeks of volatility to measure, came back flat across a
tenfold range. That turned out to be the clue: the form was wrong, not the length. A
line in the fund's own documentation described volatility as measured on daily
returns, not weekly, and on the holdings, one year of daily returns beat the old
assumption on the filings held out of the sweep. A shorter window won on the filings
it was fitted to and lost on the held-out ones, which is the signature of overfitting,
so the agent took the round one-year value instead.

The second sweep found that the index measures momentum about 42 trading days before
each rebalance, not on the rebalance date itself. That one moved the share of matching
companies from about two thirds to about 80%, and again it did better on the held-out
filings than on the ones it was fitted to.

Neither change was scored on the gate, and the commits report it honestly when the
gate didn't cooperate. The lag narrowed the return gap while two secondary measures got
slightly worse, and switching to daily volatility didn't move the return gap at all. It
was adopted on the documentation and the held-out holdings, not on the result.

Then the methodology document was downloaded by hand. Its appendix confirmed both: momentum
runs from month-end to month-end ending two months before the rebalance, which from a
late-March rebalance is about 42 trading days, and volatility is the "standard deviation
of daily price returns". It also corrected things we'd guessed wrong, including the cap on how much
any one company can weigh and the order of the rules for keeping existing holdings,
and those brought the gap to under one percentage point.

This is the most useful lesson I took from the whole exercise. Fitting the same
parameters to returns would have picked a volatility window that, measured against the
holdings, didn't matter at all, and I'd have declared the spec identified. Matching
about 100 specific companies across 13 real rebalances is not something a single free
parameter does by luck. If you're validating a model or a reconstruction, look for the
target that is hardest to hit by accident, and fit to that.

## A Green Gate With Wrong Numbers

The replica still only started in 2011, because the free source of company sizes it
used begins around then. To reach back further, an agent and I used the same trick as
in my last post. An S&P 500 index fund holds every company in the index in proportion
to its size, and it files those holdings with the SEC, going back to 1995. So, each
holding's value in those filings tells you the company's size on that date.

It passed the gate again on the years where the real fund exists, with about 81% of its
companies in common with the fund and an average weight error of 0.13 percentage points
against the fund's own filings.

Then I reviewed it, and the review still found three wrong numbers:

- The daily return series silently dropped one real day of returns in every rebalance
  period. After the fix it agreed with the period-by-period returns, and a regression
  test now pins that.
- For a company that joined the index between two filings, a fallback took its size
  from the next filing. That's a date in the future, which is the subject of my last
  post in one line. The effect was small, but the default is now strictly
  point-in-time.
- Prices jumped at the point where one data vendor's history hands over to another's,
  so the long runs were redone on a single source and the seam was filed as its own
  bug, which is still open.

The way I do these reviews isn't sophisticated. On delicate or complex work, I ask a
model to go through the result and find what's wrong, often switching to a different
model than the one that did the work, and I repeat it until the review stops finding
anything major. It's a rubber duck that argues back. It's caught more of my mistakes
than any single test, and all three of these came after every check was green.

Two more findings from that review belong to this post's theme, because they're about
noise, not bugs. The first is that leaving out the companies that later failed, the
survivorship bias from my last post, flattered the result by a measurable amount, and
by the most during the dot-com bust, because the companies that were missing were the
ones that collapsed. The second is the one I keep coming back to. Shifting every
rebalance date by a single week, earlier or later, moved the result about as much as
fixing the survivorship bias had. The calendar date you happen to start on is a
setting, and nobody tunes it on purpose, but it's in every backtest.

## And Then It Couldn't Tell Itself From Luck

After all of that, with a gate written in advance, passed on returns, checked against
holdings and reviewed until the review went quiet, the reconstruction gave me 24 years
of a momentum index, most of them from before the fund existed. Over those 24 years, its edge over the
plain S&P 500 isn't statistically established. The most carefully built number in my
research couldn't be told apart from luck.

The reconstruction did its job. When the edge you're measuring is small and markets
are noisy, telling skill from luck takes decades, not quarters, and even 24 years can
fall short. My own model runs into the same wall. On
the evidence so far, it isn't statistically distinguishable from cheaper alternatives,
and what I decided to do about that is the subject of my next post.

## What I'd Do From Day One

If I were starting a research project like this again, here's the list I'd hand to
myself and to every agent working with me:

- **Measure the noise before the first comparison.** Run one configuration across
  several seeds and work out the smallest difference you can trust. Every decision
  smaller than that is a coin flip.
- **Write the gate before the build.** Freeze the spec and the pass/fail criteria
  first, with the reason for each, and make the thing you're testing impossible to tune
  by accident.
- **Never widen a threshold.** If the gate itself turns out to be wrong, say so in
  writing, split the job, and keep the threshold.
- **Fit to the hardest target you can find.** Holdings over returns, names over totals.
- **Review after the gate goes green,** with a different model if you can, until the
  review stops finding things.
- **Shift your dates.** If moving the calendar by a week moves the answer, the answer
  is smaller than it looks.
- **Keep the graveyard.** Every closed experiment stays in the repo with the gate it
  was judged against and its verdict, so the next person, or the next agent, can see what
  was already tried and why it failed.

---

*This describes software I run on my own accounts. It is not investment advice and I
do not offer investment services.*
