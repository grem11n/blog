---
title: "Velero"
description: "Brief overview of Velero (former Heptio Ark) - a backup solution for Kuberentes. Part I of Kubernetes clusters backup story"
date: 2021-09-23
draft: false
slug: "velero"
image: "/img/posts/velero/velero-tumb.png"
categories: ["Tech"]
tags: ["kubernetes", "k8s", "backup", "recovery", "velero"]
---


_**Disclaimer**: Initially, I wanted to write a single article about Velero (former Heptio Ark) and share some opinions on when it makes sense to use it. However, I realized that this article became too broad, and I derived from the original focus. Therefore, I decided to split it into several parts._

_The first one will be just a brief overview of Velero and some of its features since this is the tool that gave a start to the whole idea. Then in the second one, I plan to share my opinions on Kubernetes backups in general. Lastly, in the third part, I would like to talk about some Kubernetes clusters as cattle and the caveats of this approach._

_I'm going to add the links to the 2nd and the 3rd parts once I complete them._

{{< figure src="https://www.eksworkshop.com/images/backupandrestore/velero.png" alt="Velero logo" caption="source: https://www.eksworkshop.com/index.html" class="center" >}}

## What's Velero?

Velero (former Heptio Ark) is a tool for backing up the Kubernetes cluster state. Unlike direct etcd backup, Velero is compliant with a generic approach. it accesses etcd via Kubernetes API server and backs up assets as Kubernetes object.

Apart from the plain Kubernetes resources, Velero can also leverage your cloud's APIs to create snapshots of the Persistent Volues that are present in your cluster. It might be handy if you have stateful applications there.

In a sense, Velero creates a logical backup of your assets, unlike a physical etcd snapshot. It also provides its own CLI to manage backups and restores. Velero uses Custom Resource Definitions for its operations. So, technically you can simply use `kubectl` instead. However, their CLI eases work with backup and restore objects, schedules, backup locations, etc.

## Some Notable Features

Since we work with logical backups here, we can easily define which assets are eligible for backup and which aren't. You can include or exclude whole namespaces in the configuration or using the CLI. You can also leverage Kubernetes annotations to include or exclude assets from backups in a more granular way.

You can also create schedules for your backups, so they're taken automatically. As well as configure TTL for backup files, so Velero will keep track of your backup versions and clean up the old stuff.

You can create snapshots of your Persistent Volumes, which can be handy if you're running some stateful applications in your cluster. However, I'm not a big fan of running any stateful things inside Kubernetes (more on this in Part III).

There are backup and restore hooks to do some actions before or after any of those operations. For example, you can stop a database service before creating a volume snapshot, so you can be sure of data integrity.

You can even use Velero to do cluster migrations! You can "take a snapshot" of your current cluster state and move it to another cluster. They even advertise this feature on their website. I will look closer at cluster migration in Part III.

That's it, I guess. I mean, what other features do you want from a backup tool from a user perspective?

Also, I recommend you watch this video. It's old, but it's a great entry-level introduction to Velero capabilities.

{{< youtube "qRPNuT080Hk" >}}

