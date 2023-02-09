---
title: "A year with Kubernetes: Recap"
description: "I’ve been using Kubernetes for almost a year and want to share some takeaways and things I’ve learned about it so far"
date: 2019-06-26T13:15:47+02:00
draft: false
slug: "k8s-year"
cover:
  image: "https://www.twistlock.com/wp-content/uploads/2017/01/kubernates-logo.jpg"
categories: ["Tech"]
tags: ["kubernetes", "eng"]
---

{{< figure src="https://techcrunch.com/wp-content/uploads/2018/01/mvimg_20180129_115533.jpg?w=1390&crop=1" class="left" caption="[Image source](https://techcrunch.com/2018/01/30/heptio-launches-its-kubernetes-un-distribution/)" >}}

I've been using Kubernetes for almost a year and want to share some takeaways and things I've learned about it so far.

Some context. We had a team of two engineers working not only on Kubernetes but on all infrastructure-related topics. Our main application was a Python monolith, plus a few satellite-services. Our infrastructure was dynamic due to autoscaling, and a lot of things were optimized for costs (check out my article about [running Kubernetes on AWS Spot instances](https://medium.com/preply-engineering/why-and-how-do-we-run-kubernetes-on-the-spot-instances-c88d32fb9df3)).

We have completed the path from an early-days PoC to running our main application on top of Kubernetes in production. It took us 11 months to achieve that. Since our estimates were about one year, 11 month seems to be all right, though. Thus, takeaway #1:

Do not overcommit
----
Setting up and battle-test the cluster may take more time than you expect even taking into account all the automation around Kubernetes (we are going to talk about the automation further). Also, if you need some specific needs, it might take even longer because your case might be not covered by tutorials or ready-to-use automation. Side note: no one in the team had any experience with k8s before. So, here is the next takeaway.

Courses are rather helpful
----
Kubernetes itself has some excellent documentation. Also, there are literally thousands of related blog-posts and rich tolling. However, it's super easy to get lost in all of this. Also, let's be honest, we do not read the documentation from A to Z usually. Sometimes we just find the part we need and skip everything else. A course can structurize this knowledge and present all of the abstractions as a single story without jumping from topic to topic. Moreover, more and more Kubernetes courses are available online, some of them paid while others are free. Also, it's easier for course-makers to follow up on all the updates made in the Kubernetes itself, which make them somewhat more up to date than the books.

Though courses are an optional thing. We didn't take any before spinning up our clusters. However, I watched a few of them, and it was like: *"Wow! If someone told me all these things before we started, there would be definitely less pain!"*

Who automates the automation?
----
Kubernetes "automation" tools require some gruntwork. For example, a very popular cluster management tool -- [kops](https://github.com/kubernetes/kops) requires setting up [IAM, DNS, and configuration storage](https://github.com/kubernetes/kops/blob/master/docs/aws.md) in advance. Be sure you have scripted these things as well.

Basically, you can say that you have a fully automated cluster setup only if you can terminate a cluster with one command or a script run and then bring it back the same way ¯\\\_(ツ)\_/¯

Managed solutions to the rescue!
----
There are plenty of managed solutions for Kubernetes nowadays. All the major cloud providers have them. Of course, a managed solution limits your freedom: you cannot change API server configuration, probably change the network plugin, etc. But here is the trick! _**Running your own control plane likely won't add any business value**_. Unless, you have some specific scenario and you understand what you're doing.

{{< tweet user="jessitron" id="1059632684098478080" >}}

Speaking further about the value. You don't want just a cluster on its own. Eventually, you will need things like Ingress controller, monitoring & logging stack (including monitoring & logging of the cluster itself), [cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler), perhaps even a service mesh, etc. There are tons of tutorials on how to set up all of these. A lot of things could be installed with a simple `helm install ...` command, but there is a catch!

Automate the service dependencies
----
> Treat each `helm install ...` run as a  `apt/yum/dnf install ...` run.

You likely have a CI/CD and a fully reproducible delivery flow for your applications, don't you? You need to treat internal dependencies accordingly:

* Pin the versions of each module (Chart, talking about Helm)
* Automate the installation of the related modules
* Do not change anything manually

By "manually" I mean, do not change the resources "on the fly" using `, kubectl edit ...` or `helm upgrade ... --set foo=bar`, etc. There are tools, which may help you here. For example, [skaffold](https://skaffold.dev/), which supports deploying remote Helm charts starting from the [`v0.30.0` version](https://github.com/GoogleContainerTools/skaffold/blob/master/CHANGELOG.md#v0300-release---05232019).

Sure, all your dependencies' settings are in YAML, but sometimes you need to execute these yamls in _**"some specific order"**_. Tools like Skaffold might help you with this.

Backup and then backup again
----
Although you can re-create everything from scratch, even if you're running on the provider-managed Kubernetes cluster, it worth to backup. Because it's always good to backup :)

Sure, cloud provider may take a lot of underlying work from you, but we are all humans and incidents do happen.

[Velero (former Heptio Ark)](https://github.com/heptio/velero) could help you with that.

Have your own playground cluster
----
Now, when you have all of this up and running, you may wonder about the cluster updates. Honestly, the only safe way to upgrade the major version of Kubernetes for us was a "blue/green cluster upgrade." We set up a new cluster with all the dependencies nearby and switched traffic there. This makes me think that you may need more than two clusters:

* A production one to run your workloads
* A staging/test cluster for internal users e.g., developers
* A dev cluster for your infrastructure team to play with the version upgrades of the underlying services including Kubernetes itself

The last one may be dynamic i.e., you spin it up only when you are going to test a cluster upgrade or a new version of the ingress controller. A dynamic cluster will serve as additional insurance that you actually can re-create your clusters from scratch.

Get the value as soon as possible
----
Speaking of the clusters for staging. As I said at the very beginning of this article, it took us 11 months to deliver Kubernetes to production. However, it doesn't mean that you cannot get the value earlier. In our case, we were able to provide on-demand testing environments for each feature in approximately 20 minutes, which gave a significant boost to our release process. There are also a few non-technical benefits of using Kubernetes -- it's a super hot topic nowadays. Thus:

* a new hire may already know about this thing and have some experience with it
* there are people on the market, who want to work with it (Kubernetes is good for resume, hehe). We had an interview with the person, who literally said: *"I've heard, you are using Kubernetes? I want to use it as well..."*

**Side note**: I do not support a resume-driven development approach, but so are the facts.

Wrapping it up
----

Honestly, I enjoyed this new k8s experience (despite all the occasional suffering). And if you ask me for one thing I've learned here, it would be this.

_Kubernetes is not a cure for all your problems, it's even not a "tool" in general sense. However, this is a framework, which allows you to leverage some of the human knowledge and infrastructure best practices in a more or less standardized way regardless of the underlying platform. It won't implement the best practices for you, though_.

And a bunch of `yamls`, of course :)
