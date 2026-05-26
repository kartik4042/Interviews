Oracle Director Round Prep — Sukhada Pendse (IC4 Calibration)
1. Introduction

“Hi Sukhada, I’m Kartik. I’ve been working primarily in backend and systems engineering roles focused around concurrency, performance, and reliability-heavy infrastructure systems.

I started at Dassault Systèmes in the SOLIDWORKS PDM team, where I worked on enterprise data management systems tightly integrated with SQL Server. A large part of my work there involved improving concurrency behavior, reducing database contention, optimizing transaction-heavy workflows, and debugging production deadlocks and latency bottlenecks in large customer deployments.

Currently at IBM Storage Protect, I work on backup and data protection infrastructure systems. My work is split between handling high-severity production escalations and core engineering work around multithreaded backup agents, release modernization, CI/CD improvements, and reliability engineering.

Across both roles, the kind of problems I naturally gravitate toward are usually around difficult system behavior — concurrency bugs, performance bottlenecks, synchronization issues, reliability under load, and debugging distributed failures.

Over time, I realized I want to move deeper toward core database and transaction infrastructure systems, which is why the work your team is doing around transaction processing and distributed systems genuinely interests me.”

2. Why Oracle?

“I’ve always been interested in systems where correctness and performance both matter simultaneously.

In my previous roles, I’ve mostly interacted with databases and infrastructure from the outside — optimizing workflows around them, debugging contention issues, improving transactional behavior, and handling reliability problems.

What excites me about Oracle is the opportunity to work closer to the core infrastructure layer itself — transaction systems, concurrency control, replication, consistency, and low-latency distributed execution.

The problems your team works on are the kind of systems problems I naturally enjoy spending time on.”

3. Why Leave IBM So Soon?

“I actually value stability quite a lot, so this isn’t a reactive switch.

IBM gave me strong exposure to reliability engineering and production-scale infrastructure systems. I learned a lot there, especially around debugging complex failures and operating enterprise-grade systems.

But over time I realized the parts of work I enjoy most are closer to core systems and database infrastructure — concurrency, synchronization, low-level performance, transactional behavior, and debugging complex runtime issues.

Oracle’s transaction engine work aligns much more directly with the direction I want to specialize in long term.”

4. Did You Discuss This With Your Manager?

“Yes. I’ve always believed transitions should be handled professionally.

My manager understood that my long-term interests are moving deeper toward database and systems infrastructure engineering. Even internally, most of the work I naturally gravitate toward tends to be the lower-level debugging and concurrency-heavy areas.”

5. Why Specifically Transaction Systems?

“I think transaction systems are one of the hardest engineering domains because you’re balancing multiple things simultaneously — correctness, concurrency, fault tolerance, latency, recovery, and scalability.

What attracts me is that small architectural decisions can have very deep implications on system behavior under load.

I genuinely enjoy understanding how systems behave under contention and failure scenarios.”

6. Explain Your Dassault Work
High-Level Context

“SOLIDWORKS PDM is an enterprise product data management system tightly integrated with SQL Server. It handles large engineering workflows involving check-ins, check-outs, permission validations, versioning, and hierarchical file relationships.”

Main Problem

“We started seeing severe contention and latency issues under high-concurrency customer workloads.

Operations like file check-in/check-out were slowing down significantly, and in some deployments we were seeing deadlocks and transaction stalls.”

How You Investigated

“I started by analyzing SQL Profiler traces and execution plans to identify expensive queries, lock contention patterns, and transaction bottlenecks.

I also analyzed SQL deadlock graphs and XML reports to understand process lists, resource lists, lock acquisition order, and contention hotspots.”

What You Specifically Did
Locking Improvements

“One major issue was overly broad transactional scope causing unnecessary lock retention.

We reduced transaction scope and separated independent workflows into smaller transactional boundaries to reduce contention duration.”

Query Optimization

“We replaced several recursive nested join patterns with recursive CTE-based approaches where appropriate, which simplified hierarchy traversal and reduced query overhead.”

Batching Optimization

“A major bottleneck came from recursive stored procedure calls repeatedly traversing hierarchy data.

I worked on redesigning that flow by introducing batched processing logic and temporary-table-backed workflows, reducing repeated DB round trips.”

Synchronization Improvements

“On the application side, I also worked on improving synchronization behavior in multithreaded workflows by reducing unnecessary critical-section contention.”

Results

“We reduced high-contention query latency from roughly ~2 seconds to under ~50 milliseconds in critical workflows, while also improving throughput stability under load.”

7. What Was Your Contribution vs Team Effort?

“The overall platform work was obviously collaborative.

But the investigation, profiling, lock analysis, execution-plan debugging, and many of the concurrency optimization changes were directly driven by me as an individual contributor.

