---
layout: post
title:  "Most effective ways to push within GitHub Actions"
date:   2022-01-20 07:30:00 +0100
categories: build
tags: continuous integration github actions automate git
author: JohT
discussions-id: 34
---

[Continuous Integration [1]][ContinuousIntegration] goes far beyond testing and building code nowadays. Test code coverage and other reports might get created, documentation might get updated and metrics and statistics might get refreshed. This article shows how these results can be pushed back into the repository using [GitHub Actions [3]][GitHubActions].

{% raw %}

## Prerequisites

- [GitHub [2]][GitHub] Repository
- Basic knowledge of [GitHub Actions [3]][GitHubActions]
- Workflow YML file in `.github/workflows/`

## Introduction
### Strive for fast feedback

Be prepared that it will likely need a couple of attempts to get a GitHub Actions Workflow to work as intended. Here are some ideas that might help:

- [Dump out all GitHub Actions context [4]][DumpContext]
- [GitHub Actions Logging [5]][GitHubActionsLogging]
- Dedicated experimental repository with a small, fast running pipeline.

### GitHub Events
Basic workflows simply use `on: [push]` to get triggered on every push regardless of the branch. The following slight enhancement is also widely used and adds some useful features. 

```yml
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that the default branch is named [main [6]][GitHubRenamingDefaultBranch] here.

<span style="font-size:1.8em;">&#9432;</span> 
Note that it might seem reasonable to use 

```yml
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
```

as described in [Github Actions workflow for merged/closed PRs [17]][MergedClosedPullRequestEvents]. However,  git commands won't work if the feature branch gets deleted right after the pull request merge.

#### What you can do with it:
- Differentiate between pull request and push event: 
```
if: github.event_name == 'push'
```
- Distinguish the original repository from a fork: 
```
if: github.event.pull_request.head.repo.full_name == github.repository
```
- Get the GIT reference of the feature branch: 
```
github.event.pull_request.head.ref
```
- Get additional information about the pull request like its title: 
```
github.event.pull_request.title
```

#### What you need to be aware of:
- `actions/checkout@v2` behaves differently
- `github.event.pull_request` properties will be empty when triggered by a main branch push
- `github.event.commits` properties will be empty when triggered by a pull request event

<span style="font-size:1.8em;">&#128214;</span> 
Further reading: [Events that trigger workflows [7]][WorkflowEvents]   
<span style="font-size:1.8em;">&#128214;</span> 
Further reading: [GitHub Actions: A deep dive into "pull_request"[8]][GitHubActionsPullRequestDeepDive]

### GIT commands

[Push to origin from GitHub Action [10]][PushOriginFromGitHubAction] shows an easy way to commit and push changed files to the repository. `git pull` fetches and merges intermediate changes and reduces the risk of race conditions when pipelines run in parallel. Here is an example using environment variables for the commit author and message:

```yml
- name: GIT commit and push all changed files
  env: 
    CI_COMMIT_MESSAGE: Continuous Integration Build Artifacts
    CI_COMMIT_AUTHOR: Continuous Integration
  run: |
    git config --global user.name "${{ env.CI_COMMIT_AUTHOR }}"
    git config --global user.email "username@users.noreply.github.com"
    git pull
    git commit -a -m "${{ env.CI_COMMIT_MESSAGE }}"
    git push
```

The following example shows how to only commit changed files in the docs folder:
```yml
- name: GIT commit and push docs
  env: 
    CI_COMMIT_MESSAGE: Continuous Integration Build Artifacts
    CI_COMMIT_AUTHOR: Continuous Integration
  run: |
    git config --global user.name "${{ env.CI_COMMIT_AUTHOR }}"
    git config --global user.email "username@users.noreply.github.com"
    git pull
    git add docs
    git commit -m "${{ env.CI_COMMIT_MESSAGE }}"
    git push
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that the username in `username@users.noreply.github.com` needs to be replaced.

## All-in-one solutions

