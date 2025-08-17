---
title: "The Hidden Forces Shaping Our Systems: Cognitive Biases in Software Development"
seoTitle: "The Hidden Forces Shaping Our Systems"
seoDescription: "Cognitive biases subtly guide our decisions down paths we often don't anticipate, reminding us that we're far less rational than we'd like to believe"
datePublished: Sun Aug 17 2025 11:21:05 GMT+0000 (Coordinated Universal Time)
cuid: cmeflhz1r000202kycwebaysu
slug: the-hidden-forces-shaping-our-systems-cognitive-biases-in-software-development
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/raJrSL-LfVg/upload/992ea66a8db23461a2e342b500234c28.jpeg
tags: software-development, programming, cognitive-bias

---

Cognitive biases subtly guide our decisions down paths we often don't anticipate, reminding us that we're far less rational than we'd like to believe. While we can't completely avoid them, recognizing their influence gives us a better chance to pause, question our instincts, and make choices that serve us — and our teams — more wisely.

We developers love to think we're logical creatures. We debug methodically, we architect systematically, we test rigorously. But here's the uncomfortable truth: our brains are still running the same buggy firmware that helped our ancestors survive on the savanna. And just like any legacy system, it comes with quirks that can significantly impact our code.

Now, there are dozens of cognitive biases out there, and the ones I'm covering here are somewhat arbitrary — probably not even the most common ones in software development. But I've chosen these particular biases because, well, I find them fascinating (if you can say you "like" mental fallacies). More importantly, I think I have a pretty good understanding of how they work and how they show up in our daily coding lives.

## Sunk Cost Bias: When Your Code Holds You Hostage

You know that feeling when you've been wrestling with a problem for hours, your solution is getting messier by the minute, but you can't bring yourself to start over? That's sunk cost bias doing its dirty work.