I worked closely with DB engineers and senior developers during implementation and rollout.”

8. How Did You Measure Improvements?

“We measured improvements using SQL Profiler traces, execution plans, latency benchmarking under simulated workloads, and production telemetry.

For locking improvements specifically, we compared contention duration, transaction wait times, and deadlock frequency before and after changes.”

9. What Exactly Caused the Deadlocks?

“The biggest contributor was inconsistent lock acquisition order combined with long-running transactional scope.

Some workflows held locks longer than necessary while recursively invoking dependent operations, which increased contention probability under concurrency.”

10. How Do You Approach Debugging Difficult Production Issues?

“I usually approach debugging in layers.

First I try to reduce the problem into observable symptoms — latency, contention, crashes, retries, resource exhaustion, packet delays, lock waits, etc.

Then I narrow the failure surface using logs, profiling tools, traces, execution plans, thread dumps, packet captures, or system metrics depending on the layer involved.

I try not to jump directly into assumptions too early.”

11. Explain Your IBM Work

“At IBM Storage Protect, I work on enterprise backup and data protection systems.

My work is roughly split into two areas:

Around 30% involves handling production escalations and reliability issues coming from enterprise customer deployments.
Around 70% is engineering work involving multithreaded backup agents, release engineering modernization, CI/CD improvements, and infrastructure automation.”
12. Explain the Concurrency Issue at IBM

“We had concurrency-related instability in multithreaded backup agents under heavy workload conditions.

The symptoms involved intermittent hangs, failover instability, and inconsistent restore behavior under parallel operations.

I started by reproducing the issue under load and analyzing thread interaction behavior, synchronization ordering, and timing dependencies.

The root cause involved synchronization contention and race conditions during failover handling paths.

We refined synchronization boundaries and improved ordering guarantees around shared-state transitions, which stabilized restore reliability significantly under load.”

13. What Did Production Escalations Teach You?

“They taught me that correctness and observability matter as much as feature delivery.

In production systems, debugging ability becomes part of the architecture itself.”

14. How Do You Balance Performance vs Correctness?

“I usually prioritize correctness first in infrastructure systems because subtle correctness issues become catastrophic at scale.

But after correctness is guaranteed, I aggressively optimize contention paths, memory access patterns, synchronization overhead, and unnecessary blocking.”

15. Tell Me About Qtick

“Qtick is a personal systems project where I’m exploring low-latency market data infrastructure concepts.

I’m building a Redis-to-kdb+ pipeline with lock-free queues and optimized IPC paths for sub-millisecond tick processing.

The goal is less about trading itself and more about understanding ultra-low-latency systems behavior, backpressure handling, and high-frequency data pipelines.”

16. Where Did You Use Lock-Free Programming?

“In Qtick mainly.

I used lock-free queues in producer-consumer paths to reduce synchronization overhead and improve throughput consistency during high-frequency ingestion.”

17. What Kind of Role Are You Looking For?

“I’m looking for a role where I can work deeply on systems problems — concurrency, transaction processing, distributed execution, reliability, and low-level performance engineering.

I enjoy environments where engineering depth matters.”

18. What Environment Helps You Perform Best?

“I perform best in environments with strong engineering ownership, technically challenging problems, and teams that value deep problem solving over constant firefighting.”

19. How Do You Deal With Ambiguity?

“I try to reduce ambiguity through investigation and decomposition.

Most difficult infrastructure problems initially look ambiguous until enough observability and system understanding is built around them.”

20. Biggest Learning at IBM?

“Production systems behave very differently from controlled environments.

I learned how important observability, failure handling, and operational reliability are in real enterprise infrastructure.”

21. Why Should We Hire You?

“I think my background aligns well with the kind of engineering problems your team works on.

Across both Dassault and IBM, most of my strongest work has naturally centered around concurrency, synchronization, transactional workflows, performance bottlenecks, and reliability engineering.

I’m also someone who genuinely enjoys debugging difficult systems problems deeply rather than staying only at the application layer.”

22. How Do We Know You Won’t Leave Oracle Quickly?

“I’m specifically trying to move toward this category of work long term.

This isn’t a random company switch for me — it’s a specialization decision toward systems and transaction infrastructure engineering.”

23. Have You Managed Anyone?

“I haven’t formally managed a team, but I’m currently mentoring interns and guiding development work around a feature initiative.”

24. Do You Want to Move Into Management?

“Right now my strongest interest is still technical depth and systems engineering.

I enjoy architecture, debugging, and infrastructure problem solving much more at this stage.”

25. But Couldn’t You Continue Growing at IBM?

“I could continue growing there, and I genuinely respect the team and the work.

But I think growth is not only about title progression — it’s also about the type of engineering problems you spend years working on.