Also, you can look through [Velero documentation](https://velero.io/docs/). It provides a general overview of all its features as well as the instructions to configure it. It's not always clear, but good enough to get the thing up and running.

## Others

I saw a few mentions on the Web that there are many similar solutions, but I personally managed to find only three reasonable alternatives: etcd snapshot, [Kasten by K10](https://www.kasten.io/product/), and [Portworks](https://portworx.com/kubernetes-backup/).

BTW, Portworx has written a kinda overview of Kubernetes cluster backup tools. However, part of those tools are not for backup at all, but for management of the persistent layer. Also, this is clearly a marketing write-up. I will leave [the link](https://portworx.com/kubernetes-backup-tools/) here, just to indicate that I saw this article and generally disagree with it.

### Why not etcd snapshot?

This is a very good question. If your cluster is created with kops, you likely have `etcd-manager` installed already. So, it's super simple to make a backup. You don't need any other tools for this as well. No CRDs, additional CLIs, etc.

However, there are dragons. See, no one needs backups for the sake of backups. Their primary attribute is to be restorable. I know that all of us hope that the "restore day" will never come, but when it does you'd better be covered.

And it happened to be that [restoring an etcd backup is not the easiest process](https://rudimartinsen.com/2020/12/30/backup-restore-etcd/) there are [a few manual steps](http://www.opslib.com/2020/10/etcd-backup-and-restore-cka-exam.html) involved and you may need to restart etcd pods in the process. This is Ok for the CKA exam, but in real life, you want to have as few manual steps as possible. Moreover, you would likely want to automate yourself from the backup and restore procedure at all! So, whoever happens to be a misfortunate person to deal with cluster restore has to simply trigger a CI job or something like that.

### Logical backups

Taking into account what's been said about etcd snapshots, logical backups seem like the best approach, IMHO. Velero does that. Kasten and Portworx do that as well. Both Kasten and Portworx are proprietary paid solutions, while Velero is open source. Yes, technically there is a free tier for Kasten, but you can have a maximum of 10 nodes in your cluster, which is a joke.

## Installation

Is super straightforward. You can simply use [the official Helm chart](https://github.com/vmware-tanzu/helm-charts/tree/master/charts/velero) for that. Alternatively, you can use Velero CLI to install Velero, but I prefer the Helm way, especially if there is automation on top of Helm already.

One thing that's not that obvious in the chart is plugin installation, though. Velero [uses plugins](https://velero.io/plugins/) cloud APIs and these plugins are installed using an [init container](https://github.com/vmware-tanzu/helm-charts/blob/master/charts/velero/values.yaml#L26) in the official chart. Without a plugin for your cloud provider, Velero will still startup, but won't be able to do anything meaningful. I actually spent some time figuring it out. Unfortunately, there are not many examples in the values file. So, you have to consult with the documentation from time to time. Even then, though, you may not find examples for some exotic clouds and use cases.

Speaking of plugins. This is a great way to ensure flexibility and extendability for your tool. But again, if you have an exotic setup, some plugins may be tricky to use. Also, there might be a lag between new Velero versions and the versions for some plugins. Although, if you're working with one of the major cloud providers, you should be covered.

Also, depending on how do you allow the applications inside a cluster to connect to your cloud API, you may need to create an additional configuration for things like [IRSA](https://medium.com/getamis/aws-irsa-for-self-hosted-kubernetes-e045564494af) or [Kiam](https://github.com/uswitch/kiam). Alternatively, you can provide the credentials as a Kubernetes Secret, of course. However, I warn you not to do so!

## Velero in Action

After you installed a new thing is the best time to test it! Remember, we are talking about logical backups here.

We basically need to read data from the Kubernetes API, compact it, and store it in some pre-defined location. In general, these operations shouldn't take a long time and they don't. I have tried to back up a little bit more than 3600 various Kubernetes objects and it took less than 3 minutes in total!

Restore is a different topic, though. Again, from Velero's perspective restore process is straightforward and happens in a few minutes as well. It reads files from the storage, unpacks them, and applies retrieved objects to Kubernetes API. Some hooks may be executed before and after, which can contribute to the overall restoration time, but it still doesn't take long to restore Kubernetes objects.

However, the presence of your objects in etcd doesn't mean that your services are running! Now it's time for the Kubernetes scheduler to take action to allocate actual resources. Things can go tricky here. How long does it take for your service to start? Are there many services that have to be restored? What are the relations between them? You will likely need to pull a lot of images into your freshly restored cluster, can your registry deal with it, are there any rate limits?

Of course, Kubernetes is an eventually consistent system. So, your cluster should be Ok in some time. Moreover, you probably can take those odds in case of a disaster. However, you need to keep in mind those caveats whenever you need to restore a cluster or migrate to a new one.

This brings us the second part of this story: _does it even makes sense to back up a Kubernetes cluster?_

_Many thanks to [Maksym Vlasov](https://www.linkedin.com/in/maxymvlasov/) for his inputs and editing efforts!_
