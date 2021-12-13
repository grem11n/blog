---
title: "Two days with Pulumi"
description: "The first IaC tool in DevOps, which came from the Dev side"
date: 2020-01-31
draft: false
slug: "pulumi"
cover:
  image: "https://www.netclipart.com/pp/m/298-2984485_pulumi-logo-transparent.png"
  alt: "pulumi logo"
categories: ["Tech"]
tags: ["pulumi", "iac", "eng"]
---

{{< figure src="https://www.pulumi.com/social/pulumi.png" class="center" >}}

[Pulumi](https://www.pulumi.com/) is just another Infrastructure as Code tool, yet it's different from what I've seen before. It offers some interesting concepts like usage of the generic purpose programming languages, state tracking, etc. I had some time recently so I decided to play around with it. Here I want to share some thoughts about this tool.

In my opinion this is the first IaC tool, which was created for the "Dev" side of "DevOps". With the raise of DevOps movement, both software and system engineers learned a lot from each other. Infrastructure as Code itself is a SWE approach. However, infrastructure usually falls into the Ops people responsibilities. Therefore, IaC tools as well as configuration management tools were built to be convenient for sysadmins. We had several different DSLs and tons of YAML. Pulumi is different. It doesn't re-invent ways to declare your infrastructure with code.

Pulumi allows you to define the infrastructure using the general purpose coding language. Currently, `TypeScript`, `JavaScript`, and `Python` are supported in a stable mode, `Go` and `C#` have experimental support. I also talked to them on the DevOps Days Ghent 2019 event and they said that `Java` support is coming.

First of all, syntax. Since you're using a general purpose language, you don't need to learn any new DSL. If you're already using one of the supported. It's more like learning a new framework.

Let's take a look at a basic example.

```js
import * as aws from "@pulumi/aws";

let group = new aws.ec2.SecurityGroup("web-sg", {
    description: "Enable HTTP access",
    ingress: [{ protocol: "tcp", fromPort: 80, toPort: 80, cidrBlocks: ["0.0.0.0/0"] }],
});

let server = new aws.ec2.Instance("web-server", {
    ami: "ami-6869aa05",
    instanceType: "t2.micro",
    securityGroups: [ group.name ], // reference the security group resource above
});
```

This is one of the examples from the official website. Basically, here we create a security group and attach it to a new instance. Looks pretty declarative apart of that the code is written in TypeScript. However, let's take a look at more interesting example:

```js
/**
We need to import Pulumi providers (I'm not sure if they call it provides, tho)
*/
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

/**
This function is capable to create Route53 records in different configurations:
1. Simple CNAME
2. Record with FailoverPolicy. See: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html#routing-policy-failover

It assumes 4 arguments:
* name - the name of the resource - required
* records - target for CNAME - required
* zoneId - Route53 zone ID - required
* failoverPrecedence - PRIMARY|SECONDARY failover option - optional

Function returns a Route53 Record. However, based on the failoverPrecedence argument it can also create a Route53 HealthCheck and CloudWatch Metric Alarm.
*/

export function createRecord(
    name: string,
    records: pulumi.Input<any>,
    zoneId: pulumi.Input<any>,
    failoverPrecedence?: string): aws.route53.Record {
        // Here I set default r53 record options
        var rawOptions = {
            name: name,
            records: records,
                        type: "CNAME",
            ttl: 300,
            zoneId: zoneId,
        }
        var options = rawOptions
        if (failoverPrecedence != null && failoverPrecedence.toUpperCase() === "PRIMARY") {
            // Create CloudWatch Metric Alarm if it's a PRIMARY R53 record
            const cwAlarm = new aws.cloudwatch.MetricAlarm("primary", {
                alarmDescription: "Monitor healthy node count",
                metricName: "HealthyHostCount",
                comparisonOperator: "LessThanThreshold",
                evaluationPeriods: 1,
                namespace: "AWS/ELB",
                period: 60,
                statistic: "Average",
                threshold: 1,
            });
            // Create Route53 HealthCheck if it's a PRIMARY R53 record
            const r53HealthCheck = new aws.route53.HealthCheck("primary", {
                cloudwatchAlarmName: cwAlarm.name,
                type: "CLOUDWATCH_METRIC",
                cloudwatchAlarmRegion: "eu-central-1",
                insufficientDataHealthStatus: "Unhealthy",
            });

            // Add non-default options
            let addOptions = {
                setIdentifier: "001",
                healthCheckId: r53HealthCheck.id,
                failoverRoutingPolicies: [{
                    type: failoverPrecedence.toUpperCase()
                }],
            }

            // Merge default and non-default options
            const options = {...rawOptions, ...addOptions};

        } else if (failoverPrecedence != null && failoverPrecedence.toUpperCase() === "SECONDARY") {
            // Non-default options for SECONDARY Record
            let addOptions = {
                setIdentifier: "002",
                failoverRoutingPolicies: [{
                    type: failoverPrecedence.toUpperCase()
                }],
                }

            // Merge default and non-default options
            const options = {...rawOptions, ...addOptions};
        } else if (failoverPrecedence != null) {
            throw new Error('failoverPrecedence must be either PRIMARY or SECONDARY!');
        }
                return new aws.route53.Record(
            name, options
        );
}
```

In this case we have a function, which creates either simple Route53 record or a record with Primary-Secondary failover policy.

Later, we can call this function in the `index.ts` like this:

```js
// Declare Route53 records
// `export` provides all the output into stdin. Helpful for debug
export const host001R53 = createRecord(`host-001`, [host001Elb.dnsName.apply(d => d)], r53ZoneId)
export const host002R53 = createRecord(`host-002`, [host002Elb.dnsName.apply(d => d)], r53ZoneId)
export const haHost001R53 = createRecord(`ha-host`, [host001Elb.dnsName.apply(d => d)], r53ZoneId, "PRIMARY")
export const haHost002R53 = createRecord(`ha-host`, [host002Elb.dnsName.apply(d => d)], r53ZoneId, "SECONDARY")
```

Notice, that even though both `haHost001R53` and `haHost002R53` have the same endpoint, variables have different names inside the Pulumi code.

Same way you can use Classes to unify resources. [Here is a pretty good example for this](https://github.com/pulumi/examples/blob/fdf37ddecedb10dcdfa77362f8ece49c52011caa/aws-stackreference-architecture/database/src/database.ts#L55) and [here is an actual resource creation](https://github.com/pulumi/examples/blob/fdf37ddecedb10dcdfa77362f8ece49c52011caa/aws-stackreference-architecture/database/src/index.ts#L23).

And here is, probably, the main difference between Pulumi and everything I've seen before. With Pulumi your infrastructure stack (naming is similar to CloudFormation) is essentially a separate application.

## Usage

Running Pulumi is is simple as well. However, they try to login your dy default into their Cloud, which is pretty confusing for a CLI tool:

1. Do `npm install` in the `pulumi/` directroy.
2. "Login" into Pulumi: `pulumi login file://.` (probably not required when a state is already there)
3. Login into your cloud account
4. Do `pulumi up` to see the changes

Like Terraform, Pulumi will provide you a plan first. So, you'll be able to see actual changes in your infrastructure before applying anything. At the same time, like with Terraform, plan doesn't guarantee that an actual apply is going to be successful.

## Main differences from Terraform (subjectively)

With Terraform you have a DSL (HCL) in which you define resournces and relation between them. While with Pulumi your stack (think of Terraform state) is a separate application.

### What I liked

* Flexibility. Using a general purpose language allows you to do whatever you want basically without DSL limitations. Loops, conditions, inheritance, etc.
* Multiple languages support. So, in theory you don't need to learn a new language if you're already using one of the supported. It's more like learning a new framework.
* Pulumi community has really nice [Slack group](https://slack.pulumi.com/) and people are helpful there :)

### What was challenging

* Since I'm not good at programming, for me Pulumi was more like learning a new framework AND a new language :)
* The way you're doing things might be confusing if you're used to the declarative Terraform syntax
* Documentation and examples. Now I understand how good are the HashiCorp's docs! Pulumi have both, but when it comes to the real-world examples, things got tricky.

### What I didn't like

* By default Pulumi will ask you to login to their platform. It's possible to use local state (which I did), also it's possible to configure remote storage for the state. [Here is the documentation](https://www.pulumi.com/docs/intro/concepts/state/).
* Missing documentation. For example, documentation for Go is missing completely, though it's supported.

## Random observations

* You can [read outputs from Terraform states](https://www.pulumi.com/blog/using-terraform-remote-state-with-pulumi/), which is pretty neat. Especially, if you already have some resournces in Terraform.
* A lot of providers are derived from Terraform providers :)

{{< figure src="https://i.imgflip.com/3nrt0s.jpg" class="center" >}}

* Names of the resournces contain random strings by default (it can be changed). This was confusing, when I created Route53 record for the first time.

* Sometimes it's impossible to make a type check when you're passing outputs of one resource as inputs for another resource. Especially, if you're getting those values from Terraform state because Terraform outputs [have `<any>` type](https://www.pulumi.com/docs/reference/pkg/nodejs/pulumi/terraform/state/#RemoteStateReference-getOutput).

{{< figure src="/img/posts/pulumi/pulumi_slack.png" class="center" >}}

## Final thoughts

This tool could be cool for the people with development background to start working with the infrastructure. With a size of our existing infra, I don't think it's reasonable to migrate it to Pulumi, especially taking into account the learning curve.

It may be a way to go for the greenfield projects, which are run by development teams, for example, configuring Kubernetes stacks. Also, it could be pretty cool for small teams and startups, which don't have dedicated OPS engineers. 

However, I'm not sure how to keep it consistent (several languages are supported and each of them allow flexibility on it's own). Though, keeping consistency is possible (like in the example with custom classes).

I don't think, I get back to Pulumi any time soon. Mainly because we are heavily using Terraform. Though, I like appreciate new competition in IaC area. I really think that Pulumi is an interesting project and it might be helpful for a lot people out here.

## Links

* [Feedback about Pulumi by Patrick Debois](https://gist.github.com/jedi4ever/816cf6642d989b70d4fec8fa6e1298bf)
* [Pulumi docs on GitHub](https://github.com/pulumi/docs)
* [tf2pulumi - tool to convert Terraform projects into Pulumi TypeScript code](https://github.com/pulumi/tf2pulumi)
* [Using Terraform Remote State with Pulumi](https://www.pulumi.com/blog/using-terraform-remote-state-with-pulumi/)
* [How Pulumi Compares to Terraform for Infrastructure as Code by Kyle Galbraith](https://dev.to/kylegalbraith/how-pulumi-compares-to-terraform-for-infrastructure-as-code-434j)
* [Pulumi examples](https://github.com/pulumi/examples)
* [Pulumi State Backends](https://www.pulumi.com/docs/intro/concepts/state/)
* [Pulumi Slack Community](https://slack.pulumi.com/)
