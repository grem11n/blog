---
title: "A year with Kubernetes: Recap"
date: 2019-06-26T13:15:47+02:00
draft: false
slug: "k8s-year"
image: "https://www.twistlock.com/wp-content/uploads/2017/01/kubernates-logo.jpg"
categories: ["Tech"]
tags: ["kubernetes", "eng"]
---

{{< figure src="https://techcrunch.com/wp-content/uploads/2018/01/mvimg_20180129_115533.jpg?w=1390&crop=1" class="left" caption="Source: https://techcrunch.com/2018/01/30/heptio-launches-its-kubernetes-un-distribution/" >}}

I've been using Kubernetes for almost a year and want to share some takeaways and things I've learned about it. A little about our background. We had a team of 2 engineers working not only on Kubernetes but on all infrastructure-related topics. Our main application was a monolith written in Python, plus a few satellite-services. Our infrastructure was dynamic due to autoscaling, and a lot of things were optimized for costs (check out my article about [running Kubernetes on AWS Spot instances](https://medium.com/preply-engineering/why-and-how-do-we-run-kubernetes-on-the-spot-instances-c88d32fb9df3)).

We have completed the path from an early-days internal PoC to running our main application on top of Kubernetes in production. It took us 11 months to achieve that. Since our estimates were about 1 year, 11 month seems to be a right timing, though. Thus, takeaway #1:

Do not overcommit
----
Setting up and battle-test the cluster may take more time than you expect even taking into account all the automation around Kubernetes (we are going to talk about the automation further). Moreover, no one in the team had any experience with k8s before. So, here is the next takeaway.

Courses are rather helpful
----
Kubernetes itself has some excellent documentation. Also, there are literally thousands of related blog-posts and rich tolling. However, it's super easy to get lost in all of this. Also, let's be honest, a lot of us do not read the documentation from A to Z. Sometimes we just find the part we need right now and move further. A course can structurize this knowledge and present all of these as a single story without jumping from topic to topic. Moreover, more and more courses on Kubernetes are available online, some of them paid while others are free. Also, it's easier for course-makers to follow up on all the updates made in the Kubernetes itself, which make them somewhat more up to date than the books.

Though courses are an optional thing. We didn't take any before spinning up our clusters. However, I watched a few of them, and it was like: "Wow! If someone told me all these things before we started, it would have taken less time to set up everything!"

I have already mentioned the automation, so let's focus on this now.

Who automates the automation?
----
Kubernetes "automation" tools require more automation. For example, a very popular cluster automation tool -- [kops](https://github.com/kubernetes/kops) requires setting up [IAM, DNS, and configuration storage](https://github.com/kubernetes/kops/blob/master/docs/aws.md) in advance. Be sure you have automated these steps as well.

{{< tweet 1114007653011689472 >}}

Basically, you can say that you have a fully automated cluster setup only if you can terminate a cluster with one command or a script run and then bring it back the same way ¯\\\_(ツ)\_/¯

Managed solutions to the rescue!
----
There are plenty of managed solutions for Kubernetes nowadays. All the major cloud providers have them. Of course, a managed solution limits your freedom: you cannot change API server configuration, probably change the network plugin, etc. But here is the trick! *Running your own control plane perhaps won't add any business value*.

{{< tweet 1059632684098478080 >}}

Speaking further about the value. You probably don't want just a cluster on its own. Eventually, you will likely need things like Ingress controller, monitoring & logging stack (including monitoring & logging of the cluster itself), [cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler), perhaps even a service mesh. There are tons of tutorials on how to set up all of these things. A lot of things could be installed with a simple `helm install ...` command, but there is a catch!


Automate the internal services
----
> Treat each `helm install ...` run as a  `apt/yum/dnf install ...` run.

You likely have a CI/CD and a fully reproducible delivery flow for your applications, don't you? You need to treat internal dependencies accordingly:
* Pin the versions of each module
* Automate the installation of the related modules
* Do not change them manually

By "manually" I mean, do not change the resources "on the fly" using `, kubectl edit ...` or `helm upgrade ... --set foo=bar`, etc. There are tools, which may help you here like [skaffold](https://skaffold.dev/), which supports deploying remote Helm charts starting from the [`v0.30.0` version](https://github.com/GoogleContainerTools/skaffold/blob/master/CHANGELOG.md#v0300-release---05232019).

Sure, all your dependencies settings are in YAML, but sometimes you need to execute these yamls in *"some specific order"*. Tools like Skaffold might help you here.

Backup and then backup again
----
Although if you can re-create everything from scratch, even if you're running it worth to think about the backups. Because it's always good to think about backups :)
[Valero (former Heptio Ark)](https://github.com/heptio/velero) could help you with that.

Have your own playground cluster
----
Now, when you have all this up and running, you may wonder regarding the cluster updates. Honestly, the only safe way to upgrade to the major version of Kubernetes was a "blue/green cluster upgrade." We set up a new cluster with all the dependencies nearby and switched traffic there. This makes me think that you may need more than two clusters:

* A production one to run your workloads
* A staging/test cluster for internal users e.g., developers
* A dev cluster for your infrastructure team to play with the version updates of the underlying services

The last one may be dynamic i.e., you spin it up only when you are going to test a cluster upgrade or a new version of the ingress controller. A dynamic cluster will serve as additional insurance that you actually can re-create your clusters from scratch.

Get the value as soon as possible
----
Speaking of the clusters for staging. As I said at the very beginning of this article, it took us eleven months to deliver Kubernetes to production. However, it doesn't mean that you cannot extract the value from it before. In our case, we were able to provide on-demand testing environments for each feature in approximately 20 minutes, which made a significant boost for our release process. There are also a few non-technical benefits of using Kubernetes -- it's a super hot topic nowadays. Thus:

* a new hire may already know about this thing and have some experience with it
* there are people on the market, who want to work with it (Kubernetes is good for resume, hehe). We had an interview with the person, who literally said: *"I've heard, you are using Kubernetes? I want to use it as well..."*

**Side note**: I do not support a resume-driven development approach, but so are the facts.

Wrapping it up
----

Honestly, I enjoyed this new k8s experience. And if you ask me for one thing I've learned here, it would be this.

Kubernetes is not a cure for all your problems, it's even not a "tool" in general sense. However, this is a thing, which allows you to leverage some of the human knowledge and infrastructure best practices in a more or less standardized way regardless of the underlying platform. It won't implement the best practices for you, though.

And a bunch of yamls, of course.
