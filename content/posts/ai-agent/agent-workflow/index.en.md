---
date: '2026-11-08T07:43:24-08:00'
draft: false
title: 'Agent Workflow'
categories:
  - AI Agent
---

Agents have certain advantages over people when it comes to reading documentation, writing code, and similar work. In real engineering, however, they still cause all kinds of problems, such as making projects hard to maintain. The goal of optimizing an agent workflow should be to keep human involvement as small as possible while ensuring that the project reaches the expected quality.

The workflow I currently have in mind goes like this: a human first sets the goal and the specific workflow, and then the agent iterates freely, with the human stepping in from time to time to give high-level guidance and adjustments. The human then organizes and accepts the work at each stage, after which the agent resumes iterating freely while the human keeps giving high-level direction, and the cycle repeats.

**First come agent search and high-level human guidance.** The agent's greatest strength is how quickly it can read materials and look up literature. If a person tries to understand every detail at this point, efficiency is bound to drop. Still, at this stage a person can use high-level knowledge to help the agent converge faster. During this efficient, autonomous iteration, the agent may perform irreversible erroneous operations, and a well-designed workflow should prevent that.

**Next comes periodic organizing and review.** When maintaining a long-term project, agents do not seem to have a good habit of keeping it clean, tidy, and maintainable, and over time their understanding of the context may also drift. This is when a person needs to do the organizing, so that the constraints and requirements do not become so redundant that they make it hard for the agent to judge and analyze. A person also needs to review, so that earlier mistakes do not lead to further work being built on a wrong foundation.

How long review takes is itself one of the factors that determine the efficiency of an agent workflow.

On one hand, code for temporary use that doesn't need to be high quality, where it's enough that it appears to work, needs almost no review.

On the other hand, the review process is also a way for a person to define the problem: knowing what counts as meeting the standard, learning to design evidence, and not blindly trusting the agent. This also tests the depth of a person's knowledge.

Beyond that, it is also necessary to design workflows that reduce the time spent on manual review. For example: what records should the agent leave as it works? Could the agent iterate in an isolated copy, with the human reproducing the experiments in the main version during review? Content judged unimportant could simply be discarded, and important content merged into the main version, so that explorations judged unimportant need no review at all. Automated checks, such as certain criteria, can also be defined by people in advance.

At this stage, one important question is when to interrupt the agent and bring in a human, which can also be decided based on the characteristics of the task.

**Finally, we should accumulate experience across iterations.** For example, we can distill it into skills, hooks, CLAUDE.md files, and so on, and build up operational documentation, all of which make the next iteration of the workflow easier.

*This post was translated from the Chinese original by Claude.*
