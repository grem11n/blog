---
title: "Building a CLI application in Go: Part 0"
date: 2022-09-02
draft: false
slug: "go-cli-part0"
cover:
  image: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQmLZiX5NYbxQ-NMaydrKXbupacqbDmClXAG7qji6lfknKIB6NDRjRPm2W7j8DNtOgKJyw&usqp=CAU"
categories: ["Tech"]
tags: ["go", "golang", "en", "cli", "pet-project", "howto"]
---

# Building a CLI application in Go: Part 0

{{< figure src="https://go.dev/blog/gopher/header.jpg" align="center" >}}

## Intro

I have written a [tiny CLI app](https://github.com/grem11n/s3bc) that can update the storage class of objects in an AWS S3 Bucket. To be completely honest, this tool is rather useless in the wild. You can achieve the same results natively with [AWS S3 Lifecycle policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html).

However, this app is my opportunity to talk about writing CLI applications in Go as well as speculate about pet projects in general.

So, this is the first or rather part zero of a tiny series about CLI applications in Go. I will try to cover some topics listed below with S3BC as an example and walk you through some struggles I’ve encountered along the way.

_This is an ongoing series. I will add links to the respective topics as I write._

- Part 0: Motivation <= You are here. In this article, I’m talking about what was my motivation to write S3BC and why I have chosen the technologies I use.
- Part 1: Initiating a CLI application in Go. Cobra framework and the directory structure.
- Part 2: AWS client and some unit tests.
- Part 3: E2E tests. [Localstack](https://localstack.cloud/) and [Minio](https://min.io/).
- Part 4: Automating all those tests in CI (GitHub Actions).
- Part 5: Automating releases with [GoReleaser](https://goreleaser.com/).

## Let me tell you a story…

So, back in 2016, a tech giant acquired the company I worked for. People who went through acquisitions know that before this IT departments spend a lot of time on cost savings activities.

We had a lot of S3 buckets with various data. Mostly, some ML stuff. My task was to convert all those objects to the `REDUCED_REDUNDANCY` storage class to save some bucks. TBH, I don’t remember if one could use AWS S3 Lifecycle policies for that back then, or I just didn’t know they exist. Anyways, I decided to write a simple script to bulk convert all those things.

I chose Go back then because it was gaining momentum in the DevOps-ish circles and I wanted to get some experience with it. This is how [rrs-converter-go](https://github.com/grem11n/rrs-converter-go) was born. The script was generic enough, so I put it on GitHub. Also, my teammates were not against that.

**Warning!** Do not use that script! First of all, I’m not sure if it still works. Second, it has a bug. I was able to spot it at some point, but my AWS SDK knowledge was very limited, hence I didn’t know how to fix it. I will cover this in more detail in Part 3.

I had some time this summer to do whatever, so I thought: why not to re-write that script in a normal way this time? This brings us to the topic of pet projects.

## Why would you spend your time on some useless tool?

Have you ever got a message from a recruiter: “I liked your GitHub profile very much…?” You check your GitHub profile and find only some rotten semi-finished projects from 2015 there.

Frankly, there are many motivating factors to write some lightweight projects in your spare time. Notice, that I’m not talking about potential startup ideas here or anything like that.

- You can re-sharpen your coding skills. Those could get rusty With time. Especially if you’ve moved to positions, which require more of your time for project or people management.
- You can learn new things. Technology evolves very fast. Even if you’ve done this before, new libraries or approaches might have appeared.
- Passive test assignment. I never had a situation like this. However, I suppose it’s Ok to tell the potential employer something like: “Look, I don’t really have time for a test assignment right now, but you can check my projects on GitHub to get an idea of what I’m capable of”.
- Have a cheatsheet at hand. It’s cool to have some code that you can refer to in the future. Not just the app code itself. Other “supporting stuff” can also be important. For example, how to output that fancy help message in the `Makefile` or how to configure those test dependencies.
- Last, but not least: you can actually help someone. When HashiCorp just released their Terraform modules repository, I wanted to push something there to see how it works. I didn’t come up with anything more creative than writing a module to automate a task I was doing at that moment e.g. configuring VPC peerings in AWS. Now, [terraform-aws-peering-module](https://github.com/grem11n/terraform-aws-vpc-peering) has 87 stars and 12 contributors and was forked 75 times. I would never think that it could become that popular when I hit that push button back then.

## Now, back to the current project

When choosing what to do here, I used the same ideas that ThePrimeagen revealed in his video on pet projects.

{{< youtube KjjjQSSbhfU >}}

Here are the top three features of a good pet project in my opinion:
- Improve skills relevant to your career or be an opportunity to learn those. It doesn’t make sense to build a React application if you have no intentions to move to Web development.
- Do not non-overwhelm yourself. [Feature creep](https://en.wikipedia.org/wiki/Feature_creep) kills even commercial products. Let’s be honest with ourselves: it doesn’t make sense to start a new OS unless you have strong motivation to finish it. Otherwise, this would become yet another semi-finished abandoned project on GitHub.
- It has to be fun! This is your private time that you can otherwise spend on random YouTube or Netflix watching. It is very silly to spend this time on something that doesn’t bring you joy!

With these three criteria in mind, I started working on S3BC. Let’s see if it checks the points:
- **Skills improvement & relevance**. Probably more than 80% of the code I’ve ever written is CLI applications. So, being able to quickly spawn a new one or figure out what’s going on in an existing app is somewhat crucial. Of course, I could choose Rust instead of Go for that. Especially, after Meta released have [recommended Rust for CLI applications](https://engineering.fb.com/2022/07/27/developer-tools/programming-languages-endorsed-for-server-side-use-at-meta/) in their memo. Also, I’ve written CLI apps in Go for my daily job before, so this is somewhat familiar to me. However, this brings us to the next criterion.
- **Well-scoped non-overwhelming project**. I’ll be honest, I’m not a full-time developer. I like writing code, but this is not the main thing I’m being paid for. Also, I don’t know Rust. It would be quite hard for me to spawn any kind of a more or less functional CLI application from the first try. I’d better go make some LeetCode challenges first to get myself more familiar with the language. Moreover, I don’t think we would have anything in Rust in my team in the nearest future. So next time…
- **Fun**. Was it enjoyable to create S3BC? Yes, somewhat. I enjoyed the process overall. Also, now I better understand Cobra. I was also able to dig a bit into the local end-to-end testing - a somewhat underrated topic for small tools development. Also, re-writing an old app brings some nostalgia. I could appreciate that.
- **Bonus points**. Even though the app itself is not that exciting, I think this is a wonderful thing to share with y’all. It’s easier to show some concepts on simpler examples. Thus, I decided that this tiny tool is a perfect candidate for this series of articles. The scope is not too broad, but at the same time, this is a functional application that you can take and use.

## Los geht’s

With this being said, see you in Part 1, where I’m talking about some initial setup and the first steps of building a CLI app with Go!