![sunk cost bias | Behavioral economics | Duke university](https://advanced-hindsight.com/wp-content/uploads/2017/10/SunkCostEffect.jpg align="left")

source: [https://advanced-hindsight.com/blog/b-e-dogs-sunk-cost/](https://advanced-hindsight.com/blog/b-e-dogs-sunk-cost/)

This bias convinces us to keep going down a path because we've already invested time or money, even when starting fresh would be smarter. The thinking goes: "I can't abandon this now — look how much I've already put into it!" [Greg McKeown warns in "Essentialism"](https://www.amazon.com/Essentialism-Disciplined-Pursuit-Greg-McKeown/dp/0804137382) that sunk cost bias is particularly dangerous because it traps us in a vicious cycle where we keep investing more resources to justify what we've already spent, making the problem worse instead of better.

The antidote is learning to step back and reflect. Ask yourself this refreshing question: "If I were starting this project tomorrow, would I follow this same path again?" If the answer is no, you might be throwing good effort after bad.

Here's how it shows up in our daily work:

**Code is not an asset.** You know that function you wrote six months ago that nobody uses anymore? The one that's confusing everyone who reads it? We keep it around because "hey, we might need it someday." Meanwhile, it's making the codebase harder to understand and maintain.

**We nurse dying codebases way past their expiration date.** I've seen teams spend years patching up systems held together with digital duct tape, all because "we've already invested so much in this thing." Sometimes the kindest thing you can do is put the old system out of its misery and start fresh.

The trick is learning when to walk away, even when it hurts. Sometimes the most expensive code is the code you keep.

## Survivorship Bias: Why Success Stories Lie

Here's a story that'll make you think twice about those "how we scaled to millions of users" blog posts. During the Second World War, Abraham Wald was asked to figure out where to put armor on military aircraft. Officials wanted to reinforce the spots where returning planes had bullet holes. Wald said "nope" — armor the spots *without* holes, because planes hit there never made it back to tell their story.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b2/Survivorship-bias.svg/500px-Survivorship-bias.svg.png align="center")

source: [https://en.wikipedia.org/wiki/Survivorship\_bias](https://en.wikipedia.org/wiki/Survivorship_bias)

We only hear from the survivors.

[Nassim Taleb calls this the "Casanova effect" in "The Black Swan"](https://www.amazon.com/Black-Swan-Improbable-Robustness-Fragility/dp/081297381X/). The legendary lover becomes famous for his conquests, but we never hear about all the times he struck out — or about all the other would-be Casanovas who tried the same techniques and failed. This survivorship bias leads us to focus on successful outcomes while completely ignoring the failures that were also part of the process. We get a skewed perception of what works because we only see the individuals or ventures that made it through — not the many who didn't.

What's worse, we underestimate the influence of luck and context. We read success stories and try to copy their advice, forgetting that our situation is different and that context is crucial for success.

In our world, this plays out in some pretty predictable ways:

**We only study the success stories.** When we're researching architecture decisions, we read about companies that scaled successfully with microservices or NoSQL databases. But we don't hear from the dozens of teams that tried the same approach and ended up with an unmaintainable mess. The failures usually don't write blog posts or give conference talks, so we get a completely skewed view of what works (However, you can find some exceptions).

## Confirmation Bias: Our Personal Echo Chambers

Let's be honest — we don't want to prove ourselves wrong. Our brains are like overeager junior developers who only run the happy-path tests.

When we latch onto an idea, we start cherry-picking evidence that supports it while conveniently ignoring anything that doesn't. It's not that we're lying to ourselves (well, not intentionally). We're just really good at finding reasons why we were right all along.

This shows up everywhere:

**We fall in love with our first solution.** Found an approach that seems to work? Great! Now let's spend zero time trying to poke holes in it or considering alternatives. Why waste time on a proper pros-and-cons analysis when we could just start coding? (Spoiler alert: finding problems on paper is way cheaper than finding them in production.)

**We become creatures of habit.** When you've got a hammer that's worked before, everything starts looking suspiciously nail-shaped. Pretty soon, you're using the same patterns for every problem, whether they fit or not.

## Hindsight Bias: Monday Morning Quarterbacking

You know how experts always seem so smart when they explain what happened after the fact? Everything feels obvious in reverse. This is hindsight bias at work — our tendency to perceive past events as more predictable than they were when they were happening.

It's like watching a movie for the second time. All the plot twists that surprised you initially now seem telegraphed from the opening scene. Your brain reconstructs the past with the benefit of knowing how everything turned out, making it feel like the outcome was inevitable all along.

[Philip Tetlock demonstrated in "Superforecasting"](https://www.amazon.com/Superforecasting-Science-Prediction-Philip-Tetlock/dp/0804136718/) just how terrible we are at predicting the future, yet we're all geniuses at explaining the past. Once we know what happened, we unconsciously rewrite our memory of what we thought would happen. The uncertainty, the multiple possibilities, the genuine confusion we felt at the time — all of that gets smoothed over by our brain's desire to create a coherent narrative.

This creates a dangerous illusion: we become overconfident in our ability to predict future events because we misremember how obvious past events were.

This one hits close to home:

**We judge legacy code like armchair architects.** You crack open some old system and think, "What idiot wrote this garbage?" But here's the thing — you don't know what constraints they were working under. Maybe it was the best solution possible given the timeline, the team, and the technology available at the time.

**Every outage becomes obvious in hindsight.** After a deployment goes sideways, managers love to say things like "the warning signs were all there." Were they, though? Because I'm pretty sure if they were obvious, someone would have caught them before they took down production. Post-mortems have this nasty habit of oversimplifying complex failures, which leads to prevention plans that miss the real complexity.

## Egocentric Bias: The Skewed First-Person Perspective

Egocentric bias stems from what psychologists call the "first-person delusion" — our natural tendency to view the world primarily through our own experiences. We have direct, vivid access to our thoughts and efforts, but only indirect access to others'.

This creates the "better than average" effect. When researchers ask married couples what percentage of household chores they handle, the numbers routinely add up to more than 100%. Each spouse vividly remembers their work, while others' efforts fade into the background noise.

[Adam Grant shares workplace examples in "Originals"](https://www.amazon.com/Originals-How-Non-Conformists-Move-World/dp/014312885X/): 94% of college professors believe their work is above average, and 32% of engineers in one company thought they were top 5% performers. This isn't ego — it's the fundamental asymmetry of human perspective.

In our day-to-day work, this shows up as:

**We remember our heroics in HD.** That time you stayed late to fix a critical bug? Crystal clear. The three times your teammate did the same thing? Kinda fuzzy. It's not that we're selfish — our own experiences just have better production values in our mental highlight reel.

**We think we're immune to bias.** This is the sneaky one. We can spot irrational decisions in other people's code all day long, but somehow our own choices always seem perfectly reasonable. It's like being able to see everyone else's blind spots except your own.

The first-person perspective also means we judge our own mistakes more charitably than others'. When we write confusing code, we remember the time pressure and unclear requirements that led to compromises. When we encounter someone else's confusing code, we're more likely to attribute it to poor skills or careless thinking.

## Building Better Code by Embracing Our Flaws

Look, I'm not suggesting we all become paralyzed by self-doubt or start second-guessing every decision we make. These biases aren't bugs to be fixed — they're features of being human, even if they're sometimes annoying features.

But here's what I've learned: the developers I most respect aren't the ones who never make mistakes. They're the ones who've figured out how to catch themselves making predictable mistakes. They build in friction where they know they're likely to go wrong. They ask for fresh eyes when they know they're too close to a problem. They document their reasoning so future teams (including future versions of themselves) can understand why certain decisions made sense at the time.

The goal isn't to become a perfectly logical machine. It's to become a human who knows they're human.

---

If you're curious to dive deeper into this fascinating world of cognitive biases, I can't recommend two books enough: ["Thinking, Fast and Slow" by Daniel Kahneman](https://www.amazon.com/Thinking-Fast-and-Slow-audiobook/dp/B005Z9GAJG/) and ["The Black Swan" by Nassim Nicholas Taleb](https://www.amazon.com/Black-Swan-Second-Improbable-Robustness/dp/B07KRM6L52/). Both are incredible sources for understanding how our minds work — and how they sometimes work against us.