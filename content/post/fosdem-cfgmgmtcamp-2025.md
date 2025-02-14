---
title: "My Notes from FOSDEM & CfgMgmt Camp 2025"
date: 2025-02-13
draft: false
slug: "fosdem-cfgmgmt-camp-2025"
cover:
  image: "/img/posts/fosdem2023/fosdem-blog-logo.png"
categories: ["Tech"]
tags: ["event", "conference", "fosdem", "cfgmgmt"]
---

{{< figure src="/img/posts/fosdem2023/fosdem-blog-logo.png" >}}

_You can also read this article [on Substack](https://newsletter.catops.dev/p/my-notes-from-fosdem-and-cfgmgmt)._

For many years now I spent the beginning of February in Belgium, first at [FOSDEM](https://fosdem.org/) in Brussels and later at [CfgMgmt Camp](https://cfgmgmtcamp.org/) in Ghent. Two years ago I shared [some notes](https://grem1.in/post/fosdem-cfgmgmt-camp-2023/) from the conferences, and I want to do so again.

But first, let me tell you why I love each of these conferences. Frankly, I've heard many concerns about the two: rooms being too crowded, logistics too chaotic, and many topics too outdated. Yet, partially this is why I love them. Both FOSDEM and CfgMgmt Camp are community-driven conferences that truly feel like events organized for enthusiasts by enthusiasts. Just to put it into perspective: both rooms on Nix and retro computing were full.

Of course, there are sponsors, but you don't have to become a master of evasion: people are genuinely interested in getting 3D-printed keychain hangers or talking to the vendors. And if you don't want to - you don't have to. Although there are keynotes, you won't hear from corporate execs about how they made their product 3% faster. I must admit that I did not attend keynotes this year, but two years ago there was a keynote from NASA, and this year there was [a talk about 14 years of systemd](https://fosdem.org/2025/schedule/event/fosdem-2025-6648-14-years-of-systemd/) by Lennart Poettering himself.

Also, unlike in 2023, I highlight only the talks I've enjoyed.

With this being said here are my highlights from FOSDEM and CfgMgmt Camp of 2025.

## FOSDEM

### [The State of Go](https://fosdem.org/2025/schedule/event/fosdem-2025-5353-the-state-of-go/)

- You'll be able to easily get VCS information from within the Go code with the `debug.ReadBuildInfo()` function.
- There are some new functions to work with slices and maps in `1.23`
- You can build your custom `range` functions
- There are more improvements to the language. Nothing revolutionary, but some things can genuinely improve your life.
### [The Inner Workings of Go Generics (FOSDEM)](https://fosdem.org/2025/schedule/event/fosdem-2025-5329-the-inner-workings-of-go-generics/)

A very nice overview of how generics work in general and in Go specifically. The speaker started with how other programming languages (C++ and Java specifically) implement generics, what are the tradeoffs, and what approach Go developers chose in the end.

The video is already available, so you can watch it yourself. The key takeaways are:

- Generics in Go do have some performance impact.
- However, you don't have to use them if you don't need to. In that case, there will be no performance penalty whatsoever.

### [Build better Go release binaries](https://fosdem.org/2025/schedule/event/fosdem-2025-4406-build-better-go-release-binaries/)

A great talk packed with practical advice on how to optimize your binaries for various use cases. Both the video and the slides are available, so you can follow them and add build flags applicable to your case in your build scripts.

### [Distributed SQL Technologies: Raft, LSM Trees, Time, and More](https://fosdem.org/2025/schedule/event/fosdem-2025-4958-distributed-sql-technologies-raft-lsm-trees-time-and-more/)

A talk from a developer of TiDB (a MySQL compatible distributed database) and YugabyteDB (a PostgreSQL compatible distributed database) about the abstractions that make a modern database and the implementation details of those abstractions in TiDB and YugabyteDB respectively.

### [PostgreSQL Performance - 20 years of improvements](https://fosdem.org/2025/schedule/event/fosdem-2025-4615-postgresql-performance-20-years-of-improvements/)

_tl;dr_: PostgreSQL gets faster and better with time. This talk has a lot of benchmark graphs that make it interesting. Unfortunately, there is no explanation for the drop in performance for PostgreSQL 17 on some of those graphs. Some other benchmarks suggest that PostgreSQL 17 is faster than 16, [example 1](https://datasystemreviews.com/postgresql-17-performance-benchmark.html) and [example 2](https://www.crunchydata.com/blog/real-world-performance-gains-with-postgres-17-btree-bulk-scans).  So, it could be an issue with the test setup.

### [Logical Replication Live Session - Keep on Streaming](https://fosdem.org/2025/schedule/event/fosdem-2025-5787-logical-replication-live-session-keep-on-streaming/)

A great presentation from a charismatic speaker. If you actively use logical replication in PostgreSQL for change data capture or zero-downtime upgrades, there wouldn't be much new information for you. However, this talk is a good practical overview of how logical replication works, what's the difference between physical replication, and how one could break it.

### [Anatomy of a Python OpenTelemetry instrumentation](https://fosdem.org/2025/schedule/event/fosdem-2025-4282-anatomy-of-a-python-opentelemetry-instrumentation/)

An overview of the packages that enable OTel auto-instrumentation for Python and a deep dive into how the auto-instrumentation works. Spoiler: monkey pathing under the hood.

### [Anatomy of Table-Level Locks in PostgreSQL](https://fosdem.org/2025/schedule/event/fosdem-2025-4603-anatomy-of-table-level-locks-in-postgresql/)

Unfortunately, there is no video, but this talk was good. There are slides, though, and the presentation has "takeaway" slides. I strongly advise showing those slides to anyone who wants to understand when and what is locked in PostgreSQL and how to minimize the impact. For example, do not mix operations that require stronger and weaker locks in a single transaction.

In this talk, she presented the [pgroll](https://github.com/xataio/pgroll) tool that achieves zero-downtime schema changes in PostgreSQL.

### [How to monitor the monitoring](https://fosdem.org/2025/schedule/event/fosdem-2025-5388-how-to-monitor-the-monitoring/)

A talk from VictoriaMetrics on how they monitor VictoriaMetrics itself. It has some useful ideas about the organization of your dashboards e.g. adding context to the graphs, interlinking graphs, etc.

---

Here are also some talks that I wanted to attend, but didn't for various reasons. I haven't watched the recordings yet, though (for the talks where they are available).

- [Swiss Maps in Go](https://fosdem.org/2025/schedule/event/fosdem-2025-6049-swiss-maps-in-go/)
- [Cache me if you can: P2P Image Sharing in Kubernetes with Spegel](https://fosdem.org/2025/schedule/event/fosdem-2025-4934-cache-me-if-you-can-p2p-image-sharing-in-kubernetes-with-spegel/)
- [A new cgroup cpu.max.concurrency controller interface file](https://fosdem.org/2025/schedule/event/fosdem-2025-6283-a-new-cgroup-cpu-max-concurrency-controller-interface-file/)
- [State of Checkpoint/Restore in Kubernetes](https://fosdem.org/2025/schedule/event/fosdem-2025-4326-state-of-checkpoint-restore-in-kubernetes/)
- [Breaking things for fun and profit](https://fosdem.org/2025/schedule/event/fosdem-2025-4095-breaking-things-for-fun-and-profit/)
- [PostgreSQL Anonymizer and the battle for privacy](https://fosdem.org/2025/schedule/event/fosdem-2025-4258-postgresql-anonymizer-and-the-battle-for-privacy/)

## CfgMgmt Camp

Videos from the main track are available in [this playlist on YouTube](https://www.youtube.com/playlist?list=PLBZBIkixHEiddF07cN9J4Egu9VmBUEwYu). There were cameras on other tracks as well, but I'm not sure if those recordings are available.

 I would especially like to share the slides from a talk that I liked the most: The Worst Issue You Ever Dealt With by Laura Nolan. In this talk, she explores a scientific approach to troubleshooting aka something you spend a lot of time on if you're in an SRE or operational position. It's also curious, that the science existed long before the rise of IT: a lot of research on troubleshooting was done on doctors to help them diagnose their patients better as well as on firefighters to understand how people behave under stress.
- Slides are available [here](https://requisitevariety.net/worstissue.pdf)
- Video is [here](https://youtu.be/Qql42n4NRGs?si=WsevBcOCHsS4i1Jb)

Also, there was a nice, albeit a bit philosophical, talk by John Willis - [From Deming to DevOps](https://youtu.be/t6-Vpgf4jfI?si=-MLQhE11P52cr54m). I haven't attended this talk this time, because I have already heard it at DevOps Days Antwerp :)

---

That's all for this time! As usual, the third day of CfgMgmt Camp is dedicated to workshops. I have attended one about [Nix](https://nixos.org/) because not so long ago I tried to use it to set up my laptop and failed. So, I hoped to get some answers on that workshop. Yet, I must admit, now I'm even more confused than before! Anyways, that was a self-paced workshop and you can find the materials [here](https://pad.okeso.net/s/x3vaYLlEG#), if you want to try it yourself.

Tschüssi!

{{< figure src="/img/posts/fosdem2023/ending.png" align="center" >}}
