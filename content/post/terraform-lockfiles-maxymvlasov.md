---
title: "How to add, use, and update `.terraform.lock.hcl` without pain"
date: 2023-06-20
author: ["Maksym Vlasov"]
draft: false
slug: "terraform-lockfiles-maxymvlasov"
cover:
  image: "https://www.datocms-assets.com/2885/1596833357-terraformshare3.jpg"
categories: ["Tech"]
tags: ["terraform", "hashicorp", "en", "guest"]
---

_**This is the first guest article in this blog. This is one is by Maksym Vlasov - my co-author of the [CatOps channel](https://t.me/catops).**_

## Pre-history

As you may know, Terraform 1.4.0 has introduced [changes](https://github.com/hashicorp/terraform/pull/32129), which break the previous unintentional behavior. Previously, you could ignore the lockfile and use cached providers as long as the version constraints in the code were okay with your local cache. Starting from 1.4.0, Terraform always checks the lockfile before going into your cache directory. In practice, it means that if you ignore the lockfile or remove it completely, Terraform will run full init, no matter what is in your `TF_CACHE_DIR` or the `.terraform` directory.

So, here are [a few options](https://github.com/runatlantis/atlantis/issues/3201) to solve this:

* Keep using Terraform 1.3.x as the new 0.11
* Set `TF_PLUGIN_CACHE_MAY_BREAK_DEPENDENCY_LOCK_FILE=true`
* Start using the lockfile and move on

When all hope was gone that our lovely workflow with `**/.terraform.lock.hcl` in `.gitignore` won't backfire, I chose to try to add `.terraform.lock.hcl` to all our 289 root modules. You may ask:

## Why are these lockfiles needed?

Well, except "highly recommended by Hashicorp way", which force you to use lockfiles, here are a few additional reasons why you would like to use them - Repeatability and Security.

### Repeatability

Imagine that you have `aws` or `kubernetes` provider and you trust that the maintainers use [SemVer](https://semver.org/) as it is designed. So, you specify:

```terraform
terraform {
  required_version = "~> 1.3"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}
```

Everything works nice, until...

* Provider is going to be broken for 3 workdays, because of a lack of testing ¯\\\_(ツ)\_/¯

  {{< figure alt="3 days of broken Kubernetes provider" src="/img/posts/terraform-lockfiles-maxymvlasov/kubernetes-provider-broken-for-3-workdays.png" >}}


* Provider adds a breaking change in a minor release because they _forgot_ to add it in a major one

  {{< figure alt="AWS breaking changes on minor version" src="/img/posts/terraform-lockfiles-maxymvlasov/aws-provider-breaking-changes-on-minor-version.png" >}}

Both of these issues happened last month.

Of course, you can set patch versions explicitly. For example, `"5.0.0"` or `"2.19.0"` and use Renovate/dependabot or [`tfupdate` pre-commit hook](https://github.com/antonbabenko/pre-commit-terraform#tfupdate) for intentional updates. Yet, with `tfupdate` you're forced to use exactly one terraform/provider/module version across all the source code. This way you will avoid the problems above. But there is more.

### Security

There are these `h1` and `zh` [hashes](https://developer.hashicorp.com/terraform/language/files/dependency-lock#hashes) inside `.terraform.lock.hcl`:

```hcl
provider "registry.terraform.io/hashicorp/kubernetes" {
  version     = "2.21.1"
  constraints = ">= 2.21.1, < 3.0.0"
  hashes = [
    "h1:2spGoBcGDQ/Csc23bddCfM21zyKx3PONoiqRgmuChnM=",
    "h1:7cCH+Wsg2lFTpsTleJ7MewkrYfFlxU1l4HlLWP8wzFw=",
    "h1:I1qWLUFmB0Z8+3CX+XzJtkgiAOYQ1nHlLN9lFcPf+zc=",
    "h1:gP8IU3gFfXYRfGZr5Qws9JryZsOGsluAVpiAoZW7eo0=",
    "zh:156a437d7edd6813e9cb7bdff16ebce28cec08b07ba1b0f5e9cec029a217bc27",
    "zh:1a21c255d8099e303560e252579c54e99b5f24f2efde772c7e39502c62472605",
    "zh:27b2021f86e5eaf6b9ee7c77d7a9e32bc496e59dd0808fb15a5687879736acf6",
    "zh:31fa284c1c873a85c3b5cfc26cf7e7214d27b3b8ba7ea5134ab7d53800894c42",
    "zh:4be9cc1654e994229c0d598f4e07487fc8b513337de9719d79b45ce07fc4e123",
    "zh:5f684ed161f54213a1414ac71b3971a527c3a6bfbaaf687a7c8cc39dcd68c512",
    "zh:6d58f1832665c256afb68110c99c8112926406ae0b64dd5f250c2954fc26928e",
    "zh:9dadfa4a019d1e90decb1fab14278ee2dbefd42e8f58fe7fa567a9bf51b01e0e",
    "zh:a68ce7208a1ef4502528efb8ce9f774db56c421dcaccd3eb10ae68f1324a6963",
    "zh:acdd5b45a7e80bc9d254ad0c2f9cb4715104117425f0d22409685909a790a6dd",
    "zh:f569b65999264a9416862bca5cd2a6177d94ccb0424f3a4ef424428912b9cb3c",
    "zh:fb451e882118fe92e1cb2e60ac2d77592f5f7282b3608b878b5bdc38bbe4fd5b",
  ]
}
```

Terraform uses them to pull exactly the same artifacts for your platform, as they were used during the last `terraform init` and => `terraform apply` commands.

It decreases the probability of the [supply chain attack](https://en.wikipedia.org/wiki/Supply_chain_attack) when the weakest link in your supply chain is terraform providers.

## Preparation for lockfiles addition

> **Note**: In all the examples below I used GitHub Workflows. Yet, you can port it to any other CI.

First of all, you need to have a valid terraform configuration.

You cannot just skip this step if you have a huge terraform codebase: almost certainly there is something broken.

So, let me introduce to you the `terraform validate` command! Just kidding. Yet, this is exactly what we need here. Sometimes the validation requires the `terraform init -backend=false` setting which you need to run for all the root modules.

For this case, there is another `pre-commit` solution, which inits your modules (and fixes existing `.terraform` if they are outdated/broken), and run validations. To use it you have to:

1. [Install dependencies in any way described in the `pre-commit-terraform`](https://github.com/antonbabenko/pre-commit-terraform#how-to-install).
2. Create a `.github/.pre-commit-tf-lockfiles.yaml` file with the content as below:

    > **Note**: We will use this file to auto-update lockifles in the CI later. `.github/` is present in the file path just to hide it from regular users and keep it as close as possible to `.github/workflows/`

    ```yaml
    repos:
      - repo: https://github.com/antonbabenko/pre-commit-terraform
        rev: v1.81.0
        hooks:
          - id: terraform_validate
            args:
              - --hook-config=--retry-once-with-cleanup=true
              - --tf-init-args=-upgrade
            # files: '^path/to/your/terraform/root/folder/[a-c]'
            exclude: '(\.)?modules/'

          # - id: terraform_providers_lock
          #   args:
          #   - --hook-config=--mode=always-regenerate-lockfile
          #   - --args=-platform=linux_arm64
          #   - --args=-platform=linux_amd64
          #   - --args=-platform=darwin_amd64
          #   - --args=-platform=darwin_arm64
          #   files: '^path/to/your/terraform/root/folder/[a-c]'
          #   exclude: '(\.)?modules/'
    ```

3. If you have huge repo - uncomment this line and specify the correct `# files: '^path/to/your/terraform/root/folder/[a-c]'`  

    `files` and `exclude` uses a Python `re.search` regex ([docs](https://pre-commit.com/#regular-expressions)). By specifying `[a-c]` at the end, we can limit the number of directories that should be processed by a single run

4. Run the command and chill for a couple of minutes

    ```bash
    pre-commit run -a --config .github/.pre-commit-tf-lockfiles.yaml
    ```

5. Once the command has finished, check that all your root modules pass the validation. If not, fix the errors and rerun `pre-commit` until all the modules are valid.

6. Edit your `.gitignore` to not ignore lockfiles. For example:

    ```txt
    !path/to/your/terraform/root/folder/[a-c]*/.terraform.lock.hcl
    !path/to/your/terraform/root/folder/[a-c]*/**/.terraform.lock.hcl
    ```

## Add lockfiles

1. Go to the previously created `.github/.pre-commit-tf-lockfiles.yaml` and:

    * Uncomment `terraform_providers_lock` hook
    * Set your [`-platform=`'s](https://developer.hashicorp.com/terraform/registry/providers/os-arch)
    * Copy `files` and `exclude` sections from `terraform_validate` to `terraform_providers_lock`
    * Comment `terraform_validate` hook to save extra time

2. Run the command below. It takes more time than the first command, so you can do something else in the meantime.

    ```bash
    pre-commit run -a --config .github/.pre-commit-tf-lockfiles.yaml
    ```

    In my tests, it took about ~2,5s per platform per provider per root module. So, for a module with 6 providers with 4 platforms, you may need about 1 minute to generate a lockfile.

3. Check that all lockfiles have `zh` hashes for each provider.

    Don't forget to remove empty files generated in the Preparation section for the directories without the terraform code.

    If some lockfiles do not have all the required hashes, check the logs. In most cases, it means that you still use something from the Terraform 0.11, which does not support one of the specified platforms (in my case `-platform=darwin_arm64` for `hashicorp/template` and `mumoshu/helmfile`)

4. If you also encounter these problems, modify `.github/.pre-commit-tf-lockfiles.yaml` and rerun `pre-commit` until everything is Ok:

    ```yaml
     - id: terraform_providers_lock
        args:
          - --hook-config=--mode=always-regenerate-lockfile
          - --args=-platform=linux_arm64
          - --args=-platform=linux_amd64
          - --args=-platform=darwin_amd64
          - --args=-platform=darwin_arm64
        exclude: |
          (?x)
            (/(\.)?modules/

            # hashicorp/template 2.2.0 is not available for darwin_arm64
            |^terraform/bootstrap/
            # mumoshu/helmfile 0.14.0 is not available for darwin_arm64.
            |^terraform/helmfiles/
          )

      # TODO: Rewrite these modules to newer providers
      - id: terraform_providers_lock
        name: Lock terraform provider versions w/o darwin_arm64
        args:
          - --hook-config=--mode=always-regenerate-lockfile
          - --args=-platform=linux_arm64
          - --args=-platform=linux_amd64
          - --args=-platform=darwin_amd64
        files: |
          (?x)
            # hashicorp/template 2.2.0 is not available for darwin_arm64
            (^terraform/bootstrap/
            # mumoshu/helmfile 0.14.0 is not available for darwin_arm64.
            |^terraform/helmfiles/
          )
    ```

    > **Note**: To save some time, you may need to comment out the last hook section for future lockfiles generation.

## Automate lockfile updates in CI

Once you have all the lockfiles, it's time to automate their updates.

1. Go to `.github/.pre-commit-tf-lockfiles.yaml` and:

    * Change `terraform_validate` `files` sections to:

        ```yaml
        files: '\.terraform\.lock\.hcl$'
        ```

        to limit `terraform init` run only for dirs with lockflie.

    * Remove `files` sections in the `terraform_providers_lock` hooks

    In the end, you will get something like this:

    ```yaml
    repos:
      - repo: https://github.com/antonbabenko/pre-commit-terraform
        rev: v1.81.0
        hooks:
          - id: terraform_validate
            args:
              - --hook-config=--retry-once-with-cleanup=true
              - --tf-init-args=-upgrade
            files: '\.terraform\.lock\.hcl$'

        - id: terraform_providers_lock
            args:
              - --hook-config=--mode=always-regenerate-lockfile
              - --args=-platform=linux_arm64
              - --args=-platform=linux_amd64
              - --args=-platform=darwin_amd64
              - --args=-platform=darwin_arm64
            exclude: |
              (?x)
                (/(\.)?modules/

                # hashicorp/template 2.2.0 is not available for darwin_arm64
                |^terraform/bootstrap/
                # mumoshu/helmfile 0.14.0 is not available for darwin_arm64.
                |^terraform/helmfiles/
              )

          # TODO: Rewrite these modules to newer providers
          - id: terraform_providers_lock
            name: Lock terraform provider versions w/o darwin_arm64
            args:
              - --hook-config=--mode=always-regenerate-lockfile
              - --args=-platform=linux_arm64
              - --args=-platform=linux_amd64
              - --args=-platform=darwin_amd64
            files: |
              (?x)
                # hashicorp/template 2.2.0 is not available for darwin_arm64
                (^terraform/bootstrap/
                # mumoshu/helmfile 0.14.0 is not available for darwin_arm64.
                |^terraform/helmfiles/
              )
    ```

2. Add a GitHub workflow, which installs all the dependencies and run `pre-commit run` every Monday. It creates a new PR in the Renovate style:

    ```yaml
    name: Maintain Terraform lockfile up-to-date
    # It is required at least Renovate fixes https://github.com/renovatebot/renovate/issues/22417

    on:
      workflow_dispatch: {}

      schedule:
        - cron: '0 4 * * 1' # Execute every Monday at 04:00

    permissions:
      contents: write
      pull-requests: write


    env:
      # Prevent GH API rate-limit issue
      GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}

    jobs:
      pre-commit-tf-lockfile:
        runs-on: ubuntu-latest
        container: python:3.11-slim
        steps:
        - name: Install container pre-requirements
          run: |
            apt update
            apt install -y \
                git \
                curl \
                unzip \
                jq \
                nodejs # Needed for Terraform installation
            curl -L https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 > /usr/bin/yq &&\
              chmod +x /usr/bin/yq
        - name: Checkout
          uses: actions/checkout@c85c95e3d7251135ab7dc9ce3241c5835cc595a9 # v3.5.3
          with:
            ref: ${{ github.base_ref }}

        - uses: actions/checkout@c85c95e3d7251135ab7dc9ce3241c5835cc595a9 # v3.5.3
        - run: |
            git config --global --add safe.directory /__w/infrastructure/infrastructure
            git fetch --no-tags --prune --depth=1 origin +refs/heads/*:refs/remotes/origin/*

        - uses: hashicorp/setup-terraform@633666f66e0061ca3b725c73b2ec20cd13a8fdd1 # v2.0.3
          with:
            terraform_version: ~1.3

        - name: Execute pre-commit
          uses: pre-commit/action@646c83fcd040023954eafda54b4db0192ce70507 # v3.0.0
          with:
            extra_args: >
              --all-files
              --config .github/.pre-commit-tf-lockfiles.yaml
              --color=always --show-diff-on-failure

        - name: Create Pull Request
          if: failure()
          id: cpr
          uses: peter-evans/create-pull-request@284f54f989303d2699d373481a0cfa13ad5a6666 # v5.0.1
          with:
            commit-message: 'chore(deps): Update terraform lockfiles'
            branch: pre-commit/update-tf-lockfiles
            delete-branch: true
            title: 'chore(deps): Update terraform lockfiles'
            body: >
              This PR update provider versions in Terraform lockfiles to their most resent values

              > **Warning**: Before merge, please, make sure that all Terraform CI runs pass successfully.
            labels: auto-update
            branch-suffix: timestamp

        - name: Pull Request number and link
          if: failure() && steps.cpr.outputs.pull-request-number > 0
          run: |
            echo "Pull Request Number - ${{ steps.cpr.outputs.pull-request-number }}"
            echo "Pull Request URL - ${{ steps.cpr.outputs.pull-request-url }}"
    ```

    {{< figure alt="Autoupdate PR for lockfiles" src="/img/posts/terraform-lockfiles-maxymvlasov/lockfile-autoupdate-pr.png" >}}


    For 289 root modules with 1180 lockfile provider definitions, it takes 2h 40min or ~2,288s per platform per provider, which is ~0.2s faster than running it locally.

Well, that's it from the technical perspective. It's time to deal with the questions like: _Why don't you just use Renovate for this, dude?_

### Why not Renovate?

Yes, I heard about Renovate and dependabot. Check out, my talks about Renovate at [Anton Babenko stream](https://www.youtube.com/live/l28pukLJvss?feature=share) and [HUG Kyiv](https://www.youtube.com/live/uK8QgE17dzg?feature=share&t=3012)(Ukr).

We don't use dependabot for the infrastructure repository because it has too many problems with monorepos, you can't simply [force `dependabot.yml` for the whole organization](https://heyko.medium.com/automated-dependabot-configuration-in-github-bb08e2c6eeeb), it is less configurable than Renovate, etc.

GitHub itself does not use dependabot properly in its repositories...

* A newly generated repository from [actions/typescript-action](https://github.com/actions/typescript-action) contains

    {{< figure alt="Newly generated repo dependabot PR" src="/img/posts/terraform-lockfiles-maxymvlasov/newly-generated-repo.jpg" >}}
* Dependabot PR is there for 2 weeks!

    {{< figure alt="How long can a dependabot PR live without any attention" src="/img/posts/terraform-lockfiles-maxymvlasov/github-dependabot-alert-long-live.png" >}}

I maintain Renovate for my organization here: [Sharable Config Presets for Renovatebot, especially useful for DevOps folks](https://github.com/SpotOnInc/renovate-config/). Also, Renovate has a [`lockFileMaintenance`](https://docs.renovatebot.com/configuration-options/#lockfilemaintenance) option but...

* For now, Renovate cannot resolve [`!=`](https://developer.hashicorp.com/terraform/language/expressions/version-constraints#version-constraint-syntax) version constrain ([renovate/#22417](https://github.com/renovatebot/renovate/issues/22417)), so it just fails to create a PR if at least one version constraint with `!=` exists in a repository.
* If you don't have any  `!=`, Renovate creates nice PRs, but it does not respect provider constraints used inside the child modules of your root module.  

    So you get something like

    ```hcl
    provider "registry.terraform.io/hashicorp/aws" {
    version     = "5.2.0"
    constraints = "~> 5.0"
    ```

    in cases when `terraform providers lock` command will create something like

    ```hcl
    provider "registry.terraform.io/hashicorp/aws" {
    version     = "5.2.0"
    constraints = ">= 2.0.0, >= 3.0.0, >= 3.64.0, >= 4.0.0, >= 4.9.0, >= 4.18.0, >= 4.22.0, >= 4.23.0, >= 4.49.0, ~> 5.0"
    ```

    And it will work fine until someone inside these modules wouldn't specify `!= 5.2.0` or `< 5.2.0`.

* Renovate specifies all the available `h1` hashes (all available provider platforms), which is pretty nice. Yet, it does not specify "vanilla" `zh` hashes, which, in my opinion, are [more strict](https://developer.hashicorp.com/terraform/language/files/dependency-lock#zh). Thus, I prefer to have `zh` hashes when it is possible.

And one more thing:

## Make sure to add lockfiles to all the new root modules

Just introduce a rule for your terraform configurations:

**Run `terraform init` when you add a new root module.**

It adds a basic `.terraform.lock.hcl`, which you can commit as it is and wait for the next lockfile update job.

Or, you could add `.pre-commit-config.yaml` with:

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.81.0
    hooks:
      # Validate and run `terraform init` which needed for terraform_providers_lock
      - id: terraform_validate
      - id: terraform_providers_lock
        args:
          - --hook-config=--mode=only-check-is-current-lockfile-cross-platform
          - --args=-platform=linux_arm64
          - --args=-platform=linux_amd64
          - --args=-platform=darwin_amd64
          - --args=-platform=darwin_arm64
        ## TODO: Rewrite these modules to newer providers
        # exclude: |
        #   (?x)
        #     (/(\.)?modules/

        #     # hashicorp/template 2.2.0 is not available for darwin_arm64
        #     |^terraform/bootstrap/
        #     # mumoshu/helmfile 0.14.0 is not available for darwin_arm64.
        #     |^terraform/helmfiles/
        #   )
```

And automate `pre-commit` executions in PRs like [this](https://github.com/antonbabenko/pre-commit-terraform/blob/32597545e7fff922a19c3b3c7cecf3fd6bc6b628/.github/workflows/pre-commit.yaml) or [this](https://github.com/SpotOnInc/renovate-config/blob/dfcbbf82cecc20e6d759c058399fdfda3956c62f/.github/workflows/pre-commit.yaml).

## Summary

Here are a few takeaways:

* It's better to have lockifles than have no (repeatability, security)
* It makes sense to automate all these updates and maintain the same versions across the codebase, and be on the cutting edge without bleeding ([Renovate](https://github.com/renovatebot/renovate), [dependabot](https://github.com/dependabot/dependabot-core), [tfupdate via pre-commit](https://github.com/antonbabenko/pre-commit-terraform/#tfupdate))
* For a better user experience, you need a source of truth (automated `terraform plan` in CI, Terratests, etc.) which shows that changes do not break anything. It can be [Atlantis](https://github.com/runatlantis/atlantis), [Spacelift](https://spacelift.io/), [Terraform Cloud](https://www.hashicorp.com/products/terraform), or you can do it in your own CI.

_**If you want to publish your article here as well, just ping me. My contacts are all over this blog.**_
