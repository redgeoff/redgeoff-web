---
title: "I let AI agents write most of a live-trading codebase. Here's the gate that made it safe."
date: 2026-09-28T09:00:00-07:00
draft: false
description: "About 1,600 agent-written pull requests on a system that trades real money. Tests became the gate, and the interesting part is the three times a green build lied."
images:
  - /posts/ai-agents-test-gate/railroad-crossing.jpg
categories:
  - Programming
tags:
  - ai
  - ai agents
  - testing
  - software development
  - claude
  - cursor
---

{{< figure src="/posts/ai-agents-test-gate/railroad-crossing.jpg" alt="A lowered railroad crossing barrier with glowing red lights across a wet road at dusk, a train blurring past behind it" align="center" attr="Image credit: AI-generated" >}}

For the last 21 months I have been the only engineer on TopSet, a system that trades
real money. It picks a portfolio with machine-learning models, rebalances on a
schedule, and executes through live broker APIs on infrastructure I own end to end.
It is my own capital, not anyone else's.

Agents wrote most of the code. Claude Code, Cursor, Copilot. About 1,600 pull
requests in those 21 months, which is more than I could have typed in three times
the span.

That leaves you with a choice everyone adopting these tools runs into. You can read
every line the agent produces, in which case you have bought yourself a slower
version of writing it yourself. Or you can skim, merge, and move fast, in which case
you accumulate code that looks right and isn't. On a system that places orders with
real money, the second option ends with trades you didn't intend.

I took a third option: make something other than my attention the thing that decides
what ships. Tests became the gate. Nothing merges or deploys without a green build,
and my own reading moved up a level, to design, to test coverage, and to the handful
of results I refuse to let change quietly.

That worked. It also failed in ways I did not expect, and the failures are the
interesting part, so I'll spend most of this post there.

## What I actually read

"Tests are the gate" is not the same as "I stopped looking." What changed is where I
spend attention.

Most pull requests get a light review. Research code I skim. Changes to the parts
that decide what to buy or how to trade it, the model training and the execution
paths, I read properly, looking for whether the approach is reasonable rather than
whether every line is correct. The tests cover the lines. I cover the idea.

Then there is one rule I do not bend. If a change moves a snapshot, or shifts the
result of an end-to-end test, that is a stop. Snapshots and integration results are
supposed to be stable, so a diff in one of them means either the change did something
I didn't intend, or it did something I did intend and I now owe an explanation. An
agent will happily regenerate a snapshot to make a build go green. That is the single
most dangerous thing these tools do, and it is why the snapshot files live in the
repo where a diff is visible in review.

For anything involving a calculation I have not verified before, the tests are not
enough either. I have the agent write a report script, and there are now 161 of them,
that dumps the pipeline to CSV. I open the CSV in a spreadsheet and check the
arithmetic by hand: return per trade, slippage, annualized return. Then I spot-check
inputs, picking a few prices and going back to the raw data directory to confirm
those were the prices the system actually used.

It is slow and it is not clever, and it has caught things nothing else would. A test
asserts that a function returns what the author believed it should return. A
spreadsheet asks whether the number is true.

## What the gate is

{{< figure src="/posts/ai-agents-test-gate/test-gate.png" alt="Diagram of the test gate: every agent pull request runs lint, types, about 7,500 tests, an empty-database check, a migration round trip and terraform validate before it can merge and deploy. A weekly suite runs end-to-end rebalancings, a 100-run flake hunt, training snapshots and leakage checks against main. Production errors page me on Discord, and every fix comes back through the same gate. Underneath: what I still check myself." >}}

Every pull request runs:

- lint (ruff) and type checking (mypy)
- about 7,500 tests across 331 test modules
- an assertion that the test database is empty afterward
- a migration rollback-and-reapply, all the way to base and back
- `terraform validate` against both AWS accounts

If any of that fails, nothing merges. The build is the merge gate, and it is not
advisory.

Once a week, on a schedule, a heavier set runs:

- two end-to-end rebalancing runs against a mock broker
- the same integration test 100 times in a row, looking for flakes
- nine deterministic model-training snapshot runs, pinned inside Docker
- two leakage checks on the training pipeline

