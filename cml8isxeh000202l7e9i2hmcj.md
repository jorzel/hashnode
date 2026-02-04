---
title: "Single Skill: What Makes Engineers Great"
seoTitle: "
Single Skill: What Makes Engineers Great
"
seoDescription: "When I think about one skill that separates good developers from exceptional ones, it's the ability to discern symptoms from root causes."
datePublished: Wed Feb 04 2026 21:09:11 GMT+0000 (Coordinated Universal Time)
cuid: cml8isxeh000202l7e9i2hmcj
slug: single-skill-what-makes-engineers-great
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/OjrmUvnkMYs/upload/53a85b9f7e471ca03089cf653f2334cc.jpeg
tags: software-development, distributed-system, software-engineering, problem-solving-skills, root-cause-analysis

---

In the AI era, where agents generate decent code fast, the idea that great developers just produce *more* is even harder to defend. Output isn't the differentiator.

When I think about one skill that separates good developers from exceptional ones, it's the ability to discern symptoms from root causes.

Consider a production incident. CPU is overutilized. The instinctive response: we need more resources. Sometimes that's true. But it's a symptom, not a cause.

Let's trace it. You check metrics: CPU spiked at 14:32. What else happened at 14:32? Traffic looks normal. No deployments. But memory usage also climbed—and stayed high. You dig into application metrics. One service is making database calls in a loop. Why? A cache is empty. Why is the cache empty? The cache service restarted at 14:30 after a routine update. It came back healthy, but cold. The first few requests triggered expensive queries that normally never run. Those queries weren't optimized—nobody noticed because the cache always absorbed them. One query had a missing index, causing full table scans.

```go
Why is the CPU overutilized?
  └── One service is making excessive database calls

      Why is it making excessive database calls?
        └── The cache is empty, so every request hits the database

            Why is the cache empty?
              └── The cache service restarted at 14:30

                  Why do empty cache requests cause high CPU?
                    └── The queries aren't optimized (missing index, full table scans)

                        ROOT CAUSE: Unoptimized query hidden behind cache for months
```

So, the symptom was CPU overutilization. The trigger was a cache restart. The root cause was an unoptimized query hiding behind a cache for months.

If you stopped at "we need more resources," you'd scale up, the cache would warm, the CPU would drop, and you'd move on. The query would still be there, waiting for the next cold start. You need observability—but observability you actually understand. Observability that leads you to the source, not just confirms the symptom.

In distributed systems, this gets tricky. Failures cascade across services. What you observe is three hops away from where it started. AI can help, but it needs the right framing. Feed it symptoms, and it chases symptoms. Overload the context window too fast, or point it in the wrong direction early, and you've wasted your best debugging partner.

So what is this skill, exactly? For me, it's a blend of deep system understanding—including the business domain—and intuition. Not gut feeling. Intuition as compressed experience. You've seen enough incidents and systems that you recognize patterns before you can articulate why. [Malcolm Gladwell explores this in *Blink*](https://www.amazon.com/Blink-Power-Thinking-Without/dp/0316010669): experts often know something is wrong before they know what.

But intuition has limits. In complex or chaotic systems, the patterns you rely on can mislead you. Confidence becomes a trap. You need awareness of where your mental models break down.

This is what matters in the AI era. AI can be a powerful ally in debugging—it can analyze logs, suggest hypotheses, and trace dependencies. But it follows your lead. If you point it at symptoms, it chases symptoms. If you frame the problem well, it accelerates your investigation tenfold.

The bottleneck isn't code generation anymore. It's navigation. Knowing what questions to ask. Knowing which logs to pull, which metrics matter, and which rabbit holes to avoid. You need to understand systems back to front—not because AI can't help, but because AI needs a skilled navigator to be useful.

Deep thinkers—people who don't just do things but understand why things work—are more valuable now than ever.