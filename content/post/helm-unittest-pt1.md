---
title: "Testing Helm Charts Part I"
date: 2024-07-11
draft: false
slug: "helm-testing-pt1"
cover:
  image: "/img/posts/helm-testing/cover-pt1-ai.webp"
categories: ["Tech"]
tags: ["kubernetes", "helm", "testing", "en"]
---

Before answering this question, we should decide _why to test Helm chart?_ and if you even need to bother with that. Following an example from this _xkcd_ comic, the real answer is: _it depends_.

![xkcd automation](https://imgs.xkcd.com/comics/automation.png)

So, I want to share with you my story of Helm chart testing. How and, more importantly, why it started, and how it played out in the end.

On a side note, I did a talk on this topic at the DevOps FW Days conference. This talk is Ukrainian, so if you know the language, you can watch [it on YouTube]([https://youtu.be/BtciJTqJvQ8?si=gKkGh_EGBCbc-48V](https://youtu.be/BtciJTqJvQ8?si=gKkGh_EGBCbc-48V)). The slides are in English, though, and [are available here]([https://docs.google.com/presentation/d/13HNGHYJt1rcxL7ZTHvshplzy8H9WPlm3700qvcFKOF8/edit?usp=sharing](https://docs.google.com/presentation/d/13HNGHYJt1rcxL7ZTHvshplzy8H9WPlm3700qvcFKOF8/edit?usp=sharing)).

## The beginning of the journey (why)

We had a pretty standard setup: a couple of Kubernetes clusters, a couple of hundreds of services and a couple of Helm charts to manage all this stuff. To be precise, we had about a dozen of charts to manage all the Kubernetes plugins, mostly the official ones. For business apps we had only three charts.

The code was organized in a way so platform teams could hide the complexity under the hood and provide the product teams an API in a form of `values.yaml` files.

This approach helped us to migrate a lot of services into Kubernetes. It also helped us to quickly onboard any new services.

On a flip side, it also meant that any change to any of our three "one size fits all" charts had a tremendous blast radius. Thus, a logical idea was to introduce some testing for our Helm charts.

YMMV when it comes to the Helm charts testing, though. I would say, it makes sense to introduce at least some testing if:

- You update your Helm charts relatively often.

- Each update impacts many teams/people or may have dire consequences if done wrong.

- Your charts contain custom logic.

- Your customers control the input i.e. they can change the values they provide without you even knowing. This also applies if your charts are public.

If this sounds familiar to you, you may find this article helpful.

## The tragedy of unit tests

Helm has an embedded `test` command, but what it actually does is it creates dummy objects in a Kubernetes cluster and runs some assertions. This approach is totally fine, but it has two major flaws that have driven us away from it:

- It's not a unit test if you have to have a cluster to test something

- What are we even testing when we spin up a random Nginx container and getting the default welcome page out of it?

The first problem may seem like a minor obstacle, especially given how easy it is to spin up a Kubernetes cluster these days with tools like [Kind]([https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/)), [k3s]([https://k3s.io/](https://k3s.io/)), or [Minikube]([https://minikube.sigs.k8s.io/docs/](https://minikube.sigs.k8s.io/docs/)). However, the majority of clusters do not exist in isolation. You likely have a bunch of controllers and integrations in place. These would be quite hard to replicate in an isolated test environment. Moreover, setting up a fresh environment for test takes time and computing resources. These resources may be better used elsewhere.

The second thing is that we don't deploy a real app in a test scenario. So, it gives us way too few insights into our code itself for the price of having a test cluster(s).

## Validating the template

_“In the most cases, configuration management can be tested using simple static code analysis”

Jeff Smith @ DevOps Days Chicago_

At the end of the day, Helm is a templating tool. So, we could template our chart with various values and check that the resulting manifests looks correct. First, we tried to leverage Terratest for this purpose. First, we tried Terratest for this purpose.

Some blog posts that are easy to find, likde [this one]([https://blog.gruntwork.io/automated-testing-for-kubernetes-and-helm-charts-using-terratest-a4ddc4e67344](https://blog.gruntwork.io/automated-testing-for-kubernetes-and-helm-charts-using-terratest-a4ddc4e67344)) or [this one]([https://medium.com/@zelldon91/advanced-test-practices-for-helm-charts-587caeeb4cb](https://medium.com/@zelldon91/advanced-test-practices-for-helm-charts-587caeeb4cb)) as well as [code examples]([https://github.com/gruntwork-io/terratest-helm-testing-example](https://github.com/gruntwork-io/terratest-helm-testing-example)) focus on the integration testing. Still, we can use Terratest to render a template and compare it with a "golden" manifest that we have at hand.

The good thing about Terratest is that it is available as a Go package. Thus, one can leverage the whole power of a general-purpose programming language. It is also very flexible thanks to the real language under the hood.

This setup worked well for us for quite some time, and I know people for whom it still works well. But, with the time, some drawbacks became visible:

- Not everybody is comfortable writing code in Go. This results in inconsistent test quality. This may be not an issue in small teams, but something to consider in bigger platform departments.

- Code duplication. For each scenario, we had to provide some test values, render a chart, and compare the results with predefined manifests. Eventually, we extracted this logic into a separate package. But, it led to the another drawback.

- Maintainability. Someone has to step in and maintain the test package. This adds another layer of complexity.

- The number of false positives. This is the most critical one. Our charts had some autogenerated data. For example, some annotations were generated from the chart version. Thus, such data had to be excluded from the evaluation. Otherwise one had to update the "golden files" on every change. At some point, it became too cumbersome to track those fields. People began to regenerate "golden" files each time the tests failed. Needless to say that such an approach undermines the whole idea of testing.

Also, so-called "golden files" are quite verbose. Those are complete YAML manifests and if you work with Kubernetes extensively, you know that those could be quite large. Because of this, it's hard for a human to fully prepare and update those files by hand.

Luckily, it's easy to generate of these manifests. However, this could lead to accidental errors sneaking into golden files when they are regenerated.

## Testing only what matters

I mentioned at the beginning that it makes sense to test Helm charts when you have some custom logic inside. In other words, when you have _what to test_. So, let's test only the parts that matter!

We came up with several functional and non-functional requirements for the tests:

- A chart can be rendered at all

- Charts themselves are following good practices

- Resulting manifests are “correct”

- Resulting manifests follow good practices (including security)

- Any logic inside charts (conditionals, includes, etc.)

- Tests run in reasonable amount of time

- Tests are simple to maintain

- Tests are reproducible

### A chart can be rendered

This is a simple one: you can use the `helm template` command as well as the [Helm JSON Schema]([https://www.arthurkoziel.com/validate-helm-chart-values-with-json-schemas/](https://www.arthurkoziel.com/validate-helm-chart-values-with-json-schemas/)) to guarantee this. The only caveat here is that you need to provide some values to the chart. We overcame this by creating some default test values for each chart that covered the basic usage of a chart.

### Charts are following good practices

Helm has a `helm lint` command that evaluates if you templates themselves are correct. I cannot say that this check brought us a lot of insights and helped to catch many errors, but if the functionality is there, then why not to use it?

### The manifests are correct

This is sorta first line of defense when it comes to YAML manifests. We used [Kubeconform]([https://github.com/yannh/kubeconform](https://github.com/yannh/kubeconform)) to check if the schema and API versions are correct an up-to-date with the Kubernetes version we've been using.

The good thing about Kubeconform is that it supports custom schemes. You can also validate custom resources. It requires a little bit of work to keep those schemes up-to-date, since the tool won't track CRD changes for you.

### The manifests are secure

It is possible to use a generic tool to check security as well. We kept it separate, because we had a dedicated security team. By separating security checks, we separate the tests' ownership within the Platform department.

These policies were then validated with [Kyverno]([https://kyverno.io/](https://kyverno.io/)).

### Testing the logic

At last, we got to the most exciting part! In theory, we could still use Terratest. But we would need to maintain the complex test logic ourselves. Luckily, there's a project called [Helm Unittest]([https://github.com/helm-unittest/helm-unittest](https://github.com/helm-unittest/helm-unittest)), which is perfect for our scenario.

With Helm Unittest it is possible to validate specific chunks of rendered manifest. So, unimportant parts won't stand in your way. Also, test definitions are much smaller compared to golden files. Hence, they are easier for a human to read and comprehend.

This article is already too long to showcase Helm Unittest in action. So, I would extract it into a separate blog post. Still, [the code]([https://github.com/grem11n/talk-props/tree/main/fw-days-devops-2024](https://github.com/grem11n/talk-props/tree/main/fw-days-devops-2024)) I used for the conference demo is available. You can take a look at it and give Helm Unittest a try already.

## The test pyramid

**test pyramic pic**

As you could already tell, some of those tools are redundant. For example, Helm Unittest runs `helm template` under the hood. So, there's no need in executing that command separately. In the same way, you can write tests to ensure the security practices without Kyverno.

However, such "testing pyramid" allowed us to separate our configuration tests into several layers:

 - Mandatory baseline: templating and linting
 - Security baseline: Kyverno
 - Optional tests of the business logic: Unittest

Mandatory baseline tests run for each chart and every PR, ensuring that our code is at least deployable. The security baseline could be "outsourced" to the security team, so we don't need to keep the tests for each chart up-to-date with the policy updates.

Last but not least, by making the business logic tests optional, we provided our colleagues a powerful tool without forcing them to write meaningless tests. Again, if your chart doesn't have any loops or conditionals, there's probably not that much value in the tests.
## Are these test enough?

Now, to the serious questions. Is it enough to have only local unit test to ensure that you Helm charts work as expected. Unfortunately, not.

Helm Unittest is a young project and it has its flaws. For example, it doesn't work well with nested lists. In such cases, we were falling back to "golden files". Luckily, Helm Unittest allows golden testing as well. So, we could leverage the same tool for various scenarios.

Although, our "test pyramid" has helped us to catch errors early on, it still doesn't guarantee that things would run smoothly in a real cluster. Also, it doesn't guarantee that the cluster itself works fine. We had other lines of defense, such as a dummy app that was mimicking a real application in a cluster. We also had E2E and stress tests for Kubernetes clusters themselves. Yet, this is a story for another time.

## Conclusion

As it always happens in tech, unit testing of Helm charts has its trade-offs. Although, this testing your code is a good practice, you may not get the most of it.

Still, if you feel like it's time to make your charts more robust, I strongly suggest you to start testing your Helm charts. Your customers would be grateful!