Running the same integration test 100 times only makes sense because the test is not
the same each time. The mock broker fills orders probabilistically: a 20% chance per
10-second cycle, 70% of those complete and the rest partial at a random size, at
random prices. Every iteration produces a different interleaving of fills, partial
fills and timing. A hundred runs is a search for race conditions, not a hundred
repeats of one path.

What it finds are real concurrency bugs wearing flaky-test costumes. Worker threads
leaking between tests. A queued task waking up after teardown had already deleted the
row it wanted. A data loader that wasn't thread-safe and corrupted a shared index
under load. Each one showed up first as "that test fails sometimes."

The weekly set exists because a flaky gate is worse than no gate. If a test fails once
every thirty runs, and thirty agent PRs land in a week, something red is always on
screen and you start ignoring red. Hunting flakes is not hygiene here. It is what
keeps the gate meaningful.

## The tests that matter are about money, not functions

Unit tests are cheap and agents write them happily. They are not what I trust.

What I trust are end-to-end tests that drive a full rebalancing against a mock
broker, with the awkward parts turned on. They're named after the ways money goes
wrong:

- orders that have to be resubmitted
- a rebalancing that resumes from the middle, with partial fills already on the books
- a resume that has both buys and sells outstanding
- smart orders cancelled mid-flight, on the sell side and then the buy side
- deposits and repeating withdrawals arriving while a rebalancing is in progress

These are exactly the control flows a language model gets plausibly wrong.
Interrupted work, partial state, money arriving in the middle of an operation: the
code for each of these reads fine in isolation. It's the interaction that bites.

Here is one that made it past review and got caught by the constraint at the bottom
of the stack. When the last pending buy of a rebalancing had its amount adjusted for
pending withdrawals, the adjusted value could land on exactly zero. The existing
guard raised an error on a negative amount, but zero isn't negative. The
affordability check, `available_cash >= amount`, was `0 >= 0`, so it passed. The
order was promoted, and the database rejected it: `amount > 0` is a check constraint
on that table.

The interesting part is what that did to the retry. The integrity error fired inside
the same transaction that carried the completing order's status update, so the
rollback discarded that too. The scheduler came back around, found identical inputs,
and did it again. Not a self-healing retry: a stuck rebalancing.

The fix took three attempts, and the second one is the useful story. Gating the
promotion on `amount > 0` looks correct and does nothing, because the code had
already assigned the adjusted value to a live ORM object. That marked it dirty, and
the next autoflush in the same transaction persisted the zero regardless of whether
the order was ever promoted. The condition guarded the decision, not the mutation.

I only know that because the failure was reproduced live against `main` before any
fix was written, and the first fix was re-run against that same repro rather than
checked by reading the code. When an agent is writing the patch, "I read it and it
looks right" is the single least reliable signal available to you.

## Three ways a green build lied to me

This is the part I'd want to read, so here are the three worst cases, all real, all
from this codebase.

### 1. The test asserted a convention that does not exist

Corporate actions have to be applied to historical prices, and stock dividends are
the fiddly ones. My broker's API returns a `rate` for each. The code treated that
rate as a bare fraction: a 5% stock dividend arrives as `0.05`.

It isn't. It's the ratio of new shares to old. Every one of the 101 stock-dividend
records across 43 symbols in my data has a rate of at least 1.0016. Not one is a
bare fraction. The convention the code assumed does not occur in the data at all.

The unit test passed, because the test asserted the same wrong convention with a
fixture value of `rate: 0.05`, a number that cannot appear in production. The test
and the code were wrong in exactly the same way, which is what you get when both are
written in the same sitting from the same misunderstanding.

The effect was that each stock dividend roughly halved the adjusted price, and it
compounded across events. One symbol with nine of them came out deflated by about
500×, turning a real price history into a fake ramp of more than a thousand-fold
over the period. And because every price stayed between a few cents and a few
hundred dollars, a check for implausible price levels would not have flagged it
either.

