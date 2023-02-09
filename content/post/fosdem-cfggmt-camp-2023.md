---
title: "My Notes from Fosdem & Config Management Camp 2023"
date: 2023-02-10
draft: false
slug: "fosdem-cfgmgmt-camp-2023"
cover:
  image: "/img/posts/fosdem2023/fosdem-blog-cover.png"
categories: ["Tech"]
tags: ["event", "conference", "fosdem", "cfgmgmt"]
---

# My Notes from Fosdem & Config Management Camp

Sup! It’s been a while.

So, both [Fosdem](https://fosdem.org/2023/) and [Config Management Camp](https://cfgmgmtcamp.eu/ghent2023/) conferences are back offline, therefore I've spent a long weekend at my usual Belgian conferences.

Here I want to share some notes with you from the talks I attended. I’m lazy, so I didn’t take notes real time. On one hand, this may be a reason of these notes be not super accurate. On another hand, I had some time to digest the content and only share things that I remember, so the things that resonate with me.

Also, I will share only talks I attended. Not all of them, though, because some were meh. Just keep in mind that there was much more content on the conferences. For example, I missed the entire [CI/CD track at Fosdem](https://fosdem.org/2023/schedule/track/continuous_integration_and_continuous_deployment/), because I’m too dumb to read the schedule properly (plus, there were some talks in other rooms at the same time).

## Fosdem 2023 [Saturday 4th of Feb]

### Drawing your Kubernetes cluster the right way. How to present the cluster without scaring people [Containers devroom]

[Link to the talk](https://fosdem.org/2023/schedule/event/container_kubernetes_cluster_right_way/).

Nice content about how to draw diagrams for complicated systems with an example of Kubernetes.

#### Key points:

- Prefer multiple simpler diagrams instead of a single complex one.
- Use colors (leverage a [color wheel](https://www.canva.com/colors/color-wheel/) to gain the best contrast).
- Do not overload the diagrams with icons. Icons are not universaly readable.
- Do not overload a diagram with arrows if it's not necessary.

### Recipes for reducing cognitive load. yet another idiomatic Go talk [Go devroom]

[Link to the talk](https://fosdem.org/2023/schedule/event/goreducecognitive/)

Very nice talk about how to write better code in Go. Some things may sound trivial, but it's important to remind people of basic things from time to time.

#### Key points on how to make your code better:

- Respect the line of sight. Ideally, your Go code has to have only 2 indentation levels: golden path and error handling. For me this was a very powerful point. I have even recalled some code that I need to refactor once I'm back to work.
- Avoid too generic packages. `Utils` package is evil. Ideally, the package name has to provide some context on what this package does.
- Work with errors as types because they are.
- Try to keep business logic in pure funcitons because they are easier to test.
- Read configuration once and pass it where it's needed. Do not read, for example, environment variables in multiple places of your application code.
- Avoid bare booleans, replace them with named constants. There are code examples in the slides, I don't want to copy it here.
- Go doesn't have method overloading, but in some cases you can use functional options to make your code DRYer.
- Prefer functions over methods because they're easier to test and reuse
- If a function doesn't change an object, pass it as a value not a pointer. It will make your code less error prone.
- Keep logic in synchronous functions, they're easier to test.
- Functions should not lie. If have a `Delete()` functions it should delete something or return an error. See the slides & the video for more details.
- Optimize the readability. Ideally, one should read your code as a newspaper. Put public functions and methods on top of your file, split a package into multiple files when it makes sense.

### Debugging concurrency programs in Go

[Link to the talk](https://fosdem.org/2023/schedule/event/godebugconcurrency/). 

To be honest, I am not that experienced Go developer to comprehend this entire talk. One of the main take aways for me personally was to use a debugger more often.

#### Key points:

- Debugging concurrent programs is not that easy. Never assume a particular execution order.
- Implement concurency on the highest possible level.
- You can add some color to your `goroutine` logs or visualize them in other way.
- Use a debugger (duh!).
- You can set breakpoints inside goroutines.
- Run your programs with `-race` flag in test environments to catch possible race conditions. It won't catch everything but nevertheless might be helpful.
- Low level tools like `strace` may help you if all other methods failed.

### Five Steps to Make Your Go Code Faster & More Efficient

[Link to the talk](https://fosdem.org/2023/schedule/event/gofivestepsefficient/).

An easy to follow algorithm to make your applications more efficient. Moreover, you can apply some of those points not only to Go.

#### Key points:

- Follow the TFBO approach: **T**est => **F**ix => **B**enchmark => **O**ptimize.
- Use `go test -bench` and run benchmarks several times to ensure that diviations caused by external factors (CPU throttling because of low battery, etc.) are not significant.
- Set clear goals before you start optimization. Sometimes, it's perfectly fine for your program to be a little bit sluggish. Without clear goals you will fall into so-called "premature optimization" pitfail.
- Use profiler, for example [`pprof`](https://pkg.go.dev/runtime/pprof) package.
- Understand, what's going on under the hood of your app. Sometimes, generic methods of the standard library or some 3rd party package are too generic to be optimal in your particulr case.

### Kubernetes and Checkpoint/Restore

[Lint to the talk](https://fosdem.org/2023/schedule/event/container_kubernetes_criu/).

There's an upcoming functionality in Kubernetes that will allow you to checkpoint and restore your containers. So, for example, you will be able to move a container to another node or pause/resume it on the same node without losing the state in memory.

If you run jobs in Kubernetes, you know how cool is it! Especially, if those are long running jobs that don't have checkpoints implemented on their own.

Check out the slides and the video for more information and a demo.

### Safer containers through system call interception

[Link to the talk](https://fosdem.org/2023/schedule/event/container_syscall_interception/).

This talk was about LXD containers and AppArmor. If you run LXD, it may interest you. However, I never saw LXD containers in the wild, to be honest.

### Automating secret rotation in Kubernetes

[Link to the talk](https://fosdem.org/2023/schedule/event/container_kubernetes_secret_rotation/).

A talk on how to use tools such as [Kubernetes External Secret Operator](https://external-secrets.io) and [Reloader](https://github.com/stakater/Reloader) to automaticaly rotate secrets in your cluster with a demo.

We are using [Banzai Bank-Vaults](https://banzaicloud.com/products/bank-vaults/) for this. So, practically speaking this talk was not very useful for me personally, but nevertheless it was good. Also, I can clearly see some advantages of the approach proposed there.

### From a database in container to DBaaS on Kubernetes

[Link to the talk](https://fosdem.org/2023/schedule/event/container_kubernetes_database_dbaas/).

Check this out, if you run databases in Kuberentes.

#### Key points:

- It's Ok to run DBs in K8s, more and more people are doing it already.
- Kubernetes operators for DBs are (mostly) fine. Also, it's beneficial to use an operator because it incapsulates some DBA knowledge.
- Cloud-based vendor lock is not Ok. I mean, this is Fosdem conference, after all.

### Effective management of Kubernetes resources for cluster admins

[Link to the talk](https://fosdem.org/2023/schedule/event/sovcloud_effective_management_of_kubernetes_resources_for_cluster_admins/).

This was a talk on [Kustomize](https://kustomize.io/).

I'll be honest with you: I don't get Kustomize. I mean, I understand that Helm is hard and Go Templates are horrible, but each time I'm in a talk about Kustomize, folks promise an easy way of managing stuff and then desribe a bunch of override layers.

However, if you like the tool, you can find some guides of how it's used in RedHat and how can you improve your operations with it.

## Fosdem 2023 [Sunday 5th of Feb]

### Practical introduction to OpenTelemetry tracing

[Link to the talk](https://fosdem.org/2023/schedule/event/tracing/).

A nice demo on how easy is it to adapt [OpenTelemetry](https://opentelemetry.io/) in your application with a demo for a couple of different languages: Java, Python, and Rust.

Like, if you have a distributed system and you don't gather traces, you definetely should start doing it!

### Adopting continuous-profiling: Understand how your code utilizes cpu/memory

[Link to the talk](https://fosdem.org/2023/schedule/event/profiling/).

I wanted to write about continous profiling on [CatOps](https://t.me/catops) some time ago, but didn't.

Basically, the idea is to periodically take snapshots of your app instead of running a full fledged profiler. So, you can observe what's your app is doing without much overhead for profiling. Also, you can do that in production.

However, this talk was mostly about the new Grafana product for continous progiling - [Phlare](https://grafana.com/blog/2022/11/14/watch-how-to-get-started-with-grafana-phlare-for-continuous-profiling/).

### Inspektor Gadget: An eBPF Based Tool to Observe Containers

[Link to the talk](https://fosdem.org/2023/schedule/event/ebpf/).

[Inspektor Gadget](https://www.inspektor-gadget.io/) is an eBPF toolset to get an idea of that your container is doing. It may provide some useful insights into missing capabilities for a K8s pod and so on.

For me this tool was definetely interesting. I'm not sure yet if I ever have a use case for it (I mean, I'm not sure if I ever need to dig so deep) but it's nice to know that such tool exists and have it in yout toolbox.

### Best Practices for Operators Monitoring and Observability in Operator SDK

[Link to the talk](https://fosdem.org/2023/schedule/event/operator/).

I mean, if you have ever written a Kubernetes operator, there probably won't be that much new information for you. Moreover, the points of this talk also exist on the ["Best Practices"](https://sdk.operatorframework.io/docs/best-practices/observability-best-practices/) page of the Oerator SDK documentation.

However, if you just going to write your own operator, those points are really valuable.

### Open Source Software at NASA

[Link to the talk](https://fosdem.org/2023/schedule/event/nasa/).

This is one of the keynotes. The talk is about Open Source technologies that help NASA explore the universe as well as fly a copter-drone on Mars.

Also, this presentation has a lot of amazing images in it!

## Configuration Management Camp 2023 [Mon-Wed 6-8 Feb]

It is much harder to share things from the CfgMgmt Camp, because many videos are not available yet. However, [there are some already](https://www.youtube.com/playlist?list=PLBZBIkixHEic8L17C7DB0I2cY7vO_eDRl).

So, this will be just a short list with some of my impressions.

###  What if Infrastructure as Code never existed?

A keynote by Adam Jacob. He's a harismatic speaker, which is every keynote needs. The talk was about a new concept of Infrastructure as Model instead of Infrastructure as Code. TBH, I haven't fully comprehended the concept. Also, the project is fairly young. So, it would be nice to see a demo of it someday.

{{< youtube 5lPa2U239C4 >}}

### Choosing an Infrastructure as Code Solution

A talk on types of IaC that exist in the wild. The talk is from Pulumi folks, so it's a little bit biased, but still a nice talk.

{{< youtube tKiNJjE1llA >}}

### Set Up a “Production-Ready” Kubernetes Cluster in 5 Minutes

Yet another opinionated Kuberentes setup. The interesting part is this picture that gives you an overview of "branches of concerns" that you may want to have in your cluster as well as some tools that cover those areas.

{{< figure src="https://cfp.cfgmgmtcamp.org/media/2023/submissions/J3TD3S/k8s-bootstrapper-stack.excalidraw_ouCJBHy.png" align="center" >}}

Video:

{{< youtube rwlHMT9ybzM >}}

### Unknown unknowns and how to know them

In the end this is a presentation of a tool that allows you to "map" your infrastructure as an invesitgation goes. However, there are some interesing points in this talk:

- There is no such thing as a "root cause", it's always a combination of factors.
- Unknown unknowns is the most common cause of incidents because you cannot test or monitor things you have no idea about.
- It's nice to know in advance what impact a potential change would have, therefore it's nice to have a way to "query" your infrastructure an relations between its components.

Video:

{{< youtube 3Xk7QRimngk >}}

### Open Policy Agent: security for cloud natives and everyone else

Just a very basic introduction to OPA ([Open Policy Agent](https://www.openpolicyagent.org/)). I already have some articles about OPA on [CatOps](https://t.me/catops), in case you want to dive deeper. I won't duplicate them, you can search them there.

However, the interesting part was a new product by Elastic - [Cloudbeat](https://github.com/elastic/cloudbeat) that allows you to scan your infrastructure against [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/), which uses OPA under the hood.

### Efficient Kubernetes scaling using Karpenter

A nice demo of [Karpenter](https://karpenter.sh/) a relatively new autoscaler for Kubernetes. It's backed by AWS and currently works onlt with AWS. However, this is probably the most powerful all-in-one scaling solution on the market (if we don't account for some proprietary tools). Unfortunately, there's no video available. I'm not even sure that that room was recorded.

### Have you hardened your Kubernetes infrastructure?

The last talk of the first day of the conference. The content was good. There ewere some practical advices on how one can harden their K8s clusters and make workloads that are running there more secure. Also, [slides for this talk are available](https://cfp.cfgmgmtcamp.org/media/2023/submissions/GLZLDP/resources/Have_you_hardened_your_Kubernetes_infrastructure_KGOGBM3.pdf). So, you can easily get this content there.

### CUE: a Glimpse into the Future of Configuration Engineering

I have also written about CUE on CatOps already. Shortly: one ring to rule them all. The idea is that there's a single language to define data structures that has some syntax sugar (otherwise, this is a superset of JSON). Will it be widely adopted? Will see, it has some adoption already. However, it's not clear how people behind CUE are going to secure fundings for their project.

Video:

{{< youtube xOZPOusz4uo >}}

### 10+ years experience of IDPs (Internal Developer Platform) to enable fast growing companies

So, this was a good one! Probably, one of the best talks of the conference. Yes, the speaker may be lacking harisma a bit (also, it looks like he doesn't like K8s for whatever reason :P). Yet, the content is great. I would summarize the main idea in these points:

- An Internal Developer Platform can boost your company's productivity.
- No, you cannot simply insall [Backstage](https://backstage.spotify.com/) and consider it's done.
- However, you can build a solid platform with the things you have: it's all about UX and contracts, not some fancy tools.

Video:

{{< youtube AAh1Lje-eAA >}}

### Kubernetes & The Myth of Cloud Portability

A nice theoretical "case study" of how to igrate a business from AWS to GCP. Some good points:

- Each challenge has more than one solution.
- If one needs portability, once has to optimize for this, which might be operationally costly.
- Also, the talk mentioned some nice tools that I never heard of before, like [Dapr](https://dapr.io/) - a framework to build agnostic apps by "outsourcing" queue communication to Dapr.

### From Monitoring to Observability: eBPF Chaos

This talk was definetely not recorded, but you can find the [slides here](https://docs.google.com/presentation/d/1f8KCUcnC7AZNM34ImOSmvWqWNh8zolavWE0Vw44AqMs/mobilepresent#slide=id.g1287bf62b57_0_209). So, this was about [eBPF](https://ebpf.io/) (obviously) and many things one can do with that both good and bad. Some tools that were shared during this talk (in no particular order):

- [Inspektor Gadget](https://www.inspektor-gadget.io/). Mentioned here already.
- [Caretta](https://github.com/groundcover-com/caretta) - a tool to automatically build a service map from a Kubernetes cluster in Grafana.
- [Coroot](https://coroot.com/) - eBPF based observability tool.
- [Chaos Mesh](https://chaos-mesh.org/) - chaos engineering platform for Kubernetes.
- [Chaos Toolkit](https://chaostoolkit.org/) - a toolkit for chaos engineering experiments.
- [Pixie](https://pixielabs.ai/) - an observability tool by NewRelic.
- [Xpress DNS](https://github.com/zebaz/xpress-dns) - experimental XDP DNS server written in BPF.

Well, that's all, folks! The third day of the Configuration Management Camp (or the fifth day overall) was for the workshops. I listened a little bit about how to build and ship Pulumi packages but since I'm not using Pulumi it was not super valuable to me.

I hope, you have enjoyed this recap! The next conference I plan to attend will be [KubeCon Europe 2023](https://events.linuxfoundation.org/kubecon-cloudnativecon-europe/). If you're also going there and want to have a chat, don't hesitate to ping me. My contacts are [here](https://grem1.in/about/).

{{< figure src="https://www.vhv.rs/dpng/d/511-5117046_thats-all-folks-thats-all-folks-meme-hd.png" align="center" >}}