Over time I realized I want my core specialization to move closer toward database internals, transaction systems, concurrency control, and distributed infrastructure. Oracle aligns much more directly with that trajectory.”

26. Why Move From Storage to Database Infrastructure?

“To me the transition actually feels connected architecturally.

At Dassault I worked on transactional enterprise workflows and concurrency-heavy DB interactions.

At IBM I moved closer toward reliability, storage systems, failover handling, and infrastructure engineering.

Oracle feels like the natural next step deeper into transaction engines and distributed database infrastructure itself.”

27. Why Did You Move From Mechanical to Software?

“During engineering I became much more interested in systems and software problem solving.

I started exploring programming deeply on my own, especially low-level systems concepts and backend engineering, and eventually transitioned fully into software engineering.”

28. You Mention Low Latency Often — Why Does That Interest You?

“I like systems where small engineering decisions materially affect behavior.

In low-latency systems, you become very conscious about synchronization, cache locality, contention, batching, memory movement, and execution paths.

It forces deeper systems thinking.”

29. What Did You Learn From Working With Production Customers?

“One important thing I learned is that enterprise workloads expose behaviors that local testing often never reveals.

You start seeing rare races, lock contention patterns, timing dependencies, failover edge cases, and operational realities that make systems engineering much more disciplined.”

30. Tell Me About a Time You Made a Mistake

“Earlier in my career, I initially focused too aggressively on optimizing a workflow before fully understanding the operational implications around locking behavior.

The optimization improved throughput locally, but under broader workloads it created unintended contention patterns.

That experience taught me to always evaluate systems changes under realistic concurrency conditions and not optimize in isolation.”

31. Tell Me About a Disagreement With a Team Member

“At Dassault there were discussions around whether to solve a latency issue through hardware scaling versus transactional restructuring.

I pushed toward reducing transactional contention first because the profiling data showed the bottleneck was largely synchronization and lock related.

We eventually validated that through testing and significantly improved latency without major infrastructure scaling.”

32. How Do You Handle Pressure or Deadlines?

“I try to stay systematic under pressure.

In production incidents especially, panic usually worsens debugging quality. I focus first on narrowing the failure surface, stabilizing impact, and then investigating root cause incrementally.”

33. What Kind of Problems Energize You Most?

“Usually difficult infrastructure behavior that requires deep debugging.

Things like intermittent race conditions, contention under load, unexplained latency spikes, failover inconsistencies, or systems behaving differently at scale.”

34. What Was the Hardest Technical Problem You Worked On?

“The deadlock and transactional contention issues at Dassault were technically very challenging because the symptoms were intermittent and workload dependent.

Understanding the interaction between stored procedures, recursive hierarchy traversal, transactional scope, and lock acquisition patterns required deep investigation.”

35. How Comfortable Are You With Distributed Systems?

“I’ve worked more on distributed infrastructure behavior than on building full distributed consensus systems directly.

At IBM especially, I dealt with distributed backup/storage environments, failover coordination, and production reliability issues across nodes.

I’m actively strengthening my distributed systems fundamentals further through deeper database internals study and personal systems projects.”

36. What Are You Currently Learning?

“I’ve been spending significant time studying database internals, transaction processing systems, storage engines, replication models, and distributed execution concepts.

I’ve also been exploring low-latency infrastructure design through Qtick.”

37. Why Do You Think You Fit an IC4 Role?

“I think IC4 requires strong ownership, debugging maturity, systems thinking, and the ability to independently drive difficult technical problems.

Most of my strongest work so far has involved independently investigating complex concurrency or reliability issues, driving root-cause analysis, and implementing scalable fixes across production systems.”

38. How Do You Handle Not Knowing Something?

“I’m comfortable saying I don’t know something directly.

Usually I break the problem into smaller pieces, study system behavior carefully, validate assumptions incrementally, and learn quickly through investigation.”

39. What Do You Value in Senior Engineers Around You?

“I value engineers who think deeply about systems behavior and correctness, not just implementation speed.

I learn the most from people who are rigorous in debugging and architectural reasoning.”

40. What Would You Want to Work On Here?

“I’d love to work on areas around transaction execution, concurrency control, replication, distributed consistency, or low-latency infrastructure paths.

Those are the kinds of systems problems I genuinely enjoy.”

41. If Asked: “Do You Have Any Questions For Me?”
Good Questions
About the Team

“What kinds of engineering challenges is the transaction engine group currently spending the most time solving?”

About Scale

“How do you see the architecture evolving with newer workloads like AI Lakehouse and vector search integrated into transactional systems?”

About Engineering Culture

“What differentiates engineers who succeed long term on your team?”

About Role Expectations

“For someone joining at this level, what kind of ownership do you typically expect in the first 6–12 months?”