Already existing all-in-one solutions provide a good starting point for many use cases. 
On the downside, their parameters and their behaviour might change over time and won't be as stable as git commands. 
As soon as more control or flexibility is needed, the examples below might be a better fit. 
If simplicity is key, also have a look at [example 1](#example-1).

- [git-auto-commit [18]][git-auto-commit] provides an easy to use solution:
> The GitHub Action for committing files for the 80% use case.

- [add-and-commit Action [21]][add-and-commit] provides additional control:
> This action lets you choose the path that you want to use when adding & committing changes so that it works as you would normally do using git on your machine.

### Pros

- No git commands needed
- Customizable
- Good documentation

### Cons

- Additional dependency
- Only for Unix/Linux systems
- [Example 1](#example-1) is nearly as easy

## The easiest solution

If the automatically generated and updated files should only be pushed into the repository on a push into the main branch e.g. after a pull request is merged, then the following example shows how this can be achieved. Even though it shows a JavaScript Node.js workflow, the build steps can easily be replaced by others.

### Example 1

```yml
name: Continuous Integration
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    env: 
      CI_COMMIT_MESSAGE: Continuous Integration Build Artifacts
      CI_COMMIT_AUTHOR: Continuous Integration
    steps:
    - uses: actions/checkout@v2

    # Build steps
    - uses: actions/setup-node@v2
      with:
        node-version: '12' 
    - name: Node Install
      run: npm ci
    - name: Node Build (lint, test, coverage, doc, build, package)
      run: npm run package

    # Commit and push all changed files.
    - name: GIT Commit Build Artifacts (coverage, dist, devdist, docs)
      # Only run on main branch push (e.g. after pull request merge).
      if: github.event_name == 'push'
      run: |
        git config --global user.name "${{ env.CI_COMMIT_AUTHOR }}"
        git config --global user.email "username@users.noreply.github.com"
        git pull
        git commit -a -m "${{ env.CI_COMMIT_MESSAGE }}"
        git push
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that it isn't necessary to prevent the workflow from being triggered again by the automatically executed push. [Triggering a workflow from a workflow [7]][WorkflowFromWorkflow] states that "events triggered by the `GITHUB_TOKEN` will not create a new workflow run". 

<span style="font-size:1.8em;">&#9432;</span> 
Note that if you use a personal access token for [actions/checkout@v2 [14]][GitHubActionsCheckout], the workflow will trigger itself again resulting in an endless loop. The next example shows how to solve this.

<span style="font-size:1.8em;">&#9888;</span> 
Be aware that this won't work when the main branch is protected. One way to overcome this is to create
a personal access token as an administrator and use this as secret token as shown in the following example.

### Example 2

This variation of [example 1](#example-1) uses a personal access token for the checkout and therefore needs to assure that the pipeline won't run again on the automatically created commit/push. 

This can be used to overcome the issue with protected branches that won't allow a push otherwise
as discussed in [How to push protected branches [23]][HowToPushProtectedBranches].
The personal access token needs to be created from an administrator or a technical user that is selected from the organisation as an exception within the [settings of the protected branch rules [24]][BranchProtectionRule]. The personal access token only needs to be granted for `public_repo` (public repositories).

```yml
name: Continuous Integration
on:
  push:
    branches:
      - main
    # Ignore changes in folders that are affected by the auto commit. (Node.js project)
    paths-ignore: 
      - 'coverage/**'
      - 'devdist/**'
      - 'dist/**'
      - 'docs/**'
  pull_request:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    env: 
      CI_COMMIT_MESSAGE: Continuous Integration Build Artifacts
      CI_COMMIT_AUTHOR: Continuous Integration
    steps:
    - uses: actions/checkout@v2
      with:
        token: ${{ secrets.WORKFLOW_GIT_ACCESS_TOKEN }}

    # Build steps
    - uses: actions/setup-node@v2
      with:
        node-version: '12' 
    - name: Node Install
      run: npm ci
    - name: Node Build (lint, test, coverage, doc, build, package)
      run: npm run package

    # Commit and push all changed files. 
    # Must only affect files that are listed in "paths-ignore".
    - name: GIT Commit Build Artifacts (coverage, dist, devdist, docs)
      # Only run on main branch push (e.g. pull request merge).
      if: github.event_name == 'push'
      run: |
        git config --global user.name "${{ env.CI_COMMIT_AUTHOR }}"
        git config --global user.email "username@users.noreply.github.com"
        git pull
        git add coverage devdist dist docs
        git commit -m "${{ env.CI_COMMIT_MESSAGE }}"
        git push
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that the folders in `paths-ignore` and `git add` need to be the same to prevent the workflow from being triggered by itself.

<span style="font-size:1.8em;">&#128214;</span> 
Further reading: [Creating a personal access token [19]][GitHubPersonalAcessToken]

<span style="font-size:1.8em;">&#128214;</span> 
Working example: [data-restructor-js [22]][data-restructor-js]

## Run some steps on auto commit

If some of the steps in the workflow should also be executed for the automatically generated commit/push, the commit name and author can be compared to detect the auto commit run. At least the step that creates the auto commit needs to be skipped when the auto commit triggered the workflow. Furthermore, a personal access token needs to be created, otherwise the whole workflow would be skipped for the automatically created commit.

### Auto commit environment variable

The following workflow step shows how the automatically created commit can be detected and how the result can be written into an environment variable. 

```yml
- name: Set environment variable "is-auto-commit"
  if: github.event.commits[0].message == env.CI_COMMIT_MESSAGE && github.event.commits[0].author.name == env.CI_COMMIT_AUTHOR
  run: echo "is-auto-commit=true" >> $GITHUB_ENV
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that this will only work for the GitHub event "push". The variables will be empty when triggered by a "pull_request" event. Thus, "is-auto-commit" would always be false. [GIT commit within a pull request](#git-commit-within-a-pull-request) shows how to solve this.

Add optional display steps for debugging and maintenance purposes as you like:

```yml
- name: Display Github event variable "github.event.commits[0].message"
  run: echo "last commit message = ${{ github.event.commits[0].message }}" 
- name: Display Github event variable "github.event.commits[0].author.name"
  run: echo "last commit author = ${{ github.event.commits[0].author.name }}" 
- name: Display environment variable "is-auto-commit"
  run: echo "is-auto-commit=${{ env.is-auto-commit }}"
```
### Skip push on auto commit

The environment variable defined above can then be used to skip selected steps.
At least the step that pushes the changed files into the repository needs to be skipped. 

```yml
- name: Commit build artifacts (dist, devdist, docs, coverage)
  # Only run on main branch push (e.g. pull request merge). 
  # Don't run again on an already pushed auto commit.
  if: github.event_name == 'push' && env.is-auto-commit == false
  run: |
    git config --global user.name "${{ env.CI_COMMIT_AUTHOR }}"
    git config --global user.email "username@users.noreply.github.com"
    git pull
    git commit -a -m "${{ env.CI_COMMIT_MESSAGE }}"
    git push
```

Any other steps can be skipped for the auto commit, but don't have to.

### Example 3

```yml
name: Continuous Integration
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    env: 
      CI_COMMIT_MESSAGE: Continuous Integration Build Artifacts
      CI_COMMIT_AUTHOR: ${{ github.event.repository.name }} Continuous Integration
    steps:
    - uses: actions/checkout@v2
      with:
        token: ${{ secrets.WORKFLOW_GIT_ACCESS_TOKEN }}

    # Set environment variable "is-auto-commit" 
    - name: Set environment variable "is-auto-commit"
      if: github.event.commits[0].message == env.CI_COMMIT_MESSAGE && github.event.commits[0].author.name == env.CI_COMMIT_AUTHOR
      run: echo "is-auto-commit=true" >> $GITHUB_ENV

    # Display variables for debugging
    - name: Display Github event variable "github.event.commits[0].message"
      run: echo "last commit message = ${{ github.event.commits[0].message }}" 
    - name: Display Github event variable "github.event.commits[0].author.name"
      run: echo "last commit author = ${{ github.event.commits[0].author.name }}" 
    - name: Display environment variable "is-auto-commit"
      run: echo "is-auto-commit=${{ env.is-auto-commit }}"

    # Build (will also run on auto commit)
    - uses: actions/setup-node@v2
      with:
        node-version: '12' 
    - name: Install node packages
      run: npm ci
    - name: Build package (lint, test, build, package, merge)
      run: npm run package

    # Commit and push all changed files.
    - name: Display event name 
      run: echo "github.event_name=${{ github.event_name }}"
    - name: Commit build artifacts (dist, devdist, docs, coverage)
      # Don't run again on already pushed auto commit. Don't run on pull request events.
      if: env.is-auto-commit == false && github.event_name != 'pull_request'
      run: |
        git config --global user.name "${{ env.CI_COMMIT_AUTHOR }}"
        git config --global user.email "joht@users.noreply.github.com"
        git pull
        git commit -a -m "${{ env.CI_COMMIT_MESSAGE }}"
        git push
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that a personal access token is needed as mentioned above.

## GIT commit within a pull request

If it is mandatory that automatically changed files need to be pushed within a pull request, e.g. to be able to review them, then there are a couple of things that need to be taken into account:

- In contrast to the [push event payload [12]][GitHubActionsPushEventPayload] there is currently no way to get the last commit from the [pull request event payload [13]][GitHubActionsPullRequestEventPayload]. 

- [Ignoring paths [11]][GitHubActionsIgnoringPaths] is off the table, since a skipped workflow run on the latest commit in the pull request is currently seen as "unsuccessful" (yellow state) and can't be merged if a successful run is mandatory. 

- [actions/checkout@v2 [14]][GitHubActionsCheckout] behaves differently for the "pull_request" event and needs to be tweaked to be able to get the intended commit message and author using git commands.

### Adapt Checkout

The ref parameter of [actions/checkout@v2 [14]][GitHubActionsCheckout] needs to be set to the head reference of the [pull request [8]][GitHubActionsPullRequestDeepDiveCheckout] to checkout the feature branch and be able to get the last commit of it:

```yml
- uses: actions/checkout@v2
  with:
    ref: ${{ github.event.pull_request.head.ref }}
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that this also works when the workflow is triggered by a push event. The reason is that `${{ github.event.pull_request.head.ref }}` will be empty which will be replaced by the default value.

<span style="font-size:1.8em;">&#9432;</span> 
Note that `${{ github.event.pull_request.head.sha }}` won't work as checkout ref. It will lead to an error message like `fatal: You are not currently on a branch.` when the automatically changed files are pushed.

### Detect commit message and author using git log

The command [git log [15]][GitLog] shows the commit history. The output can be limited to only display the last commit and formatted to only show the commit message or author.

```bash
git --no-pager log -1 --pretty=format:'%s' # prints the message of the last commit
git --no-pager log -2 --pretty=format:'%an' # prints the author name of the last commit
```

<span style="font-size:1.8em;">&#9432;</span> 
Note that the [git command option --no-pager [20]][GitCommand] is used to print the result directly to the console. This is not necessary when the command is used within an echo command.

The following workflow steps show how to put commit message and author into environment variables in a linux or unix shell:

```yml
- name: Set environment variable "commit-message"
  run: echo "commit-message=$(git log -1 --pretty=format:'%s')" >> $GITHUB_ENV
- name: Set environment variable "commit-author"
  run: echo "commit-author=$(git log -1 --pretty=format:'%an')" >> $GITHUB_ENV
```

Add optional display steps for debugging and maintenance purposes as you like:

```yml
- name: Display environment variable "commit-message"
  run: echo "commit-message=${{ env.commit-message }}"
- name: Display environment variable "commit-author"
  run: echo "commit-author=${{ env.commit-author }}"
```

<span style="font-size:1.8em;">&#128214;</span> 
Further reading: [GIT pretty format [16]][GitPrettyFormats]

### Example 4

```yml
name: Continuous Integration
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    env: 
      CI_COMMIT_MESSAGE: Continuous Integration Build Artifacts
      CI_COMMIT_AUTHOR: ${{ github.event.repository.name }} Continuous Integration
    steps:

    # Checkout that works with "push" and "pull_request" trigger event
    - uses: actions/checkout@v2
      with:
        ref: ${{ github.event.pull_request.head.ref }}
        token: ${{ secrets.WORKFLOW_GIT_ACCESS_TOKEN }}

    # Set environment variables based on the last commit
    - name: Set environment variable "commit-message"
      run: echo "commit-message=$(git log -1 --pretty=format:'%s')" >> $GITHUB_ENV
    - name: Display environment variable "commit-message"
      run: echo "commit-message=${{ env.commit-message }}"

    - name: Set environment variable "commit-author"
      run: echo "commit-author=$(git log -1 --pretty=format:'%an')" >> $GITHUB_ENV
    - name: Display environment variable "commit-author"
      run: echo "commit-author=${{ env.commit-author }}"

    - name: Set environment variable "is-auto-commit"
      if: env.commit-message == env.CI_COMMIT_MESSAGE && env.commit-author == env.CI_COMMIT_AUTHOR
      run: echo "is-auto-commit=true" >> $GITHUB_ENV
    - name: Display environment variable "is-auto-commit"
      run: echo "is-auto-commit=${{ env.is-auto-commit }}"

    # Build
    - uses: actions/setup-node@v2
      if: env.is-auto-commit == false
      with:
        node-version: '12'
    - name: (Main) Install nodes packages
      if: env.is-auto-commit == false
      run: npm ci
    - name: (Main) Build package (lint, test, doc, build, package)
      if: env.is-auto-commit == false
      run: npm run package
    
    # Commit generated and ^d files
    - name: Display event name 
      run: echo "github.event_name=${{ github.event_name }}"
    - name: Commit build artifacts (dist, devdist, docs, coverage)
      # Don't run again on already pushed auto commit. Don't run on pull request events.
      if: env.is-auto-commit == false && github.event_name != 'pull_request'
      run: |
        git config --global user.name "${{ env.CI_COMMIT_AUTHOR }}"
        git config --global user.email "username@users.noreply.github.com"
        git pull
        git commit -a -m "${{ env.CI_COMMIT_MESSAGE }}"
        git push
```

## Conclusion
 
The easiest solution shown in [example 1](#example-1) pushes all changed files automatically when 
a pull request is merged or a commit is directly pushed into the main branch. This will most likely 
cover common use cases like JavaScript distribution bundles, static site generation and test coverage reports. 

[Example 2](#example-2) shows how to overcome a protected main branch using a personal access token.

Preventing the workflow from being triggered by itself is essential to avoid an endless loop.
This isn't an issue when using the build-in `GITHUB_TOKEN`. As soon as a personal access token is used,
it needs to be assured, that the auto commit is only executed once. 

Whilst [example 2](#example-2) skips the whole workflow for the automatically created commit using [paths-ignore [11]][GitHubActionsIgnoringPaths], [example 3](#example-3) shows how this can be configured for every single step detecting the auto commit by its message and author.

To be able to review an automatically pushed commit it is required that it is executed within a pull request.
For this some extra effort is needed as shown in [example 4](#example-4).

## Examples

- [1](#example-1) The easiest solution to auto commit on push (not within pull request)
- [2](#example-2) Personal access token variation of [example 1](#example-1) that works with a protected main branch
- [3](#example-3) Decide for each step if it should run for the auto commit
- [4](#example-4) Complex solution that executes the auto commit within a pull request e.g. to review it

{% endraw %}

<br>

----

## Updates

- 2022-08-01: [Refine post to use git pull before auto commit](https://github.com/JohT/johtizen/pull/22)

## References
- [[1] Continuous Integration][ContinuousIntegration]  
https://martinfowler.com/articles/continuousIntegration.html
- [[2] GitHub][GitHub]  
https://github.com
- [[3] GitHub Actions][GitHubActions]  
https://docs.github.com/en/actions/learn-github-actions
- [[4] Dump out all GitHub Actions context][DumpContext]  
https://til.simonwillison.net/github-actions/dump-context
- [[5] GitHub Actions Logging][GitHubActionsLogging]  
https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging
- [[6] Renaming the default branch][GitHubRenamingDefaultBranch]  
https://github.com/github/renaming
- [[7] Events that trigger workflows][WorkflowEvents]  
https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows
- [[8] GitHub Actions: A deep dive into "pull_request"][GitHubActionsPullRequestDeepDive]  
https://frontside.com/blog/2020-05-26-github-actions-pull_request
- [[10] Push to origin from GitHub action][PushOriginFromGitHubAction]  
https://stackoverflow.com/questions/57921401/push-to-origin-from-github-action
- [[11] GitHub Actions Ignoring Paths][GitHubActionsIgnoringPaths]  
https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#example-ignoring-paths
- [[12] GitHub Actions Push Event Payload][GitHubActionsPushEventPayload]  
https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#push
- [[13] GitHub Actions Pull Request Event Payload][GitHubActionsPullRequestEventPayload]  
https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#pull_request
- [[14] GitHub Actions Checkout][GitHubActionsCheckout]  
https://github.com/actions/checkout
- [[15] GIT log][GitLog]  
https://git-scm.com/docs/git-log
- [[16] GIT pretty formats][GitPrettyFormats]  
https://git-scm.com/docs/pretty-formats
- [[17] Github Actions workflow for merged/closed PRs][MergedClosedPullRequestEvents]  
https://shipit.dev/posts/trigger-github-actions-on-pr-close.html
- [[18] git-auto-commit Action][git-auto-commit]  
https://github.com/stefanzweifel/git-auto-commit-action
- [[19] Creating a personal access token][GitHubPersonalAcessToken]   
https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
- [[20] Git Command][GitCommand]   
https://git-scm.com/docs/git
- [[21] add-and-commit Action][add-and-commit]
https://github.com/EndBug/add-and-commit
- [[22] data-restructor-js][data-restructor-js]
https://github.com/JohT/data-restructor-js
- [[23] How to push protected branches][HowToPushProtectedBranches]
https://github.community/t/how-to-push-to-protected-branches-in-a-github-action/16101/33
- [[24] Managing a branch protection rule][BranchProtectionRule]
https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule


[ContinuousIntegration]: https://martinfowler.com/articles/continuousIntegration.html
[GitHub]: https://github.com
[GitHubActions]: https://docs.github.com/en/actions/learn-github-actions
[DumpContext]: https://til.simonwillison.net/github-actions/dump-context
[GitHubActionsLogging]: https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging
[GitHubRenamingDefaultBranch]: https://github.com/github/renaming
[WorkflowEvents]: https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows
[WorkflowFromWorkflow]: https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#triggering-a-workflow-from-a-workflow
[GitHubActionsPullRequestDeepDive]: https://frontside.com/blog/2020-05-26-github-actions-pull_request
[GitHubActionsPullRequestDeepDiveCheckout]: https://frontside.com/blog/2020-05-26-github-actions-pull_request/#how-does-pull_request-affect-actionscheckout
[PushOriginFromGitHubAction]: https://stackoverflow.com/questions/57921401/push-to-origin-from-github-action
[GitHubActionsIgnoringPaths]: https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#example-ignoring-paths
[GitHubActionsPushEventPayload]: https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#push
[GitHubActionsPullRequestEventPayload]: https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#pull_request
[GitHubActionsCheckout]: https://github.com/actions/checkout
[GitLog]: https://git-scm.com/docs/git-log
[GitPrettyFormats]: https://git-scm.com/docs/pretty-formats
[MergedClosedPullRequestEvents]: https://shipit.dev/posts/trigger-github-actions-on-pr-close.html
[git-auto-commit]: https://github.com/stefanzweifel/git-auto-commit-action
[GitHubPersonalAcessToken]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
[GitCommand]: https://git-scm.com/docs/git
[add-and-commit]: https://github.com/EndBug/add-and-commit
[data-restructor-js]: https://github.com/JohT/data-restructor-js
[HowToPushProtectedBranches]: https://github.community/t/how-to-push-to-protected-branches-in-a-github-action/16101/33
[push-protected]: https://github.com/CasperWA/push-protected
[BranchProtectionRule]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule