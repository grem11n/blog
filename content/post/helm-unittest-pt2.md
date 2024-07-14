---
title: "Testing Helm Charts Part II"
date: 2024-07-13
draft: false
slug: "helm-testing-pt2"
cover:
  image: "/img/posts/helm-testing/cover-pt2-ai.webp"
categories: ["Tech"]
tags: ["kubernetes", "helm", "testing", "en"]
---

_This article is also available [on Substack](https://newsletter.catops.dev/p/testing-helm-charts-part-ii).

This is a very basic example of using Helm Unittestas well as an example of the "test pyramid" discussed in the [previous article](https://grem1.in/post/helm-testing-pt1/). The code is [available on GitHub](https://github.com/grem11n/talk-props/tree/main/fw-days-devops-2024).

## Structure

We have two charts:

- `fw-demo` - a chart with our demo application

- `monitiring` - a library chart that applies some DataDog monitoring rules for the main chart.

`monitoring` chart is a dependency. The main `Chart.yaml` looks like:

```yaml

---

apiVersion: v2

name: fw-demo

description: A demo chart for FW Days DevOps 2024

type: application

version: 0.0.1

appVersion: "0.0.0"

dependencies:

  - name: monitoring

    version: 0.0.1

    repository: "file://../monitoring/"

```

## Basic tests

For basic tests, we want to ensure that the main chart is "templatable" and that `helm lint` passes. The main chart requires some values. For the tests, we can define them in the `fw-demo/tests/minimal.values.yaml`. This file contains the minimal set of required values. We can check if `helm template` passes as well as it hints our users of the minimal required configuration:

```yaml

---

owner: test

clusterName: test-cluster

```

Now, we can run the `template` and `lint` commands. We don't care about the actual output at this stage, so it can be omitted.

```bash

helm template fw-demo --values fw-demo/tests/minimal.values.yaml 1>/dev/null

echo $?

0

```

```bash

helm lint fw-demo --values fw-demo/tests/minimal.values.yaml

==> Linting fw-demo

[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

echo $?

0

```

## Conformance tests

To run the conformance tests, we are going to use [Kubeconform](https://github.com/yannh/kubeconform). However, it only works with rendered manifests. So, let's pipe the `helm template` output into it:

```bash

helm template fw-demo --values fw-demo/tests/minimal.values.yaml | \

kubeconform --summary -ignore-missing-schemas -kubernetes-version 1.30.1

Summary: 5 resources found parsing stdin - Valid: 4, Invalid: 0, Errors: 0, Skipped: 1

```

- `--summary` outputs some brief information about the validated objects

- `-ignore-missing-schemas` instructs Kubeconform to ignore custom resources. We don't have custom schemas for them in this test, although it's possible to add them

- `-kubernetes-version` allows you to set the cluster version for API versions check

## Policy checks

So far our test pyramid looks good. There are some basic checks for the chart, and validations of the rendered manifests. Now, let's imagine that we have a set of policies provided by the Security team that our chart must adhere to. Thus, in this example, this policy resides in a separate folder. In your case, this could be even a separate repository.

We will use [Kyverno](https://kyverno.io/)for this example, but [OPA/Gatekeeper](https://github.com/open-policy-agent/gatekeeper) would do as well. I choose Kyverno for this demo because it's designed for Kubernetes and it's YAML-based. It makes its policies easier to comprehend compared to OPA's Rego.

For the illustration purposes, I took a policy from [the public library](https://kyverno.io/policies/best-practices/require-labels/require-labels/). This policy ensures that a label `app.kubernetes.io/owner` is set. Since we do have this label set in `_helpers.tpl`, we should be able to run the test successfully. Notice, that you need to template the chart first, because Kyverno only works with rendered manifests.

```helm

{{/*

Common labels

*/}}

{{- define "fw-demo.labels" -}}

helm.sh/chart: {{ include "fw-demo.chart" . }}

{{/* We want to hint the ownership here! */}}

app.kubernetes.io/owner: {{ required "Owner .Values.owner is required." .Values.owner }}

{{ include "fw-demo.selectorLabels" . }}

{{- if .Chart.AppVersion }}

app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}

{{- end }}

app.kubernetes.io/managed-by: {{ .Release.Service }}

{{- end }}

```

```bash

helm template fw-demo --values fw-demo/tests/minimal.values.yaml --namespace fw-demo --kube-version 1.30.0 | \

kyverno apply policies -r -

Applying 3 policy rule(s) to 5 resource(s)...

pass: 2, fail: 0, warn: 0, error: 0, skip: 0

```

Let's see what happens if we remove this label from our manifests.

```bash

sed -i '/app.kubernetes.io\/owner/d' fw-demo/templates/_helpers.tpl

helm template fw-demo --values fw-demo/tests/minimal.values.yaml --namespace fw-demo --kube-version 1.30.0 | \

./kyverno apply policies --v 3 -r -

Applying 3 policy rule(s) to 5 resource(s)...

2024-07-11T21:00:23+02:00       LEVEL(-3)       engine.validate validation/validate_resource.go:329       validation error        {"policy.name": "require-labels", "policy.namespace": "", "policy.apply": "All", "new.kind": "Deployment", "new.namespace": "default", "new.name": "release-name-fw-demo", "rule.name": "autogen-check-for-labels", "path": "/spec/template/metadata/labels/app.kubernetes.io/owner/", "error": "resource value '<nil>' does not match '?*' at path /spec/template/metadata/labels/app.kubernetes.io/owner/"}

2024-07-11T21:00:23+02:00       LEVEL(-3)       engine.validate validation/validate_resource.go:329       validation error        {"policy.name": "require-labels", "policy.namespace": "", "policy.apply": "All", "new.kind": "Pod", "new.namespace": "default", "new.name": "release-name-fw-demo-test-connection", "rule.name": "check-for-labels", "path": "/metadata/labels/app.kubernetes.io/owner/", "error": "resource value '<nil>' does not match '?*' at path /metadata/labels/app.kubernetes.io/owner/"}

pass: 0, fail: 2, warn: 0, error: 0, skip: 0

```

I use more verbose output with `--v 3` to show you the actual validation error.

Now, the baseline is complete, and it's time to move to the most exciting part - actual unit tests! Before we jump there, though, I want to mention a tool that was brought to my attention after I gave this talk. It is called [Chart testing](https://github.com/helm/chart-testing)or `ct`. It's a CLI that allows you to run linter and YAML validations. Yet, I haven't tried it, so I don't know how useful it is compared to bare Helm commands. With this been said, let's jump to the most interesting part!

## Unit Test

[Helm Unittest](https://github.com/helm-unittest/helm-unittest) allows you to declare your test cases in plain YAML. I understand that no one has asked for more YAML, yet here we are. One thing that I like the most about it is that it allows one to validate specific parts of the resulting manifest. Places that contain logic: conditions, loops, etc. This is much easier than messing around with "golden templates". However, Helm Unittest supports snapshot testing as well.

There are two conditional parts in the demo chart:

1. Applies an HPA resource if `.Values.autoscaling.enabled` is `true`

2. Applies a DataDog Monitor custom resource, if `.Values.monitoring.enabled` is `true`

The test code resides in the [`tests/`](https://github.com/grem11n/talk-props/tree/main/fw-days-devops-2024/fw-demo/tests) directory of our chart. This is a requirement of Unittest. I won't copy-paste the test code here, because it is too verbose. Here are scenarios we want to test:

1. Autoscaling is disabled, thus an HPA resource is absent and the number of replicas is set to a fixed value.

2. Autoscaling is enabled, this an HPA object is created. The number of replicas is not specified in the Deplyment object by default

3. Monitoring is disabled, thus DataDog custom resource is not present

4. Monitoring is enabled, thus the custom resource is there, and it has specific `apiVersion` and `kind`.

The configuration is pretty straightforward, but Unittest supports more complex scenarios. The only thing I found cumbersome is to work with nested lists. In such cases, we used to fall back to snapshot testing, which is also possible with this tool.

Here's an example of a successful test run:

```bash

untt fw-demo

### Chart [ fw-demo ] fw-demo

 PASS  fw-demo  fw-demo/tests/deployment_test.yaml

 PASS  fw-demo  fw-demo/tests/monitoring_test.yaml

Charts:      1 passed, 1 total

Test Suites: 2 passed, 2 total

Tests:       6 passed, 6 total

Snapshot:    0 passed, 0 total

Time:        13.164702ms

```

## Bringing it all together

It's straightforward to bring all these pieces together under a single CI job that would validate your Helm charts before they hit production. I even automated it all in a [`Makefile`](https://github.com/grem11n/talk-props/blob/main/fw-days-devops-2024/Makefile).

Despite I execute all the commands using local binaries, it is also possible to run all these things in Docker for greater portability. This is actually how we used to run this test suite in the real world scenario. I decided to drop the Docker part for the demo, because it is not that relevant.

I hope you enjoyed this little read, and I hope you could take away some ideas on how to make your Helm charts more bulletproof!
