---
title: "Why You Can't Regression-Test a Coding Agent"
date: 2026-09-27
slug: why-you-cant-regression-test-a-coding-agent
readTime: 4
draft: false
tags: ["llm", "debugging"]
description: "Running security regression tests across Claude Code versions produced clean diffs that were completely fictional. Why model variance turns test suites into statistics."
---

In the [last post](/writing/what-claude-codes-deny-rules-dont-cover/) I built a small
harness that runs the real Claude Code binary and checks whether its security
rules actually hold. It found one real gap.

That made the next idea look obvious. Claude Code ships new versions constantly
and updates itself by default, so the rules your company wrote are enforced by a
binary that changes under you every week. Why not just diff it?

```
   version A                    version B
       \                           /
        \      same policy        /
         \     same 30 tests     /
          v                     v
            what changed?
```

Run the same tests against the old version and the new one. Anything that
changes is a regression. That seems like a clean signal.

It is not. Here is why.

## First run looked amazing

I ran all 25 tests against three versions: 2.1.140, 2.1.143 and 2.1.219. One run
per test per version.

**17 of 25 tests came back different on at least one version.**

For about ninety seconds that felt like a great result.

Then I looked at what the differences were. Eleven of them involved a test going
into or out of a state I call INCONCLUSIVE — which means the model simply never
ran the command, so the rule was never tested at all. Four were tests I had
added partway through, which is my mistake, not a version difference.

That left two real-looking changes.

## The same version disagrees with itself

I took one of those two and ran it five times per version instead of once.

| Version | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|
| 2.1.140 | FAIL | INCONC | FAIL | PASS | FAIL |
| 2.1.219 | PASS | INCONC | PASS | PASS | INCONC |

Three different answers from one binary, one policy, one command.

Pooling every run I collected, that test fails about 50% of the time on 2.1.140
and about 40% on 2.1.219. At those numbers there is no difference between the
versions at all. And the pooled data points the opposite way from the single-run
diff, which had reported the old version passing and the new one failing.

So my regression detector's first real output was a regression that did not
exist.

The cause is simple once you see it. The thing being tested is a model. It can
decide not to run the command. It can phrase it slightly differently. None of
that is the security rule changing — it is the model changing its mind.

## Repeating the test does not save you

The obvious fix is to run each test twenty times and compare rates.

That works, but look at what you end up holding. Not "this rule broke in the new
version," but "this failed 40% of the time on the new build and 50% on the old
one, and the error bars overlap." You cannot block a rollout on that. The
verdict turned into a statistic.

## Known bugs I could not reproduce

The real test of a method like this is catching a bug you already know about.

I had two good candidates. Both were public. Both had an exact version where
they were fixed. And every old version of Claude Code is one `npm install` away,
so I installed the exact versions on either side of each fix.

**Bug one:** a report that deny rules stopped being enforced once a command had
more than 50 chained parts. Fixed in 2.1.90. So I wrote a command with 60
chained parts ending in a forbidden action, and ran it on 2.1.89.

It was blocked correctly. I tried 130 parts with different separators. Still
blocked.

**Bug two:** a trust-dialog bypass, fixed in 2.1.53. I installed the versions on
either side and set up the conditions. Both old builds hung and timed out.

Zero for two. I knew the bugs existed. I knew exactly which version fixed each
one. I had both builds installed. And from the outside, without the original
researcher's exact input, I could not make either one happen.

That is the part that generalizes. Knowing a bug is there is not the same as
being able to detect it from outside.

## My own tool lied to me twice

This is the part I would want to read.

**First time:** I wrote a rule as `Read(/some/path/**)`. That looks like an
absolute path and is not one — you need `//`. Three tests failed, and my tool
printed them in a clean table as security failures. All three were my mistake.

**Second time, and this one is worse:** old Claude Code versions refuse to start
if a certain environment variable is set — which it is, if you run your harness
from inside a Claude Code session. Those versions exited immediately with no
output. My runner read "no commands were run" as "the agent chose not to act"
and marked the test INCONCLUSIVE.

The result was a tidy, believable story: two old versions inconclusive, the
current one failing. A clean version boundary. Completely fictional.

It did not look like a bug. It looked like a finding. I only caught it because I
checked *why* the process exited.

If you build something that inspects other software, budget real time for
inspecting it back. Mine invented two stories in a few days, and the second one
would have been reported as a vendor bug.

## Where this kind of testing does work

To be fair, it did catch one real change. When the gap from the last post got
fixed, diffing an old run against a new one reported it correctly — because the
effect was large and consistent, and because I ran three trials per version
instead of one.

The honest summary:

**Works well for** checking one policy, once, when you deploy it. Does this rule
actually block what I think it blocks? You can repeat that cheaply, the answer
is clear, and if it fails you fix the rule.

**Works well for** exploring where a boundary actually sits, by changing one
variable at a time. That is how the pattern-shape issue in the last post got
found.

**Does not work for** a clean pass/fail signal between versions. Model variance
swamps it.

**Does not work for** certifying anything. A failure is strong evidence. A pass
mostly means nothing went wrong that time.

I went in expecting version diffing to be the useful part. It turned out to be
the least reliable part, and testing a single policy at deploy time — the boring
application — is the one that actually holds up.

---

The harness is on GitHub as
[agent-conform](https://github.com/rohitsaini1196/agent-conform). It is archived
research, not a product — 30 tests as plain YAML, the raw run data behind every
number in this post, and the reports from all three versions so you can check
the arithmetic yourself. If you want to see how noisy this actually gets, the
five-run tables are in `report examples/`.