{{< figure src="/posts/ai-agents-test-gate/scco-stock-dividends.png" alt="Log-scale chart of Southern Copper (SCCO) adjusted close from 2016 to 2026. The fixed series runs from about $15 to $200. The buggy series starts about 490 times lower, near 3 cents, and steps up at each of nine stock-dividend ex-dates in 2024 to 2026 until it meets the fixed series." caption="Southern Copper's real daily prices, adjusted twice by the same code. The only difference is whether the stock-dividend factor is 1 / rate or 1 / (1 + rate)." >}}

The fix was a corrected fixture, plus a regression test that runs a nine-event
sequence and asserts the cumulative adjustment factor stays sane. The lesson I took:
a fixture is a claim about the world, and nobody reviews fixtures.

### 2. The test passed on NaN

In a research tool, an agent added a second way to select a model configuration and
reported the result in the PR: 100% agreement with the existing selector, at every
sample size tried.

That number was not real. The tool built its pool from a source that was missing the
columns the default scoring version needs, so every score came out NaN. With all
scores NaN, the selector fell through to a fixed tie-break order, which of course
agrees with itself 100% of the time at every sample size.

The test that was supposed to cover this had been passing on those same NaN scores
for as long as it had existed. It never checked anything. Green, meaningless, and
quoted in a PR description as a finding.

The follow-up PR made the condition a hard error rather than a silent NaN, pinned
the test to a scoring version where the columns exist, and added guards. It also
flagged that an older result produced the same way might carry the same artifact.
That is the right instinct: when you find a lie, go and check what else it told you.

### 3. The change did nothing at all

An agent wired a new training target through what looked like the right code path.
The first screen ran clean and produced numbers bit-for-bit identical to the
baseline. Every metric, to four decimal places.

Identical results are not a pass. They're a smell. Two different configurations
producing byte-equal output usually means one of them didn't happen.

It hadn't. The runner always sets a parameter that routes execution to a completely
separate function, which builds its own training target and never looked at the new
field. Unpickling both runs and diffing the actual selections showed 238 of 238
identical picks for the test year. After the real fix, 0 of 238 were identical, and
a regression test now asserts the two modes produce different output.

Nothing failed. The compute was spent. The result would have gone into the research
log as "no effect," which is a conclusion, and it would have been wrong.

### The shape of all three

In every case the build was green, and the number was wrong. Tests prove that the
code does what the tests say. They say nothing about whether what the tests say is
true, whether the code under test ran at all, or whether the data going in is what
you think.

That is the agent-era failure mode, and it is worse with agents than without them,
for a simple reason: an agent writes the code and the test together, from one
reading of the problem. If the reading is wrong, both artifacts are wrong, and they
agree. A human pair would at least have had two readings.

What I do about it now, concretely:

- Distrust results that are identical, unchanged, or perfect. Check the output, not
  the exit code.
- Treat fixtures as claims. If a fixture value can't occur in production, that's a
  bug in the test.
- Make impossible states loud. NaN, missing data, and empty inputs should raise, not
  flow through and produce a number.
- When something is found to be wrong, go back and check what else that thing told
  you.

## The reviewer that isn't a test

Some mistakes are not code-shaped and no test will ever catch them.

So I added a second kind of review: a deliberately skeptical reviewer persona, run
against a finished result rather than a diff, with instructions to argue that the
result is not real. I iterated on the persona with Claude until it stopped being
agreeable.

It found things I am glad it found. That a configuration had been tuned by
repeatedly checking results on data I had promised myself I would only look at once,
so my holdout wasn't a holdout any more. That a cost model was running with zero
financing cost while the strategy it priced used nearly four times leverage.

Neither is a bug. Both are the kind of error that invalidates a conclusion, and no
test suite in the world was going to tell me.

## The gate after the gate

CI stops caring the moment something merges. After that the system runs unattended,
on a schedule, against a live broker, and the question changes from "is this code
correct" to "is this thing behaving right now."

Every ERROR the system logs in production becomes a CloudWatch alarm that posts to a
Discord channel on my phone. That is the whole alerting stack, and for a one-person
operation it is enough: I am not running a rotation, I just need to know.

The loop from there is the same one I use for everything else. Point an agent at the
production logs, hand it a local copy of the database from around the failure, ask
for a diagnosis and a fix with a test. The logs and the data are usually enough for
it to find the problem, and the test it writes is the thing that stops the problem
coming back. Several of the bugs in this post arrived that way rather than through
CI.

