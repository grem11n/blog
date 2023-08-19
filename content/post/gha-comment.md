---
title: "Trigger a GitHub Action Pipeline with a Comment"
date: 2023-08-18
draft: false
slug: "gha-comment-trigger"
cover:
  image: "https://techworm.net/programming/wp-content/uploads/2018/10/github-actions.jpg"
categories: ["Tech"]
tags: ["cicd", "github", "actions", "graphql"]
---

# How to Trigger a GitHub Actions Pipeline with a Comment

Building comment-based workflows is a pretty neat thing from the UX perspective. You can work on the code in your IDE, create a pull request, and then leverage PR comments to run some automation.

GitHub Actions is a native CI if you're using GitHub (which you probably do). It's convenient to use because you don't have to configure a CI server for your project or open an account with another cloud CI.

Usually, you create a pull request event and your CI kicks in. Yet, it can be handy if you have some long-running tasks that you don't need to run every time. For example, you may have end-to-end tests which you don't want to run for every change. Frankly, this task required me to do some research on GitHub Actions and GitHub's GraphQL API. I also hit some road bumps along the way. Unfortunately, I haven't found any step-by-step guide for building such workflows, even though this use case seems common. Thus, I decided to create one! I hope you will enjoy it!

There are a couple of caveats with comment-based pipelines, which I want to discuss here. I will try to create an [Atlantis](https://www.runatlantis.io/)-like workflow for an arbitrary PR using GitHub Actions. These caveats are in _italics_, so you can just look for them in the text if you don't want to read everything.

## Building the Workflow

First, you should set [the trigger action](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#issue_comment) for your pipeline to the `issue_comment`.

```yaml
name: My Comment-based Pipeline
on:
  issue_comment:
    types:
      - created
...
```

You can run workflows on events other than `create`, but you likely do not need that.

And here come the first caveats with such type of workflows:

- _The workflow is only available once it is merged into your `main` branch. It also means that you need to merge a new PR to change anything about the pipeline. This makes testing a bit cumbersome, but I fully understand why GitHub goes with this approach._
- _GitHub API often treats `Issues` and `Pull Requests` as the same entity. Yet, sometimes they're not. It brings some confusion when working with API or setting the conditions for your workflow._

Thus, you need to filter only PR comments in your workflow, which can be easily done with the `${{ github.event.issue.pull_request }}` stanza. Another important thing to keep in mind is that _**it is easy to abuse your comment-based workflow**_ if there are no safety nets in place. By safety nets here I mean that you need to watch only for specific comments and stop the execution immediately if the condition isn't met.

Luckily, it's easy to do with the conditionals:

```yaml
...
jobs:
  long-job:
    name: A long job that only runs on a specific PR comment
    if: ${{ github.event.issue.pull_request && github.event.comment.body == '/build' }}
    steps:
...
```

I used the codeword `/build` in this example, but you can use whatever you want. Also, you can use GHA expressions like [`contain()`](https://docs.github.com/en/actions/learn-github-actions/expressions#contains) to trigger a workflow on a specific word or a phrase in the comment body.

The next caveat is that _GitHub won't show workflows triggered with the `issue_comment` action in the status section of your PR._ Yet, there's an easy way to fix that.  We are going to do the same thing that [Atlantis](https://www.runatlantis.io/) does and put an emoji to the trigger comment to display the status. _The list of available emojis for PR comments is [quite limited](https://github.com/LangLangBart/gh-look/blob/main/gh-look#L67)_, but we can work with it.

You can manage the emojis with GitHub CLI which provides direct access to GraphQL API. The neat part is that [GitHub CLI is preinstalled on all GitHub-hosted runners](https://docs.github.com/en/actions/using-workflows/using-github-cli-in-workflows).

```yaml
...
    steps:
      - name: Put a reaction to the comment
        run: gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_ID: ${{ github.event.comment.node_id }}
...
```

Let's focus on this code snippet a bit more. I do not have much experience with GraphQL. So, this part was counterintuitive for me at first.

A GraphQL request contains its type: `query` or `mutation` as well as arguments required for this request. We can break down our "AddReaction" request like this:

```
mutation AddReaction {	<-- request type and arbitrary name for it
    addReaction( <-- the mutation itself. See: https://docs.github.com/en/graphql/reference/mutations#addreaction
        input:{
            subjectId: "$NODE_ID",	<-- GraphQL node ID for the triggering comment. Notice _this is not the same as the REST comment it!_
            content: EYES <-- Emoji name
        }
    ){
        reaction {	<-- The body of the request comes below
            content
        }
        subject {
            id
        }
    }
}
```

For whatever reason _I was unable to split this GraphQL payload into multiple lines in my GHA workflow_, but if you have more luck, please, let me know.

Also, notice that you have to provide `GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` in order for GitHub CLI to work and `NODE_ID: ${{ github.event.comment.node_id }}` which is the GraphQL Node ID for the comment we want emoji on.

Another caveat is that by default GHA builds the workflow for any pull request. However, you may want to limit the execution to the open PRs only. GitHub API comes to the rescue here!

```yaml
...
      - name: Check if PR is open
        run: |
          STATE=$(gh pr view $PR_NUMBER --repo ${{ github.repository }} --json state --jq .state)
          if [ "$STATE" != "OPEN" ]; then
            echo "Cannot build for closed PRs"
            (
              echo "**${{ github.workflow }}**"
              echo "Cannot build Kuby for a closed PR. Use the `latest` version (built for the `master` branch) or create a new PR."
            ) | \
            gh pr comment "${PR_NUMBER}" --repo ${{ github.repository }} -F -
            gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:THUMBS_DOWN}){reaction{content}subject{id}}}"
            gh api graphql --silent --raw-field query="mutation RemoveReaction {removeReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
            exit 1
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.issue.number }}
          NODE_ID: ${{ github.event.comment.node_id }}
...
```

There are a couple of new things here. First, I'm using the GitHub CLI to identify the status of the pull request. I must provide the repository reference because I haven't cloned any code yet.

If the PR is not in the `OPEN` state, I put a comment there using GitHub CLI to notify the user that they can only run this workflow for open pull requests.

Then I use the familiar `addReaction` to put a `thumbs_down` emoji on that comment and a new `removeReaction` mutation to remove the previous `eyes` emoji. `removeReaction` works the same as `addReaction`, just in reverse :)

This step could be the first one in the workflow. Yet, I still want to use emojis for UX purposes. So, it's up to you where to put this validation.

Finally, we came to the part where you do meaningful work. Here you clone your code and run whatever long task you need. I won't focus on this part because this custom part has nothing to do with the comment-based workflow.

Yet, once all the work is done, you may want to notify your user. Remember, GitHub won't display this workflow in the `status` section!

```yaml
...
      - name: Final Comment
        run: |
          gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:THUMBS_UP}){reaction{content}subject{id}}}"
          gh api graphql --silent --raw-field query="mutation RemoveReaction {removeReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
          (
            echo "**${{ github.workflow }}**"
            echo "The long task is done!"
            echo
            echo "You can find the workflow here:"
            echo "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          ) | \
          gh pr comment "${PR_NUMBER}" --repo ${{ github.repository }} -F -
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.issue.number }}
          NODE_ID: ${{ github.event.comment.node_id }}
...
```

Similarly to the "Check if PR is Open" step, we replace the `eyes` emoji, this time with `thumbs_up` to tell the user that their workflow is successful. Afterward, we put a small comment with the link to help them find the workflow.

In that comment, you can output whatever you want. For example, you can use [GHA Outputs](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs) to output some artifacts from the previous steps.

And one more thing.

It's a rule of good tone to notify your users if the workflow has failed. In GitHub Actions you can do that with another job in your workflow that is triggered if your main job has failed.

```yaml
...
  notify-job:
    needs: [build]
    if: ${{ always() && contains(needs.*.result, 'failure') }}	<-- check that status of the previous job
    steps:
      - name: Notify on Failure
        run: |
          gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:THUMBS_DOWN}){reaction{content}subject{id}}}"
          gh api graphql --silent --raw-field query="mutation RemoveReaction {removeReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
          (
            echo "**${{ github.workflow }}**"
            echo "**Something went wrong!**"
            echo
            echo "**Details:** ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          ) | \
          gh pr comment "${PR_NUMBER}" --repo ${{ github.repository }} -F -
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.issue.number }}
          NODE_ID: ${{ github.event.comment.node_id }}
```

This step is basically the same as the "Final Comment" with a different wording. The most important part is the conditions below:

```yaml
    needs: [build]
    if: ${{ always() && contains(needs.*.result, 'failure') }}
```

This way, we ensure that:

- This job runs even if the prerequisite defined with `needs` has failed. `always()` stands for that.
- At the same time, we only want to execute this code on failures. Hence, the `&& contains(needs.*.result, 'failure')` part.

## The End Results

So, the resulting pipeline looks somewhat like below.

<details><summary>Click here to expand</summary>

```yaml
name: My Comment-based Pipeline
on:
  issue_comment:
  types:
    - created

jobs:
  long-job:
    name: A long job that only runs on a specific PR comment
    if: ${{ github.event.issue.pull_request && github.event.comment.body == '/build' }}
    steps:
      - name: Put a reaction to the comment
        run: gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_ID: ${{ github.event.comment.node_id }}
 
      - name: Check if PR is open
        run: |
          STATE=$(gh pr view $PR_NUMBER --repo ${{ github.repository }} --json state --jq .state)
          if [ "$STATE" != "OPEN" ]; then
            echo "Cannot build for closed PRs"
            (
              echo "**${{ github.workflow }}**"
              echo "Cannot build Kuby for a closed PR. Use the `latest` version (built for the `master` branch) or create a new PR."
            ) | \
            gh pr comment "${PR_NUMBER}" --repo ${{ github.repository }} -F -
            gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:THUMBS_DOWN}){reaction{content}subject{id}}}"
            gh api graphql --silent --raw-field query="mutation RemoveReaction {removeReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
            exit 1
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.issue.number }}
          NODE_ID: ${{ github.event.comment.node_id }}
 
      - name: Checkout source code from Github
        uses: actions/checkout@v3.5.3
        with:
          fetch-depth: 0
 
      - name: Run a long task
        run: |
          make long-task
 
      - name: Final Comment
        run: |
          gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:THUMBS_UP}){reaction{content}subject{id}}}"
          gh api graphql --silent --raw-field query="mutation RemoveReaction {removeReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
          (
            echo "**${{ github.workflow }}**"
            echo "The long task is done!"
            echo
            echo "You can find the workflow here:"
            echo "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          ) | \
          gh pr comment "${PR_NUMBER}" --repo ${{ github.repository }} -F -
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.issue.number }}
          NODE_ID: ${{ github.event.comment.node_id }}
 
  notify-job:
    needs: [build]
    if: ${{ always() && contains(needs.*.result, 'failure') }}	<-- check that status of the previous job
    steps:
      - name: Notify on Failure
        run: |
          gh api graphql --silent --raw-field query="mutation AddReaction {addReaction(input:{subjectId:\"$NODE_ID\",content:THUMBS_DOWN}){reaction{content}subject{id}}}"
          gh api graphql --silent --raw-field query="mutation RemoveReaction {removeReaction(input:{subjectId:\"$NODE_ID\",content:EYES}){reaction{content}subject{id}}}"
          (
            echo "**${{ github.workflow }}**"
            echo "**Something went wrong!**"
            echo
            echo "**Details:** ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          ) | \
          gh pr comment "${PR_NUMBER}" --repo ${{ github.repository }} -F -
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.issue.number }}
          NODE_ID: ${{ github.event.comment.node_id }}
```

</details>

---
That's all, folks! Hope this content was useful to you. If you would like to get more content like this, feel free to subscribe to my [Substack](https://newsletter.catops.dev/) or the [Telegram channel](https://t.me/catops).

Cheers!

## Useful Links

- [GitHub CLI Manual](https://cli.github.com/manual/)
- [GH Look](https://github.com/LangLangBart/gh-look/) - a tool from which I have borrowed the idea to use GraphQL API for emojis
- [GitHub GraphQL API Docs](https://docs.github.com/en/graphql)
- [Forming calls with GraphQL](https://docs.github.com/en/graphql/guides/forming-calls-with-graphql)
- [GitHub Actions: Using conditions to control job execution](https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution)
- [Trigger a Github workflow if it matches a particular comment in the Pull Request](https://github.com/orgs/community/discussions/25389)
- [Issue_comment trigger event](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#issue_comment)
- [Using the GitHub CLI on a runner](https://docs.github.com/en/actions/examples/using-the-github-cli-on-a-runner)
- [GitHub actions get URL of test build](https://stackoverflow.com/questions/59073850/github-actions-get-url-of-test-build)
