---
title: "Kubernetes Operations Survey 2023"
date: 2023-06-02
draft: false
slug: "k8s-survey-2023"
cover:
  image: "/img/posts/k8s-survey-2023/catops.png"
categories: ["Tech"]
tags: ["kubernetes", "k8s", "en", "survey", "catops"]
---

# Annual Kubernetes Survey by CatOps

It is the second time I run this survey. You can find the previous year's results [here](https://grem1.in/post/k8s-survey-2022-1/). 
You can find the raw data [in the Google Sheets](https://docs.google.com/spreadsheets/d/1rR8bcKi1NnQpbnOA9zVK6IhV89hTr4aJdi8O3Rcb-mY/edit?usp=sharing).
You can also [read this article on Substack](https://newsletter.catops.dev/p/kubernetes-operations-survey-2023).

## Introduction

This year, I ran this survey for two months: from the 14th of March to the 18th of May, and gathered 122 responses in total. Yet, I must admit that I only promoted this survey for a month. There are more responses than the last year (122 vs. 102). Some of the channels to share this survey were different this time. Also, I started later than the previous year (spring vs winter). Although, this difference should not impact the results.

Many questions in this survey are the same as in the previous one. I had to change a couple of them because they were not clear enough. In any case, it should be possible to compare the results of this survey with the previous year.

I provide the absolute number of responders and the percentage for each answer. A percentage is in (parentheses).

## Cluster Types and the Number of Clusters

{{< figure src="/img/posts/k8s-survey-2023/01_number-of-clusters.png" >}}

48 (39.3%) of responders use multiple static clusters for different teams or departments. This number has grown since the last year. 29 (28%) of the responders selected this answer the previous year. The percentage of people using a single static cluster per environment also increased: 35 (34%) in 2022 vs. 44 (36.1%) in 2023.

18 (14.8%) of responders mentioned that they use dynamic on-demand clusters.

In this question, I did a poor job covering multi-region scenarios. Thus, some people have used the field "Other" for multi-environment / multi-region setups. I will add this option to the following survey. Even though there were just a few such responses, I believe some folks selected the closest answer.

## Which Kubernetes Distribution Do People Use

{{< figure src="/img/posts/k8s-survey-2023/02_k8s-distros.png" >}}

The majority of the responders are using a cloud-managed Kubernetes solution - 99 (81.1%). One could argue that Cloud-managed solutions are not distributions on their own. Yet, based on the last year's survey, I decided to add them as such.

The second choice is vanilla Kubernetes with 30 (24.6%) votes. 

Rancher has both the third and the fourth places, with RKE(1&2) being selected by 14 (11.5%) and k3s with 12 (9.8%) of the replies. 

Notice that this was a multi-choice question. Some might have chosen "Vanilla Kubernetes" and "Cloud-managed." Also, some people are using various distributions at the same time. Thus, the total percentage is more than 100%.

Some have clarified that they use multiple "distributions" because they are migrating to the cloud-managed one.

Another reason for running a mix of cloud and on-premise clusters is a hybrid infrastructure which consists of on-premise data centers and a cloud for compliance or other reasons.

And my favorite reason:

> Company is stupid

## Cloud-Managed Solutions

96 (78.7%) of the responders use a cloud-managed solution. 51 (41.8%) manage the clusters on their own and only 3 (2.5%) have a 3rd-party vendor to manage the clusters for them.

{{< figure src="/img/posts/k8s-survey-2023/03_cloud-clusters.png" >}}

{{< figure src="/img/posts/k8s-survey-2023/04_cloud-vendors.png" >}}

This is an implicit way to understand the popularity of different clouds.

Quite predictably, AWS is the most popular: 68 (68%).

GCP GKE is in second place with 37 (37%) replies, and Azure AKS is third with 28 (28%).

These results contradict the [Cloud infrastructure services vendor market share statistics](https://www.statista.com/statistics/967365/worldwide-cloud-infrastructure-services-market-share-vendor/). According to it, Azure is in the 2nd place by popularity with GCP being 3rd. It might have happened that there are more GCP users in the channels I used to share this survey. Another possible explanation is that Azure users are less likely to use managed AKS clusters. To answer this question, I need to ask what cloud providers people use before asking about managed clusters. Yet, these results align with the last year's responses.

## Kubernetes Versions

{{< figure src="/img/posts/k8s-survey-2023/05_k8s-version.png" >}}

It's a tight game here. 32 (26.2%) use the latest available version (a cloud provider might not support the latest version yet). 

27 (22.1%) use a mix of different versions. 

26 (21.3%) use an older version of Kubernetes that has not yet reached EOL. 

11 (9%) people are using the latest minor version minus one. I have this option because sometimes this is a corporate policy.

13 (10.7%) of the responders have replied that they use a non-maintained version of Kubernetes. This number has increased compared to the last year (6 (5.9%)). This might be because different people participated in this survey. Or because more people use Kubernetes and cannot keep up with its release cycle.

Only 10 (8.2%) people keep us with the latest patches, which is also a little bit higher compared to the previous year (6 (5.9%)).

## Kubernetes Upgrades

{{< figure src="/img/posts/k8s-survey-2023/06_how-often-upgrade.png" >}}

More than half of the respondents upgrade their clusters' version when they need to - 68 (55.7%). Some people have replied that they upgrade when the version is going EOL or when they have time or budget for an upgrade.

27 (22.1%) of respondents upgrade their clusters regularly, and 26 (21.3%) when a new eligible version is available. Notice that "eligibility" may mean different things for different people.

1 (0.8%) person has replied that they have stuck with an unsupported version of Kubernetes due to the "strange demands."

{{< figure src="/img/posts/k8s-survey-2023/07_upgrades-timescale.png" >}}

On the timescale, 38 (31.7%) of respondents upgrade the Kubernetes version at least once each half a year and 36 (30%) at least once a quarter.

19 (15.8%) of respondents upgrade their clusters once a month or even more often, and 20 (16.7%) do it at least once a year.

7 (5.8%) of respondents have replied that they upgrade the Kubernetes version less than once a year.

## How Do You Upgrade Kubernetes Version

{{< figure src="/img/posts/k8s-survey-2023/08_how-upgrade.png" >}}

The vast majority of respondents upgrade their Kubernetes version in place - 103 (84.4%). Only 16 (13.1%) create a new cluster and migrate workloads there.

Some people use a mixed approach depending on the criticality and running workloads - 3 (2.5%).

One person mentioned that they currently perform upgrades in place but are evaluating changing the approach.

These results align with the last year's responses. The majority of folks upgrade the Kubernetes version in place. Percentage-wise, these numbers didn't change much, either.

{{< figure src="/img/posts/k8s-survey-2023/09_migration-upgrade.png" >}}

Most people who migrate workloads to a new cluster use GitOps for this - 45 (68.2%). 29 (43.9%) use their custom automation.

Yet, some folks migrate workloads manually - 13 (19.7%) or ask respective teams to redeploy - 5 (7.6%). These numbers a lower compared to the previous year.

## Automation for Kubernetes Upgrades

{{< figure src="/img/posts/k8s-survey-2023/10_upgrade-automation.png" >}}

Almost half of the respondents - 58 (47.5%) have semi-automated upgrades. It means that automation exists, but some manual work is still required.

28 (23%) of the respondents have their Kubernetes version upgrade fully automated.

And for 36 (29.5%) of the respondents, it's still a manual process.

Surprisingly, more people replied last year that their Kubernetes version upgrade was fully automated, and fewer responded that this is a manual process for them. Either folks got more honest this year, or I hit different demographics.

{{< figure src="/img/posts/k8s-survey-2023/11_upgrade-tools.png" >}}

Unsurprisingly, a lot of people are using cloud vendors' tools to upgrade their Kubernetes version - 82 (54.7%).

24 (16%) of the respondents use some 3rd-party tool to automate cluster upgrades, open source or not. Among these tools are kOps, Kubespray, KubeOne, and, of course, Terraform.

18 (12%) have in-house automation for cluster upgrades, and 16 (10.7%) do not use automation tools. Cluster API is still not widely adopted, with only 10 (6.7%) respondents using it.

There is a discrepancy between the number of people who answered that their upgrade process is not automated and those who responded that they don't use automation tools. Likely, it's because cluster setup tools often require someone to operate them.

## Core Components

I use the term "core components" for all the plugins that allow one to run a functional Kubernetes cluster, such as Ingress Controllers, Observability tools, GitOps operators, etc.

{{< figure src="/img/posts/k8s-survey-2023/12_core-components-align.png" >}}

71 (58.2%) respondents have answered that they manage the core independently from the cluster lifecycle.

Yet, 32 (26.2%) respondents align the lifecycle of some core components with the cluster lifecycle.

16 (13.1%) respondents roll out a new cluster each time they need to upgrade a core component.

And some people have struggled to answer this question.

On the timeline it looks like the following:

{{< figure src="/img/posts/k8s-survey-2023/13_core-upd-freq.png" >}}

- 65 (53.3%) don't have any specific policy regarding core components' upgrades.
- 22 (18%) upgrade when there is a critical bug or vulnerablity.
- 21 (17.2%) upgrade their core components on a fixed cadence.
- 13 (10.7%) do it when a new version of the component is available.

Speaking of the tools people use to install the core components:

- Helm is in first place with 89 (73%) users.
- Terraform is in second place with 82 (67.2%)
- ArgoCD is the third with 47 (38.5%)
- FluxCD is twice less popular as ArgoCD, with 21 (17.2%) users
- Unsurprisingly, there is a tail of other tools that people use to automate the installations of the plugins.

{{< figure src="/img/posts/k8s-survey-2023/14_core-tools.png" >}}

Notice that the sum is more than 100% because people could choose multiple answers.

## Differences Between Environments

{{< figure src="/img/posts/k8s-survey-2023/15_env-diff.png" >}}

Speaking of the difference between environments (dev, staging, production). The majority of respondents, 97 (79.5%), keep the same versions of both Kubernetes and core components. This includes people who run all the environments in the same cluster (1 person) and the people who create clusters on demand. This is a good sign, and it shows that Kubernetes brings us closer to the similarity between development and production environments.

11 (9%) of respondents keep the Kubernetes version the same but may run different versions of core components, and 5 (4.1%) have different core components across environments.

8 (6.6%) respondents have replied that they run different versions of Kubernetes across environments, and one person has struggled to answer.

## Disaster Recovery

{{< figure src="/img/posts/k8s-survey-2023/16_have-dr-plan.png" >}}

Only 18 (14.8%) of the responders said they have a disaster recovery plan and it's regularly tested. 29 (23.8%) - said that they have the plan but only tested it once, and 41 (33.6%) said that they have a plan in place but actually never tested it. 34 (27.9%) of the respondents said they have no disaster recovery plan.

{{< figure src="/img/posts/k8s-survey-2023/17_have-backup.png" >}}

Yet, 47 (38.5%) respondents said they could restore their clusters using the GitOps approach. Therefore, they do not take cluster backups. 24 (19.7%) people use backups tools such as [Velero](https://github.com/vmware-tanzu/velero) to back up their clusters, and 15 (12.3%) take snapshots of ETCD. 36 (29.5%) respondents said they do not have backups for their clusters.

## The Fun Part

In the last year, I have asked people to share their most painful experiences with Kubernetes.

Interestingly, while some people point out the struggles of managing Kubernetes on their own, others say that non-transparent and sometimes unpredictable updates of the managed Kubernetes services were the most painful experience for them. I assume this is because of the division between "self-managed" vs. "cloud-managed" clusters. The bottom line here is that Kubernetes is a complex system. Cloud providers can, of course, remove some complexity but not all.

Yet, many people have said their struggles come from running stateful services in Kubernetes. Sure, there are more and more solutions to run stateful things in Kubernetes each year, but apparently, it's still quite challenging.

You can find all the replies [in the Google Sheets](https://docs.google.com/spreadsheets/d/1rR8bcKi1NnQpbnOA9zVK6IhV89hTr4aJdi8O3Rcb-mY/edit?usp=sharing).

## What's next

I've got some feedback about this survey this year. Some important points:

- To include some questions about Observability next year. This is an excellent point. I'll do that in the next year's survey.
- To have a more precise goal for this survey. I thought I had one, but I will communicate it better.

Thanks for reading! Glory to Ukraine!