Worth saying plainly: the tests did not catch those. The tests caught the next
occurrence.

## What stays human

Architecture. Test design. What "done" means. Merge judgment when the tests pass and
something still feels off.

And one more thing that turned out to matter: the repo's own instruction file. Every
time an agent reached a wrong conclusion in a way another agent would repeat, I
wrote the correction down there. One rule says that any claim of data leakage must
name the specific date boundary it violates, because two different agents had
asserted leakage from reasoning alone, and both were wrong. That file is now a list
of the mistakes this codebase invites. It's the closest thing I have to
institutional memory for a team of one.

## What it cost

The per-PR suite takes about ten minutes. That is the tax on every change, and it is
the number I would defend hardest: ten minutes is short enough that I never route
around it, and long enough to run tests that touch a database and a fake broker.

The machine bill is about 3,000 GitHub Actions minutes a month, of which roughly $20
is paid on top of what's included. The agents themselves are Claude Pro and Cursor
Pro, $20 each. So the tools that wrote most of this codebase cost about the same as
the compute spent checking them, and neither is what the project actually cost.

What it actually cost is fixtures and flakes. Writing a mock broker that fails in
realistic ways, maintaining a self-contained dataset so model training can be tested
at all, and then the long tail of chasing tests that fail once in thirty runs. None
of that is glamorous and all of it is load-bearing. Every hour I skipped there came
back later as an hour of not being able to trust a result.

And sometimes the gate itself is the thing that's broken. The 100-run soak lives in a
shell script rather than inline in CI, because the inline version ran under a POSIX
shell where `{1..100}` isn't brace expansion. It was a literal string. The loop ran
exactly once, and reported success, for longer than I'd like to admit.

The tests are not where I over-built. Every gate in this post exists because
something broke, or nearly did, and the existing checks let it through. I would add
all of them again.

Where I over-built is everything the tests had to cover.

I started TopSet expecting to turn it into a product, so I built it for customers who
never existed. A bot lifecycle with eleven kinds of change request, each one a queued,
locked, asynchronous workflow object, so that deposits and withdrawals and stops could
be requested safely by someone who wasn't me. Two broker integrations instead of one.
A staging environment that mirrors production, behind a VPN. Cross-account backups
with compliance-grade retention locks on a database that holds my own trades.

For one person running their own accounts, several of those could have been a script I
ran by hand while watching it.

That is the part that compounds, and it is why I am putting it in a post about
testing. Every one of those surfaces is something the gate has to cover. The change
requests need end-to-end tests for money arriving mid-rebalance. Two brokers means two
of everything, including the mocks. The staging environment is a second place for
migrations to fail. I did not pay for the product shape once, in the building. I have
paid for it every week since, in the size of the harness that has to keep it honest.

If I were starting again for a book of one, I would build the trading and the research
properly and leave the product machinery until somebody asked for it.

## If you're rolling this out to a team

I've done this alone, with tests I wrote and one person's judgment. That is not the
same problem as a team of fifteen, and I'd be careful about anyone who tells you
otherwise. But some of it transfers, and the ordering I'd insist on comes from a
team I did run: at a healthcare company, requiring automated tests on every PR is
what took active medium and high bugs down 72% and weekly hotfixes from 7 to 1.5.
That was before agents. It's the same first move.

1. **Fix the gate before you raise throughput.** Coverage on the paths that lose
   money or data, and no tolerated flakes. Agents multiply whatever your CI already
   is.
2. **The author owns the change, whoever typed it.** "The agent wrote it" is not a
   review comment, and it is not a defense in a postmortem.
3. **Review the result, not the diff.** For anything that produces a number, ask
   what would look different if the change had done nothing.
4. **Measure escaped defects and hotfix rate.** Not lines, not PR counts. Agents
   make those metrics meaningless overnight.

The part I would not change: the gate is what let one person move at this speed on a
system where mistakes cost real money. The part I underestimated: a gate is a claim
about correctness, and claims need checking too.

---

*This describes software I run on my own accounts. It is not investment advice and I
do not offer investment services.*
