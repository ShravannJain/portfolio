---
title: "Created my first first GitHub Actions workflow file."
date: 2024-07-13
description: "Kicking off the writing feed."
tags: ["github-actions", "ci-cd", "github"]
---
My First GitHub Actions Workflow

Today, I created my first GitHub Actions workflow file, and it was an exciting experience! I’m learning about GitHub Actions and how it helps with CI/CD pipelines specifically, continuous integration and continuous deployment.

I made a simple workflow triggered by creating a pull request. I wrote a YAML file where I:

runs-on: Ubuntu-latest: Uses latest Ubuntu version. gh pr comment: Comment on PR with $PR_URL. env:

GITHUB_TOKEN: Authenticates workflow. PR_URL: PR URL from GitHub event. This hands on exercise taught me about workflows, triggers, jobs, and steps. It was a fun way to learn something new, and I can’t wait to explore more features of GitHub Actions!