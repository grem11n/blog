---
title: "Crossplane"
description: "I briefly went through the installation and configuration process and also shared my thoughts about Crossplane"
date: 2021-02-01
draft: false
slug: "crossplane"
image: "/img/posts/crossplane/crossplane.png"
categories: ["Tech"]
tags: ["kubernetes", "k8s", "iac", "crossplane"]
---

## What's Crossplane?

[Crossplane](https://crossplane.io/) is an Open Source tool, which allows you to manage Cloud infrastructure as Kubernetes objects. In other words, you can create, modify, and delete AWS cloud assets using only Kubernetes manifests in the same way as with Terraform (or another IaC tool). It also allows you to manage cloud resources in different public clouds using a concept of providers very similarly to Terraform. Therefore, Crossplane allows engineers to manage the whole application lifecycle from a single entry point e.g. a Helm chart.

{{< figure src="https://crossplane.io/docs/v0.2/media/arch.png" alt="crossplane architecture diagram" caption="source: https://crossplane.io/docs/v0.2/" >}}

However, it might be hard to define, which resources belong to a particular application since you can have shared services. You can of course define your entire infrastructure as Crossplane objects and keep it separate. Although, in this case, you will miss the whole point of this project, in my opinion.

In this article, I briefly went through the installation and configuration process and also shared my thoughts about Crossplane. Feel free to skip to the Final Thoughts section if you're only interested in the latter.

**Important Disclaimer**: I've played with Crossplane version `v0.13`. The current stable version is `v1.0`. Some things might have changed since then.

## Installation

### Install Crossplane itself

Crossplane has some [good official installation guide](https://crossplane.io/docs/v1.0/reference/install.html). It's distributed as a Helm chart, so you can install it into your cluster with just a few commands. Crossplane requires Kubernetes version >= 1.16.0 and Helm >= 3.0.0

The latest stable version is `v1.0.0`, so you can install it with this command:

```bash
kubectl create namespace crossplane-system
helm repo add crossplane-master https://charts.crossplane.io/master/
helm search repo crossplane-master --devel

helm install crossplane --namespace crossplane-system crossplane-master/crossplane --devel --version v1.0.0
```

As you can see, the official documentation suggests installing Crossplane into a separate namespace. It doesn't necessarily, but having a separate namespace for it can save you some time if you need to debug something. Also, make sure to check the [documentation](https://crossplane.io/docs/v1.0/reference/install.html) for installation options.

### Install a public cloud provider

Next, you will need to install a provider for your public cloud. You can have multiple if you are running a multi-cloud as well.

First of all, you will need to configure permissions for a Crossplane provider. And here we have the first chicken-egg problem. Presumably, you want to manage all of the cloud resources in code including the permissions, but you cannot use a tool without the permissions. To be honest, this problem is actual for any IaC tool, which has a centralized runtime, which in this case is a Kubernetes cluster. However, this problem is easy to fix if you already have something like Terraform or [Pulumi](https://grem1.in/post/pulumi/).

With this been said, let's take a look at the provider configuration. I'll use AWS as an example. Crossplane provides a semi-automated way to configure a provider (you still need to configure IAM), but a manual configuration provides a few interesting insights. You can find it in the [official doc](https://crossplane.io/docs/v1.0/cloud-providers/aws/aws-provider.html) as well:

```yaml
cat > provider.yaml <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-account-creds
  namespace: crossplane-system
type: Opaque
data:
  credentials: ${BASE64ENCODED_AWS_ACCOUNT_CREDS}
---
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-account-creds
      key: credentials
EOF

# apply it to the cluster:
kubectl apply -f "provider.yaml"

# delete the credentials variable
unset BASE64ENCODED_AWS_ACCOUNT_CREDS
```

Where `BASE64ENCODED_AWS_ACCOUNT_CREDS`is an environment variable, which contains your AWS credentials in the same syntax as `~/.aws/credentials`:

```bash
BASE64ENCODED_AWS_ACCOUNT_CREDS=$(echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $aws_profile)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $aws_profile)" | base64  | tr -d "\n")
```

And this is what's wrong here. Crossplane requires broad IAM permissions. Of course, you may use it only for a limited subset of cloud resources, but you'd better keep these keys secure either way. And Kubernetes Secrets is not secure storage, because [Base64 is not encryption](https://archive.org/details/youtube-f4Ru6CPG1z4). Therefore, I would strongly advise using different secret storage like HashiCorp Vault or similar. Or even better! Use [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html), which is not only supported in EKS now or tools like [kiam](https://github.com/uswitch/kiam) or similar.

Depending on how you authenticate against AWS API, you need to modify the provider configuration. Basically, Crossplane AWS Provider consists of four resources.

- Deployment
- ServiceAccount
- ClusterRole
- ClusterRoleBinding

You only need to modify the Deployment and add required annotations there, secret manager path references, etc. I recommend you create a custom configuration for it and just apply it all together. Perhaps, it will be possible to use something different from Kubernetes Secrets in Crossplane in the future, will see.

If you have installed Crossplane in a separate namespace, you should be able to validate installation with:

```bash
kubectl get all
```

Also, you can validate the state of the provider with:

```bash
kubectl get providers.pkg.crossplane.io
```

## Provisioning Infrastructure

Now you are ready to create your first infrastructure resources with Crossplane! You can find all the available resources in[ the API documentation](https://doc.crds.dev/github.com/crossplane/provider-aws) for your provider. In this example, I'm going to create a simple IAM policy:

```yaml
cat <<EOF | kubectl apply -f -
---
apiVersion: identity.aws.crossplane.io/v1alpha1
kind: IAMPolicy
metadata:
  name: test-policy-crossplane
  annotations:
    crossplane.io/external-name: "arn:aws:iam::538639307912:policy/test-crossplane"
spec:
  deletionPolicy: Orphan
  forProvider:
    description: "Test policy"
    name: test-crossplane
    document: |
      {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Sid": "VisualEditor0",
                  "Effect": "Allow",
                  "Action": "s3:ListBucket",
                  "Resource": "*"
              }
          ]
      }
EOF
```

[Crossplane API reference for AWS IAM policy](https://doc.crds.dev/github.com/crossplane/provider-aws/identity.aws.crossplane.io/IAMPolicy/v1alpha1)

And this should be it. This code should create an IAM policy, which allows to List S3 buckets. Also, since all this stuff is managed as Kubernetes API objects, you can template definitions using tools like Helm and so on.

If a resource is not reconciled in AWS, you can check logs of the AWS Provider pod and if there are no errors, describe the resource itself. For example:

```bash
kubectl describe iampolicy <POLICY_NAME>
```

## Where the State Goes?

Just like Terraform Crossplane tracks the state of your resources as custom Kubernetes resources and can reconcile them if necessary. So, the state of your infra is stored in the ETCD of your cluster. Which might be a bit worrisome. Especially, if you treat your clusters like cattle, spin up new and bring down old clusters regularly. It would be also a concern if one day you want to make a ship-and-lift migration to the new cluster.

Fortunately, Crossplane has handles to mitigate that. These handles are also valuable for importing existing infrastructure.

### Importing Existing Infrastructure

You can provide a special annotation `crossplane.io/external-name`, which points to the existing resource. This annotation tells Crossplane to check AWS for existing resources before trying to create a new resource. If the policy from the previous example already existed, execution would just fail. However, with the `external-name` set, Crossplane will checks AWS makes an import.

A fun fact: I only tried to play with IAM policies and it turned out that IAM policy is a "special resource" and you need to provide full ARN as `external-name`to import it. Check out [this GitHub issue](https://github.com/crossplane/provider-aws/issues/323) for reference.

## Keep Cloud Resource When CR is Deleted

There is another configuration option, which you should put to successfully "migrate" to another cluster. `deletionPolicy: Orphan` tells Crossplane that it shouldn't delete a cloud resource if the Kubernetes Custom Resource was deleted. If you want to delete a resource on CR deletion, you need to set `deletionPolicy: Delete`, which is the default behavior.

With these two configuration options, you can re-import cloud resources created with Crossplane to another cluster using only Crossplane itself. In theory, you can also try to migrate both `spec` and `status` of Crossplane Custom Resources with tools like [Velero](https://github.com/vmware-tanzu/velero), but I haven't tried that.

Also, I tried to migrate only IAM policies, which are pretty static. I'm not sure, how other AWS resources will behave if you import them to another cluster.

## Final Thoughts

Crossplane is promising in its aim to provide the single entry point to manage both your infrastructure and an application, which is running in Kubernetes. In other words, it can help you to completely abstract from your users the creation of certain cloud resources and limit the scope of their work to the application related configs only. It can reduce time to market by reducing the interactions between different teams (PR approvals, etc.)

It is also a big project backed by multiple people and it just hit its huge `v1.0` milestone. So, Crossplane is not just some small Kubernetes operator from GitHub managed by a single enthusiast. It's less likely that this project will be suddenly abandoned. So, you can sleep better at night.

Crossplane doesn't try to backport an existing IaC solution into Kubernetes as well and I'm pretty damn happy about it! I already saw things like AWS CloudFormation Operator or Terraform Operator. Of course, it's easier to build your product on top of an existing tool, but Crossplane doesn't need to fit it into k8s conventions. It's cloud-native by design!

However, I have concerns regarding this tool as well. First of all, it has some rough edges and a bit rigid configuration when it comes to authentication against AWS and so on.

Another thing that bothers me is: what would happen if a provider pod dies during the reconciliation? These are probably just flashbacks to Terraform since you only need to fire a couple of API calls and then check the resource status from time to time.

And lastly, Crossplane might be tough to manage in a multi-cluster setup. Which cluster manages what? Shall you have a dedicated "infra" cluster (I would advocate against it!) and many more other questions to answer. This is just a very young tool. Hence, there are not many best practices around it yet.

Wrapping this up. If you ask me whether you should use Crossplane or not, I would say: check where you are. You have a brand new greenfield project with just a couple of infra dependencies, which runs on Kubernetes and you also want to try something new? Sure, give it a try! However, you have to be ready to rebuild things often, because some breaking changes can be merged into the tool itself or you may later discover that it doesn't cover all your use cases. Even if you won't manage your whole infra with this tool, you can drastically increase your velocity, because you won't need to onboard any DSLs.

If you are running a relatively old infrastructure with some skeletons in the closet set up with probably more than one IaC tool, Crossplane might be not for you. Your engineering teams are probably already onboarded with existing solutions and workarounds. Keep in mind, that Crossplane requires a Kubernetes cluster and you may not even have a cluster in all the environments. Also, there is a risk that you won't be able to migrate all the infrastructure that you want to Crossplane and it will end up "just as yet another tool to do the infra that we have".

### Everything in a Single Package

{{< figure src="https://crossplane.io/docs/v0.3/media/crossplane-overview.png" alt="multi-tenant architecture" caption="source: https://crossplane.io/docs/v0.3/" >}}

The special beauty of Crossplane is that now you can create a bundle, which not only describes your application, but also its infrastructure dependencies. Moreover, you can describe your app's deployable and it's infra, in the same way, using the same syntax. Of course, real life is more complicated than a code example on the Internet, but with the growing popularity of managed Kubernetes solutions, I can foresee that this project will get much wider adoption.
