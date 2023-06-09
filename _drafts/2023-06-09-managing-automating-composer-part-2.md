---
title: Managing and Automating You Composer Packages - Part 2
description: Trigger composer repository updates from other repositories and deploying repository secrets.
date: 2023-06-09 08:00:00 -0400
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

### Adding an Action Secret
