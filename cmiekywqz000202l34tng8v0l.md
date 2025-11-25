---
title: "Oh My Claude! How Software Development Is Changing with Agentic Coding"
seoTitle: "Oh My Claude! How Software Development Is Changing with Agentic Coding"
seoDescription: "After spending time with CLI tools like Copilot CLI and Claude Code, I began to see genuine potential."
datePublished: Tue Nov 25 2025 12:57:19 GMT+0000 (Coordinated Universal Time)
cuid: cmiekywqz000202l34tng8v0l
slug: oh-my-claude-how-software-development-is-changing-with-agentic-coding
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/yLcK3Itx6ok/upload/d67760365a3c8b1305d0bc2766d3096d.jpeg
tags: ai, software-development, programming, software-engineering, ai-agents

---

Recently, the phrase "vibe coding" has gained popularity. It sounds fun — describe what you want, let the AI figure it out, and ship. But here's the uncomfortable truth: vibe coding without understanding what's happening under the hood is an irresponsible way to build software. You might get something working fast, but maintaining it? Good luck.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764074571114/6753343b-bad7-478c-bb69-b351d6296748.png align="center")

source: [https://x.com/karpathy/status/1886192184808149383](https://x.com/karpathy/status/1886192184808149383)

[Simon Willison coined a more thoughtful approach](https://simonwillison.net/2025/Oct/7/vibe-engineering/) in his concept of "vibe engineering" — using AI to build software faster while maintaining a deep understanding of what's being built and how to sustain it long-term. This distinction matters more than ever as agentic coding tools reshape our daily work.

I was skeptical at first. Another shiny tool promising to revolutionize development? I'd seen enough hype cycles. However, after spending time with CLI tools like Copilot CLI and Claude Code, I began to see genuine potential. Not the "AI will replace developers" kind of potential, but something more nuanced: a new way of working that requires us to think differently about our craft.

## The Declarative Shift

What makes agentic coding different from simple code completion? The fundamental shift is declarative problem-solving. You describe what you need: "Here's my codebase, here's a bug, please fix it." The agent enters a loop — researching, implementing, validating against tests — until it arrives at a solution.

This is a departure from the imperative style, where you write every line yourself. It's closer to how senior engineers delegate to juniors: you describe the outcome, provide context, and let them figure out the path. The difference? The agent never gets tired, never forgets the spec, and can iterate at machine speed.

The tools are evolving rapidly. [Claude Code](https://www.claude.com/product/claude-code) was released in February 2025, followed by [Gemini CLI](https://github.com/google-gemini/gemini-cli) and [Codex API](https://github.com/openai/codex) in the spring. The pace of change is staggering. Maybe we should wait and see what sticks? Perhaps. But ignoring this shift entirely feels like ignoring version control in the early 2000s.

## Does It Actually Make Us Better?

Here's the question everyone asks: Are we more effective with these tools? Or just faster? Or neither?

The research is starting to come in. [Stanford's Software Engineering Productivity group](https://softwareengineeringproductivity.stanford.edu/) has been tracking real-world outcomes, and the results are nuanced. Context matters enormously.

* **Codebase size**: The larger and more complex your codebase, the harder it is for LLMs to navigate effectively. They struggle with intricate dependency chains and implicit architectural decisions that experienced developers understand intuitively.
    
* **Language popularity**: More common languages mean more training data. If you're working in Python or JavaScript, expect better support than if you're in Elm or Haskell.
    
* **Complexity and field maturity**: Greenfield projects in well-established domains? Agents excel there. Brownfield work in complex legacy systems? Much harder. The agent doesn't know why that weird workaround exists or what breaks if you touch that ancient module.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764074675708/015a542a-848e-4175-9443-d68d632b014b.png align="center")

source: [https://www.youtube.com/watch?v=tbDDYKRFjhk](https://www.youtube.com/watch?v=tbDDYKRFjhk)

For straightforward tasks—writing boilerplate, generating tests, scaffolding new features—the productivity gains are real. I've seen what would take hours compressed into minutes. But for truly complex problems, agents still need significant human guidance.

## The Real Challenges

I care about code quality. Proper logging, meaningful metrics, comprehensive tests — these aren't optional for me. So how do agentic tools fit into that picture?

Here's what I've discovered: you can drive agents toward quality, but it requires effort. Using [specification files](https://github.com/jorzel/claude-vibes/tree/main/.claude), I've been able to guide Claude to follow code style requirements and testing standards. With some iteration, it produced comprehensive services with proper observability—logging, metrics, unit tests, and integration tests. When something was missing, I could point it out and get corrections.

For what would have been two hours of manual work, the efficiency gains were substantial. But here's the catch: this wasn't magic. I had to know what good code looks like to recognize when the output fell short. I had to understand testing patterns to specify what I wanted. The agent amplified my expertise; it didn't replace it.

So how do you excel even in difficult conditions — large codebases, complex tasks, brownfield projects?

* **Context management is everything**. Proper spec files are crucial. The less irrelevant context the agent has to process, the better its outputs. This isn't simple prompt engineering; it's a new discipline. You need to understand your codebase deeply to provide effective guidance.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764074786635/39c800ce-5c36-4384-b896-bd1b34522884.png align="center")

source: [https://www.youtube.com/watch?v=tbDDYKRFjhk](https://www.youtube.com/watch?v=tbDDYKRFjhk)

* **Delegation to subagents** helps manage complexity. Breaking large tasks into smaller, focused subtasks with clear boundaries produces better results than asking for everything at once.
    
* **The research-plan-implement pattern** emerged from practitioners at projects like [HumanLayer](https://github.com/humanlayer/humanlayer/tree/main). You don't just throw a task at an agent. You guide it through phases: understand the problem space, design an approach, then execute. Each phase benefits from different prompting strategies.
    

## Leverage Cuts Both Ways

Here's something that doesn't get discussed enough: leverage amplifies everything, including mistakes.

If you fail at the specification stage, you'll propagate that failure at machine speed across your entire implementation. Bad architectural decisions encoded in prompts become bad code at scale. Incorrect assumptions about requirements become incorrect features everywhere.

This creates a new kind of risk profile. In traditional development, you catch many mistakes during the slow, manual coding process. With agentic coding, the implementation happens so fast that flawed thinking gets materialized before you have time to reconsider.

The mental alignment problem is real. When you're not writing every line, do you truly understand why the system is changing and how? Code review becomes even more critical, but it's a different kind of review. You're not just checking logic; you're verifying that the agent's interpretation matches your intent.

Some proponents suggest we'll eventually read specs instead of code. I'm not sure we're ready for that. The gap between specification and implementation contains enormous complexity, and that complexity doesn't disappear just because an agent handles it.

## The Junior Developer Question

Are we approaching a time when software engineers aren't needed? Probably not. But here's the more interesting question: what happens to mid and junior-level developers?

The traditional path to becoming a senior engineer involves years of writing code, making mistakes, and developing intuition through practice. If juniors outsource all that practice to AI, how do they develop the judgment needed to guide these tools effectively?

This isn't hypothetical. I've watched engineers rely so heavily on code completion that they struggle to debug when the suggestions are wrong. If agentic coding amplifies this pattern, we might produce a generation of developers who can describe what they want but can't verify whether they got it.

AI assistance requires great skills to validate and direct agents effectively. You need to understand software engineering practices thoroughly to recognize when an agent is heading in the wrong direction. Those practices are learned through doing, not just through watching agents do.

The burden of becoming skilled can't be fully outsourced. If we try, we'll face a pipeline problem: where do future senior developers come from if juniors never develop foundational skills?

## What Stays With Us

The tempo of change is enormous right now. Tools are released monthly. Capabilities expand weekly. Maybe the smart move is to wait for stabilization before investing heavily in new workflows.

But I don't think complete waiting makes sense either. The core pattern — declarative problem description, agentic execution, iterative refinement — appears to be stable even as implementations evolve. Learning to work this way is worthwhile regardless of which specific tool dominates next year.

We're not in a moment where software engineers become obsolete. But engineers who can't harness these tools effectively will increasingly struggle. The skill ceiling is rising: you need everything you needed before, plus the ability to guide and validate AI collaborators.

The question isn't whether to adopt agentic coding. It's how to adopt it responsibly — maintaining the understanding and craftsmanship that make software sustainable while capturing the efficiency gains these tools offer.

That balance is the real challenge. And honestly? We're all still figuring it out.