# GitHub Actions for CI/CD

Ansible CI/CD workflows are recommended to be configured via GitHub Actions.

In the top-level repo, create the following directory:
```
.github/workflows/
```

And within the GitHub workflows directory, create a config file with the name of your choice:
```
.github/workflows/dev_merge.yml
```

And then populate the file with the details of what actions you would like to perform! 

For example, this will check out a dev branch [uncomment the `tags` line and it will look for something tagged with a version], run pre-commits and linting, and -- assuming they all pass -- those changes would then be pushed to the main branch with a new tag.

```
on:
  push:
    branches: [dev]
    # tags:
    #  - v*

jobs:
  build:

   runs-on: ubuntu-latest

   steps:
   - uses: actions/checkout@v3

   - uses: pre-commit/action@v3.0.0

   - uses: ansible-community/ansible-lint-action@main

   - uses: stefanzweifel/git-auto-commit-action@v4
     with:
      commit_message: publish release
      branch: main
      # tagging_message: 'v1.2.3'
```

With a GitHub Action setup like this, it would trigger the corresponding rules only when a version tag is present. Or we could simply add the `git-auto-commit` action to pull from dev and push to main — along with adding a version tag.

To finish this process, the last thing we need to do is set the dev branch to be a protected branch. This will require someone to approve a PR to the `dev` branch – at this point they will be able to see the full results of, and a pass/fail for, each of the steps taken by Actions:

```
Checkout
Pre-commit
Linting
Auto-commit
```
