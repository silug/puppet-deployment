# Puppet Deployment for GitLab CI

One simple solution for Puppet deployments when you are using it in conjunction with GitLab is to install a [shell runner](https://docs.gitlab.com/runner/executors/shell.html) on your Puppet server and configure it to run as a user that is allowed to run `r10k` via `sudo`.
If you have multiple servers, the solution is a bit more complicated.  As described in [GitLab issue #16474](https://gitlab.com/gitlab-org/gitlab/-/issues/16474), GitLab does not currently have a way to run a job on all runners with a specific tag.
We can work around this problem by using a script to dynamically generate a list of jobs with server-specific tags.

The idea for this solution came from [a clever comment on that issue](https://gitlab.com/gitlab-org/gitlab/-/issues/16474#note_588203659).

- [Puppet Deployment for GitLab CI](#puppet-deployment-for-gitlab-ci)
  - [Getting Started](#getting-started)
    - [Import this project](#import-this-project)
    - [Set `GITLAB_API_TOKEN`](#set-gitlab_api_token)
    - [Installing the runner](#installing-the-runner)
    - [Prerequisites](#prerequisites)
  - [Enabling for a Control Repo](#enabling-for-a-control-repo)
  - [Enabling for a Puppet Module](#enabling-for-a-puppet-module)

## Getting Started

For this to work, we need a shell runner on each Puppet server with exactly two tags: one that is common to every Puppet server, and one that is specific to the server.  We use `r10k` for the common tag, but you can use anything *as long as you change the tag in the `generate` job in [.gitlab-ci.yml](.gitlab-ci.yml)*.

Note that the server-specific tag also needs to match the description of the runner.  This lets us avoid multiple API calls to GitLab.

### Import this project

Import this project into your local GitLab instance.  You will need the relative path to this project later.

### Set `GITLAB_API_TOKEN`

As a user with permissions to the project or group where you imported this project, generate an API token with `read_api` permissions.  Add the token to the CI/CD variables for this project or group as `GITLAB_API_TOKEN`.

### Installing the runner

To start, we need a shell runner that is registered using the registration token of either this project or the group containing this project, **not** the GitLab instance.

The simplest way to configure the runner is to use the following Puppet modules:

* [puppet-gitlab_ci_runner](https://forge.puppet.com/puppet/gitlab_ci_runner)
* [saz-sudo](https://forge.puppet.com/saz/sudo)

If you are using Hiera for classification, the configuration will look like this:

```yaml
---
classes:
  - sudo
  - gitlab_ci_runner

gitlab_ci_runner::manage_docker: false
gitlab_ci_runner::concurrent: 1

gitlab_ci_runner::runner_defaults:
  # Replace this with the URL of your GitLab server.
  url: 'https://gitlab.my.domain/'
  # Replace this with your registration token.
  registration-token: >
    ENC[eyaml-encoded registration token]
  executor: shell

gitlab_ci_runner::runners:
  "%{facts.hostname}-r10k":
    description: "%{facts.hostname}-r10k"
    executor: shell
    shell: bash
    tag-list: "r10k,%{facts.hostname}-r10k"

sudo::configs:
  'gitlab-runner':
    content:
      - "gitlab-runner %{facts.hostname} = (root) NOPASSWD: /usr/bin/r10k deploy *"
      - "gitlab-runner %{facts.hostname} = (root) NOPASSWD: /opt/puppetlabs/bin/puppet generate types *"
```

### Prerequisites

The included script will try to use the `ruby` that is bundled with the `puppet-agent` package.  It *either* needs the `bundler` gem *or* the `httpclient` gem installed.  If you have a profile class for your Puppet servers, add one of the following:

```puppet
package { 'bundler':
  ensure   => installed,
  provider => puppet_gem,
}
```

***or***

```puppet
package { 'httpclient':
  ensure   => installed,
  provider => puppet_gem,
}
```

Alternatively, run `/opt/puppetlabs/puppet/bin/gem install httpclient` as root on each server.

## Enabling for a Control Repo

Add the following to your control repo's `.gitlab-ci.yml`:

```yaml
---
stages:
  - deploy

deploy_environment:
  stage: deploy
  only:
    refs:
      - branches
  except:
    changes:
      - Puppetfile
  variables:
    PUPPET_DEPLOYMENT_TYPE: environment
    PUPPET_ENVIRONMENT: $CI_COMMIT_BRANCH
  trigger:
    project: puppet/puppet-deployment


deploy_environment_modules:
  stage: deploy
  only:
    refs:
      - branches
    changes:
      - Puppetfile
  variables:
    PUPPET_DEPLOYMENT_TYPE: environment_modules
    PUPPET_ENVIRONMENT: $CI_COMMIT_BRANCH
  trigger:
    # Set this to the relative path of this repo in your GitLab.
    project: puppet/puppet-deployment
```

## Enabling for a Puppet Module

If you are using PDK, add the following to your `.sync.yml` and run `pdk update`:

```yaml
---
.gitlab-ci.yml:
  custom:
    custom_stages:
      - deploy
    custom_jobs:
      deploy:
        stage: deploy
        only:
          refs:
            - branches
        variables:
          PUPPET_DEPLOYMENT_TYPE: module
          PUPPET_ENVIRONMENT: $CI_COMMIT_BRANCH
          PUPPET_MODULE: $CI_PROJECT_NAME
          DEFAULT_BRANCH: $CI_DEFAULT_BRANCH
        trigger:
          # Set this to the relative path of this repo in your GitLab.
          project: puppet/puppet-deployment
```

If you are **not** using PDK, you can add the equivalent to your `.gitlab-ci.yml` directly:

```yaml
---
stages:
  - deploy
deploy:
  stage: deploy
  only:
    refs:
      - branches
  variables:
    PUPPET_DEPLOYMENT_TYPE: module
    PUPPET_ENVIRONMENT: $CI_COMMIT_BRANCH
    PUPPET_MODULE: $CI_PROJECT_NAME
    DEFAULT_BRANCH: $CI_DEFAULT_BRANCH
  trigger:
    # Set this to the relative path of this repo in your GitLab.
    project: puppet/puppet-deployment
```
