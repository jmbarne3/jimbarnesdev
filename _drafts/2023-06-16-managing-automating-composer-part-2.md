---
title: Managing and Automating You Composer Packages - Part 2
description: Trigger composer repository updates from other repositories and deploying repository secrets.
date: 2023-06-16 08:00:00 -0400
categories: [DevOps, Package Management]
tags: [php, composer, packagist, automation, github pages]
img_path: /assets/images/posts/managing-automating-composer-part-2/
image:
  path: hero.jpg
  alt:
status: draft
---

In the first two parts of this series, we setup a private composer repository, got it up and running on GitHub Pages, and laid the foundation for automatic updates using GitHub Workflows. In this article we'll setup automation on our package repositories so that they report to the composer repository whenever they've been tagged, triggering the docs for that package to update.

## The Workflow

Our end goal is to trigger a partial rebuild of the composer repository anytime one a new release of one of our packages is created or updated. We could trigger rebuild on a push to any branch or a particular branch, but for the sake of demonstration we'll stick to releases in this article.

![Diagram showing the three primary steps to updates: triggering the partial build of the composer repository, the partial build completing and committing to the gh-pages branch and the deployment of the rebuild pages.](workflow-diagram.jpg)
_Diagram showing the three primary steps to updates: triggering the partial build of the composer repository, the partial build completing and committing to the main branch and the built-in deployment action for GitHub Pages._

Every partial update will require 3 steps to complete:

1. The repository that has an update will send an HTTP request to the composer repository asking it to do a partial rebuild of a particular package.
2. The composer repository will execute the partial-build workflow rebuilding the files specific to the package specified and commit the changes to the main branch.
3. Because we're hosting our files from a branch and within a particular directory, GitHub will automatically deploy the newly generated files to GitHub pages.

## Personal Access Tokens and Action Secrets

One major problem you will pretty quickly run into with this process is that it's not possible to trigger a workflow from another repository without token with appropriate permissions to do so. We'll manually deal with this problem now, but will look at an automated solution towards the end of the article.

### Generating a Personal Access Token

