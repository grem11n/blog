---
title: "On orchestrators, schedulers, and platforms"
description: "Here I want to wrap up my thoughts and opinions on what's going on on the infrastructure management scene recently and how we ended up like this. And also why I believe that things like Nomad and ECS are not quite a competition to Kubernetes nowadays"
date: 2021-04-11
draft: false
slug: "orchestrators"
image: "https://www.networkcomputing.com/sites/default/files/styles/flexslider_full/public/1-intro_12_0.jpg?itok=a0sMEFXG"
categories: ["Tech"]
tags: ["en", "k8s", "kubernetes", "nomad"]
---

I briefly mentioned this issue in a podcast recently. So, if you understand Russian, you're[ welcome to listen to it](https://youtu.be/GVvYuFy0GJg). However, after I heard the news that Apache Mesos[ is going to be moved to Attic](https://news.ycombinator.com/item?id=26713082), I decided to write this post. Here I want to wrap up my thoughts and opinions on what's going on on the infrastructure management scene recently and how we ended up like this. And also why I believe that things like Nomad and ECS are not quite a competition to Kubernetes nowadays.

Five or six years ago, we were having coffees with a friend of mine. I was working in telecom at that time, and he was already an infrastructure engineer leveraging DevOps practices. A company he worked for was having some kind of a hackathon where he and his colleagues tried out Kubernetes for the first time. They were using Docker Swarm at that time. I recall very clearly, how he shared with me that Kubernetes is an overcomplicated crap, which makes the same thing as Docker Swarm but with additional steps. It's hard to imagine a conversation like this in 2021.

So, how did we come from Kubernetes being a "Swarm with additional steps" to an orchestration mogul it is today?

## What are we even doing?

Which problem are we trying to solve? I'd say the problem of waking up at night. Of course, the whole picture is much broader. It was relatively easy to operate a couple of servers. It's getting harder once you have a hundred, a thousand of them. Headcount shouldn't scale linearly with your systems, yada yada yada.

We want to automate some basic tasks away. A lot of basic tasks actually. So, how did we get to where we are?

## In the beginning, there was Cron

This article is called "On Orchestrators, Schedulers, and Platforms." Hence, we need to talk about schedulers obviously. Or, in the case of Cron, I would rather say pre-scheduler or proto-scheduler. Here is why. The idea is that you schedule a task and then get a result, success or failure. That's it. You don't get it with Cron, though. It triggers a task, and that's it. Sure, a lot of people, including myself, were creating different hacks to overcome that. Usually, email some service addresses the results of execution. That was somehow working with a couple of servers but led to a total mess in the distributed systems.

{{< figure src="https://digitalspyuk.cdnds.net/16/34/980x490/landscape-1472039769-lion-king-rafiki-throws-simba.gif" >}}

## Here comes a feedback loop

With the increasing systems' complexity come real schedulers like Jenkins. A lot of people use to call it a CI/CD system, but it's not just that. Basically, a scheduler allows you not only to start your task but also to get execution results in a centralized place. So, now you have a single control room. Of course, you can still fire a task and forget about it. But the most important part is that now you have a feedback loop embedded into a system.

You can also schedule tasks on events, which made schedulers so popular for CI/CD. Because builds and deploys are basically just tasks that have to be run from time to time.

## Each Action has a Reaction

Now, you can trigger your tasks on events or schedules. You get results from each execution. You can also leverage distributed workers to optimize performance. Just one thing is missing to call it a true orchestration - a reaction.

What should the system do if a task is completed? What should be done if a task failed? Do we need to try again, if yes how many times do we need to try before giving up? By adding an automated reaction based on execution results, you're stepping into the orchestration area.

This is something that HashiCorp Nomad does. You get a task, you allocate this task to a client. If it fails, you can restart or you can just mark it as failed and do nothing. You can watch the logs of your allocation as well. That's it, a task executor, plus feedback loop, plus reconciliation based on feedback - you got an orchestrator.

Some people tend to believe that orchestrators are closely related to containers. That's not completely true. On one hand, the rise of orchestrators happened at the same time as the rise of containers. On another hand, containers are basically isolated Linux processes. So, these things are not interdependent. Moreover, Nomad for example can orchestrate JVM processes, not only Docker containers.

{{< figure src="https://production-cci-com.imgix.net/blog/media/Diagram-01.jpg" caption="via [CircleCI Blog](https://circleci.com/blog/the-feedback-loop-how-to-adapt-to-constant-change/)" >}}

## Let's talk about platforms now

This is intended to be an article about Kubernetes but we haven't talked about it yet. No worries! We are getting there.

Kubernetes solves pretty much the same business problem as Nomad, AWS ECS, or Docker Swarm, or Mesos (RIP). It takes your workloads, runs it on a pool of distributed workers, tracks the execution, and can perform some action based on the execution results. However, there must be something except marketing support from Google that allowed it to become a hegemon of the orchestration landscape.

In the very beginning there was a statement that Kubernetes does the same thing as Docker Swarm, but with a few extra steps. You can find a handful of mocking articles on the Internet, which compares a number of lines of code required to run a simple web server on a server vs in a container vs as a Kubernetes deployment. But what are those walls of YAML exactly?

API definitions. And this is where Kubernetes excels. It doesn't just juggle your containers. It provides you a consistent API to work with it. And the more popular it gets, the more values it brings to the engineers. This is a network effect. Now, you don't have to learn a proprietary toolset that does exactly the same thing each time you change the company. You don't depend on a provider either. If you have experience running your applications on EKS, it would be still relevant when you join a company, which has GKE or a bare-metal installation. You can benefit the same way SysAdmins benefited from Linux taken over the server OS world. Of course, there are differences in operating those platforms, but the previous statement is still relevant.

You can even extend the API! And this is something that excites me the most. Kubernetes has a concept of Custom Resource Definitions. You can create your custom API objects and controllers which allow you to work with that objects. This is exactly why I would rather put Kubernetes under Platforms umbrella rather than Orchestrators. You see, now you have one place to run all necessary dependencies using the same API-building concepts and predictable definitions. You don't just have an orchestrator and a service discovery next to it and then a network layer somewhere in between and then also secret management and dozens of other stuff surrounding it. You have an orchestrator, service discovery, network layer, secret management, and all the surroundings managed by the same orchestrator! And I think it's beautiful!

## But everything comes with a price

The previous paragraph may sound like we have just reached a singularity. Ponies and rainbows everywhere, straightforward declarative APIs to rule everything. Life is wonderful! Unfortunately, this fairyland comes with a price.

Distributed systems are hard, let's not pretend they aren't. And the complexity doesn't go away on its own. You can redistribute the complexity, move it around, throw it over a wall, but you cannot eliminate it. People moved their work to Clouds because they don't need to care about data centers after that. However, it doesn't mean that all the physical servers went away, just someone else taking care of them now.

The same thing happens to Kubernetes. You see, no one needs a bare installation of Kubernetes. You likely will need a handful of other plugins and controllers to operate it as a truly PaaS. And someone has to take care of those plugins. That's why people call Kubernetes "a framework to build Platforms" but not a Platform itself. And I tend to agree with them.

It's not even the biggest problem, to be fair. A lot of companies out there are trying to solve this problem with their out-of-the-box solutions like[ OpenShift](https://www.openshift.com/) or[ Cluster.dev](https://cluster.dev/) or managed Kubernetes solutions (I think every major cloud provider now has one). But then there comes another thing.

You can take a "vanilla orchestrator" and just put it alongside your existing infrastructure. You may need to add some more components to make it work, sure. However, it won't need to re-engineer the whole world. You can have your old workloads running on virtual machines and some new stuff running on top of Nomad or ECS alongside it. You can re-use some existing parts of your infrastructure, say, service discovery, and so on. Unfortunately, it's not the case with Kubernetes. Once you're committed to embracing it, you should be very certain in your desires. You will have to re-architect some parts of your processes and systems entirely. You'll need to train people, you'll need to do a bunch of additional stuff. Remember, you're trying to adopt a platform now, not just another tool. Back in a day, I saw a semi-sarcastic comment in one of the DevOps-ish public chats: "Not every company survives through the adaption of Kubernetes."

{{< figure src="https://www.techrepublic.com/a/hub/i/r/2020/09/10/3e2a29a8-2251-4a98-86a8-6c724b36ff15/thumbnail/770x578/97bb2a95d9258350cf9fdc6fe67a7c62/crazymap.jpg" caption="via [TechRepublic](https://www.techrepublic.com/article/take-a-look-at-the-cloud-native-landscape-if-you-dare/)" >}}

## ...

These thoughts bring me to the conclusion that orchestrators like Nomad or ECS are not quite a competition to Kubernetes nowadays. They were but Kubernetes has outgrown them. And that's not bad at all! Remember, you don't need to adapt a high-end complex distributed system to run your small e-commerce website built on WordPress. It does make sense for a lot of people to move their workloads to Kubernetes, but[ for others, it doesn't](https://mrkaran.dev/posts/home-server-nomad/). I truly believe there is room for both "vanilla orchestrators" and "platforms". Just like you still can use schedulers for their use cases. It brings us to the "fit for purpose" discussion, which is a completely separate topic. Also, I personally think that Nomad fits to run standalone jobs aka batch jobs better than Kubernetes.

So, which technology should you pick? I don't know. Think about what better suits your use case, which tool would fit the purpose. Let's not tighten the screws with a hummer and do the nails with a screwdriver.

_Big thanks to [Maksym Vlasov](https://www.linkedin.com/in/maxymvlasov/) for reading a draft of this._