Generating personal access tokens (PAT) is [very well documented by GitHub within their documentation](https://docs.github.com/en/enterprise-server@3.4/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens), but there are a few things to keep in mind when creating one.

> If you're setting up this automation for an organization, you will likely need to agree to a particular set of terms of use that GitHub will prompt you with. this probably isn't an issue, but you may want to check with your organization's legal counsel about it before accepting.
>
> Additionally, you may want to consider creating the PAT using a service account that isn't tied to a particular user to prevent loosing access to the token if the employee leaves.
{: .prompt-warning }

A good rule of thumb is to create Personal Access Tokens with as few permissions as necessary in order to get the job done. In this case, our token only needs to be able to trigger workflows throughout our user or org.

The PAT will need to have need to have **read access to metadata** and **read and write access to actions and workflows**. It does not need the ability to commit code, as the workflows themselves do the actually committing to the repository. This PAT simply causes them to run.

### Adding an Action Secret

> If you are using GitHub Enterprise, instead of configuring this as a repository secret on individual repositories, you should create an organization secret. The rest of the steps in this guide will work exactly the same, as you can access org secrets in the `secrets` variable within workflows, but you will only have to update the secret in one place instead of on every single repository.
>
> If you do not have GitHub Enterprise, no need to worry - we'll be going over a [method to distribute secrets to the repository below](#automating-the-automation).
{: .prompt-info }

Now that we've created our Personal Access Token, we need to add it as an Action Secret so that we can reference it in our workflows. If you go into the Settings page of your repository and open the "Secrets and variables" tab in the left navigation, you'll see an option for "Actions".

![The Actions variable and secrets can be found under Secrets and variables](actions.jpg)

Within this screen you should see two tabs, each with two lists of variables:

- Secrets
  - Environment Secrets
  - Repository Secrets
- Variables
  - Environment Variables
  - Repository Variables

In this case we want to create a new repository secret, which you can do by clicking the "New repository secret" button. This variable can be named whatever you like, but it's a good idea to decide on something descriptive and memorable, as you'll be creating this same variable on every other repository that needs to report updates to the composer repo. for the purposes of our examples below, I'll call this variable `COMPOSER_REPO_PAT`. For the value, you'll copy and paste in the personal access token you created in the step above. Save this and repeat for any other repositories you want to configure. Once this variables is created on your repositories, we're ready to create our workflow.

### The Notification Workflow

We want to create a workflow that will notify our composer repository whenever a package has been updated so that the repository can do a partial rebuild of just that package. We'll continue to use our fictional data from prior posts and the diagram above, and setup the workflow for the `bilbo/riddle-generator` repository.

First, we'll need to create a workflow file in the repository under `.github/workflows` and we'll call this one `notify-composer-update.yml`. We have [a bunch of different options](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) here on when we want to notify the composer repo, so I'll add in a few in the example.

```yaml
name: Notify Composer Update

on:
  push:
    branches:
      - main
      - develop
  release:
    types:
      - published
      - edited
      - deleted
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
        run: |
          curl -XPOST \
            -H "Authorization: Bearer ${{ secrets.COMPOSER_REPO_PAT }}" \
            -H "Accept: application/vnd.github.everest-preview+json" \
            -H "Content-Type: application/json" \
            https://api.github.com/repos/bilbo/composer-repo/actions/workflows/partial-build.yml/dispatches \
            --data '{"ref": "main", "inputs": {"package": "bilbo/riddle-generator"}}'
```
{: file="notify-composer-update.yml" }

In this example, we're going to trigger this workflow when either the main or develop branch is pushed to or whenever a release is created, edited or deleted. The `workflow_dispatch` trigger also allows us to run the workflow manually from the GitHub GUI at any time. When the workflow is triggered, the API endpoint that triggers a workflow is called containing the personal access token we just created along with a couple pieces of information in the POST data:

1. The branch or "ref" we want the workflow to run against, in this case `main`.
2. The input we created for the `partial-build.yml` workflow: `package`. In this case, we send it the package name of "bilbo/riddle-generator".

Commit this into the repository and trigger the workflow manually to test. You should see the notification happen in the "Actions" tab of the repository, and should also see the `partial-build` action run over in your composer repository.

## Automating the Automation

Now that we have the automation setup, there's really one last difficultly to deal with: how do we get a workflow and that PAT out to all of our repositories? Do you really have to manually create the secret and workflow file on _n_ number of repositories?

Of course not. In the same way we can run PHP scripts and commit to repositories via scripts, we can deploy secrets and write files via scripts too.

There are a number of ways to approach the problem, but for the sake of seeing what's involved, let's walk through a simple nodejs script that does the following:

1. Reads the repositories that need to have the secret and workflow file from the `satis.json` file in our repo.
2. Connect out to each repository and make sure the `secret` is created and is named correctly, in this case `COMPOSER_REPO_PAT`.
3. Check to make sure the `notify-composer-update.yaml` file exists and contains the appropriate configuration.

### Getting Setup

We're going to need to install a few dependencies in order to run this, so let's setup a node project using node 16+ and install the following packages:

```bash
npm init -y
npm install --save axios commander dotenv libsodium-wrappers octokit
```

Axios and commander are completely optional here, but I like using both and I'll be using them in the example below. Referring back to our list of goals for this script, we're going to need a few pieces of information, so let's pull those in as arguments using commander.

```javascript
const commander = require('commander');

commander
  .version('1.0.0', '-v', '--version')
  .usage('[OPTIONS]...')
  .option('-p, --composer <url>', 'The URL of the composer repository to scan for a satis.json file', null)
  .option('-o, --org <org>', 'The ORG name of the repos being updated.', 'UCF')
  .option('-k, --key <key>', 'The key to deploy to the repositories.', null)
  .option('-s, --skip-secret', 'When present, the secret key will not be updated.')
  .option('-f, --force', 'Forces workflow file to be updated.')
  .parse(process.argv);
```
{: file="index.js" }
