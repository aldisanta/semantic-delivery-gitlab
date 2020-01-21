# @hutson/semantic-delivery-gitlab

> Automatically generate a deliverable, along with a corresponding git tag, for GitLab-hosted source code.

When you create a new deliverable for your GitLab project, you probably do several of the following:

- Get a list of all commits to the project that has not been released previously.
- Determine the appropriate [semantic version](http://semver.org/) to use for the deliverable.
- Generate a git tag for the repository on GitLab with that version.
- Publish a [GitLab release page](https://docs.gitlab.com/ce/workflow/releases.html) with a list of changes in that version.
- Inform people subscribed to GitLab issues, or merge requests, about the deliverable.

By automating these steps `@hutson/semantic-delivery-gitlab` alleviates some of the overhead in managing a project, allowing you to quickly and consistently deploy enhancements that provide value to your consumers.

## Table of Contents
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [@hutson/semantic-delivery-gitlab](#hutsonsemantic-delivery-gitlab)
  - [Table of Contents](#Table-of-Contents)
  - [Features](#Features)
  - [Installation](#Installation)
  - [Usage](#Usage)
    - [CLI Tool](#CLI-Tool)
    - [Programmatically](#Programmatically)
    - [CLI Options](#CLI-Options)
      - [[--help]](#help)
      - [[--token ${GITLAB_TOKEN}]](#token-GITLABTOKEN)
      - [[--preset]](#preset)
      - [[--dry-run]](#dry-run)
    - [How the Deliverable is Generated](#How-the-Deliverable-is-Generated)
    - [Required GitLab CE/EE Edition](#Required-GitLab-CEEE-Edition)
    - [Continuous Integration and Delivery (CID) Setup](#Continuous-Integration-and-Delivery-CID-Setup)
  - [Release Strategies](#Release-Strategies)
    - [On Every Push To A Repository With New Commits](#On-Every-Push-To-A-Repository-With-New-Commits)
    - [On A Schedule](#On-A-Schedule)
  - [How to Publish Project to an npm Registry](#How-to-Publish-Project-to-an-npm-Registry)
  - [Version Selection](#Version-Selection)
  - [Common Issues](#Common-Issues)
    - [GitLabError: 404 Project Not Found (404)](#GitLabError-404-Project-Not-Found-404)
  - [Security Disclosure Policy](#Security-Disclosure-Policy)
  - [Professional Support](#Professional-Support)
  - [Debugging](#Debugging)
  - [Node Support Policy](#Node-Support-Policy)
  - [Contributing](#Contributing)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Features

- [x] Get a list of unreleased commits using [git-raw-commits](https://www.npmjs.com/package/git-raw-commits).
- [x] Detect commit message convention used by a project with [conventional-commits-detector](https://www.npmjs.com/package/conventional-commits-detector).
- [x] Determine appropriate version for the deliverable, or whether to generate a deliverable at all, with [conventional-recommended-bump](https://www.npmjs.com/package/conventional-recommended-bump).
- [x] Publish a [GitLab release](http://docs.gitlab.com/ce/workflow/releases.html) using [conventional-gitlab-releaser](https://www.npmjs.com/package/conventional-gitlab-releaser).
- [x] Create an annotated [git tag](https://git-scm.com/book/en/v2/Git-Basics-Tagging) on GitLab.
- [x] Post a comment to GitLab issues closed by changes included in a deliverable.

## Installation

To use the `@hutson/semantic-delivery-gitlab` tool in your project's delivery process you may either install the package or use our [Docker image](https://hub.docker.com/repository/docker/hutson/semantic-delivery-gitlab).

To install the package please run the following command:

```bash
yarn add --dev @hutson/semantic-delivery-gitlab
```

For Docker, please see the [_Continuous Integration and Delivery (CID) Setup_](#Continuous-Integration-and-Delivery-CID-Setup) section below

## Usage

There are two ways to use `@hutson/semantic-delivery-gitlab`, either as a CLI tool or programmatically.

To learn how `@hutson/semantic-delivery-gitlab` can be used to automatically generate deliverables for your project on new changes to your repository, please see the [_Continuous Integration and Delivery (CID) Setup_](#Continuous-Integration-and-Delivery-CID-Setup) section below.

### CLI Tool

Call `@hutson/semantic-delivery-gitlab` from within your project's top folder:

```bash
$(yarn bin)/semantic-delivery-gitlab
```

### Programmatically

```javascript
const semanticDeliveryGitlab = require(`@hutson/semantic-delivery-gitlab`);

const config = {
  /**
  * Options are the camelCase form of their respective CLI flag.
  */

  /**
  * For example, the `--dry-run` option can be set like so:
  */
  dryRun: true,

  /**
  * The `--preset` option can be set like so:
  */
  preset: `angular`,

  /**
   * The `--token` option can be set like so:
   */
  token: `TOKEN`,
};

try {
  const result = semanticDeliveryGitlab(config);
  /* Package successfully delivered to GitLab. */
} catch (error) {
  /* Do any exception handling here. */
}
```

### CLI Options

The following CLI options are supported and can be passed to `@hutson/semantic-delivery-gitlab`:

#### [--help]

Help on using the CLI.

```bash
$(yarn bin)/semantic-delivery-gitlab --help
```

#### [--token ${GITLAB_TOKEN}]

For `@hutson/semantic-delivery-gitlab` to publish a release to GitLab you must pass a [GitLab Personal Access Token](https://gitlab.com/profile/personal_access_tokens):

```bash
$(yarn bin)/semantic-delivery-gitlab --token ${GITLAB_TOKEN}
```

The personal access token must have the following scope set:

- `api`

The personal access token must be setup on an account for a member of the project you're wanting to automatically release.

> GitLab permissions are documented on the [GitLab Permissions](http://docs.gitlab.com/ce/user/permissions.html) site.

#### [--preset]

Version selection and the format of the release notes are configured through a `conventional-changelog` preset package.

For example, to use the [ESLint release conventions](https://www.npmjs.com/package/conventional-changelog-eslint), first install their preset package along-side `@hutson/semantic-delivery-gitlab`:

```bash
yarn add --dev conventional-changelog-eslint
```

Then pass the name, minus `conventional-changelog-`, of the preset package to `@hutson/semantic-delivery-gitlab`:

```bash
$(yarn bin)/semantic-delivery-gitlab --preset eslint
```

If a preset is not provided `@hutson/semantic-delivery-gitlab` will use [`conventional-changelog-angular`](https://www.npmjs.com/package/conventional-changelog-angular).

#### [--dry-run]

To verify your environment is setup correctly, but not generate a deliverable, you can conduct a dry run of ``@hutson/semantic-delivery-gitlab`:

```bash
$(yarn bin)/semantic-delivery-gitlab --dry-run
```

This work well with the [_Debugging_](#Debugging) section below.

### How the Deliverable is Generated

The first step of `@hutson/semantic-delivery-gitlab` is to get a list of commits made to your project after the latest semantic version tag. If no commits are found, which typically happens if the latest commit in your project is pointed to by a semantic version tag, then `@hutson/semantic-delivery-gitlab` will exit cleanly and indicate no changes can be released. This ensures you can run the release process multiple times and only release new versions if there are unreleased commits. If unreleased commits are available, `@hutson/semantic-delivery-gitlab` will proceed to the next step.

Unless you have configured a preset, the convention used by your project will be determined by `conventional-commits-detector`. Once we have determined your convention, we pass along the preset package associated with that convention to `conventional-recommended-bump`. `conventional-recommended-bump` will determine the appropriate version to release. For more information on how versions are determined, please see the [_Version Selection_](#Version-Selection) section below.

Once a recommendation has been provided by `conventional-recommended-bump`, we generate a new [GitLab release page](http://docs.gitlab.com/ce/workflow/releases.html), with a list of all the changes made since the last version. Creating a GitLab release also creates an annotated git tag (You can retrieve the annotated tag using `git fetch`).

Lastly, a comment will be posted to every issue that is referenced in a released commit, informing subscribers to that issue of the recent release and version number.

### Required GitLab CE/EE Edition

Version [9.0](https://about.gitlab.com/2017/03/22/gitlab-9-0-released/#api-v4-ce-ees-eep), or higher, of GitLab CE/EE, is required for `@hutson/semantic-delivery-gitlab`.

Core features used:

- [GitLab release page](http://docs.gitlab.com/ce/workflow/releases.html)
- [API v4](https://docs.gitlab.com/ce/api/README.html)

> This only applies to you if you're running your own instance of GitLab. GitLab.com is always the latest version of the GitLab application.

### Continuous Integration and Delivery (CID) Setup

Since `@hutson/semantic-delivery-gitlab` relies only on a GitLab token, and a package published to the public npm registry, `@hutson/semantic-delivery-gitlab` should work on any GitLab platform.

Configuring a GitLab CI job is facilitated through a `.gitlab-ci.yml` configuration file. To publish changes using `@hutson/semantic-delivery-gitlab` you will need to create a dedicated build stage that executes only after all other builds and tests have completed successfully.

That can be done with GitLab CI by creating a dedicated `deliver` stage and adding it as the last item under `stages`. Next, create a job called `deliver` and add it to the `deliver` stage. Within `deliver` call `semantic-delivery-gitlab`.

You can see a snippet of a the `.gitlab-ci.yml` file below with this setup:

```yaml
stages:
  - build
  - test
  - deliver

deliver:
  before_script:
    - yarn install --frozen-lockfile
  image: node:8
  only:
    - master@<GROUP>/<PROJECT>
  script:
    - $(yarn bin)/semantic-delivery-gitlab --token ${GITLAB_TOKEN}
  stage: deliver
```

An alternative is to use our [Docker image](https://hub.docker.com/repository/docker/hutson/semantic-delivery-gitlab):

```yaml
stages:
  - build
  - test
  - deliver

deliver:
  image:
    name: hutson/semantic-delivery-gitlab:9.1.0
    entrypoint: [""]
  only:
    - master
  script:
    - semantic-delivery-gitlab --token ${GITLAB_TOKEN}
  stage: deliver
```

Full documentation for GitLab CI is available on the [GitLab CI](http://docs.gitlab.com/ce/ci/yaml/README.html) site.

In addition to publishing a new release on every new commit, which is the strategy shown above, you may use any number of other strategies, such as publishing a release on a given schedule. Please see the [_Release Strategies_](#Release-Strategies) section below for a few such alternative approaches.

## Release Strategies

You can employ many different release strategies using an automated tool such as `semantic-release-gitlab`.

Below we document a few release strategies. Please don't consider this list exhaustive. There are likely many other ways to decide when it's best to generate a new release for your project.

### On Every Push To A Repository With New Commits

Publishing a new release on every push to a repository with new commits is the approach taken by this project. If you take this approach, you can push a single commit, leading to a release for that one change, or you can create multiple commits and push them all at once, leading to a single release containing all those changes.

An example our setup for GitLab CI can be seen in the [_Continuous Integration and Delivery (CID) Setup_](#Continuous-Integration-and-Delivery-CID-Setup) section above.

### On A Schedule

You may also release your changes on a schedule. For example, using a CI platform like [Jenkins CI](https://jenkins.io/), you can create and configure a job to run on a given schedule, such as once every two weeks, and, as part of a _Post Build Action_, run the release tool.

Other CI platforms besides Jenkins also allow you to run a particular action on a given schedule, allowing you to schedule releases as you could with Jenkins CI.

With this strategy all commits that have accumulated in your repository since the last scheduled job run will be incorporated into a single new release. Because this release tool uses a `conventional-recommended-bump`, which recommends an appropriate new version based on all commits targeted for release, you can be assured that each scheduled release will use a version appropriate for the changes accumulated in that release.

## How to Publish Project to an npm Registry

Once `@hutson/semantic-delivery-gitlab` has created a release on GitLab, the next step for an `npm` package is to publish that package to an `npm`-compatible registry. To publish your project to an `npm`-compatible registry, please use [npm-publish-git-tag](https://www.npmjs.com/package/npm-publish-git-tag).

## Version Selection

As noted earlier `@hutson/semantic-delivery-gitlab` uses [`conventional-recommended-bump`](https://www.npmjs.com/package/conventional-recommended-bump) to determine if a deliverable should be generated and whether that should be a new `major`, `minor`, or `patch` version.

The process involves `@hutson/semantic-delivery-gitlab` passing the list of all unreleased commits, along with your project's commit message convention, to `conventional-recommended-bump`. `conventional-recommended-bump` will either report that no new deliverable is recommended, or it will recommend a new `major`, `minor`, or `patch` version.

If `conventional-recommended-bump` indicates that no new deliverable should be made, `@hutson/semantic-delivery-gitlab` will **not** generate a new version of your project.

If a deliverable is recommended, and no previous version exists, we will always set the first version to `1.0.0`.

If a previous version exists, we take that version and increment it according to the recommendation using the default behavior of the [`inc`](https://www.npmjs.com/package/semver#functions) function provided by the [`semver`](https://www.npmjs.com/package/semver) package.

## Common Issues

A collection of common issues encountered while using `@hutson/semantic-delivery-gitlab`.

### GitLabError: 404 Project Not Found (404)

In some instances you may see the following error after running `@hutson/semantic-delivery-gitlab`:

```bash
semantic-delivery-gitlab failed for the following reason - GitLabError: 404 Project Not Found (404)
```

The project, or at least the project URL used by `@hutson/semantic-delivery-gitlab`, may not be valid. Please make sure the [repository field](https://docs.npmjs.com/files/package.json#repository) in your `package.json` is correct. If it is correct, please consider running `@hutson/semantic-delivery-gitlab` in debug mode, following the [_Debugging_](#Debugging) section below, to see what URL is being used to interact with GitLab.

It may also be that a token was not provided either on the command line using `--token` or as a configuration property.

## Security Disclosure Policy

To report a security vulnerability in this package, or one of its dependencies, please use the [Tidelift security contact](https://tidelift.com/security) page. Tidelift will coordinate the process to address the vulnerability and disclose the incident to our users.

## Enterprise Support

Available as part of the Tidelift Subscription.

The maintainers of `@hutson/semantic-delivery-gitlab` and thousands of other packages are working with Tidelift to deliver commercial support and maintenance for the open source dependencies you use to build your applications. Save time, reduce risk, and improve code health, while paying the maintainers of the exact dependencies you use. [Learn more.](https://tidelift.com/subscription/pkg/npm-hutson-semantic-delivery-gitlab?utm_source=npm-hutson-semantic-delivery-gitlab&utm_medium=referral&utm_campaign=enterprise)

## Debugging

To assist users of `@hutson/semantic-delivery-gitlab` with debugging the behavior of this module we use the [debug](https://www.npmjs.com/package/debug) utility package to print information about the release process to the console. To enable debug message printing, the environment variable `DEBUG`, which is the variable used by the `debug` package, must be set to a value configured by the package containing the debug messages to be printed.

To print debug messages on a UNIX system set the environment variable `DEBUG` with the name of this package before executing `semantic-delivery-gitlab`:

```bash
DEBUG=semantic-delivery-gitlab semantic-delivery-gitlab
```

On the Windows command line you may do:

```bash
set DEBUG=semantic-delivery-gitlab
semantic-delivery-gitlab
```

## Node Support Policy

We only support [Long-Term Support](https://github.com/nodejs/LTS) versions of Node.

We specifically limit our support to LTS versions of Node, not because this package won't work on other versions, but because we have a limited amount of time, and supporting LTS offers the greatest return on that investment.

This package may work correctly on newer versions of Node. It may even be possible to use this package on older versions of Node, though that's more unlikely as we'll make every effort to take advantage of features available in the oldest LTS version we support.

As each Node LTS version reaches its end-of-life we will remove that version from the `node` `engines` property of our package's `package.json` file. Removing a Node version is considered a breaking change and will entail the publishing of a new major version of this package. We will not accept any requests to support an end-of-life version of Node. Any merge requests or issues supporting an end-of-life version of Node will be closed.

We will accept code that allows this package to run on newer, non-LTS, versions of Node. Furthermore, we will attempt to ensure our own changes work on the latest version of Node. To help in that commitment, our continuous integration setup runs against all LTS versions of Node, in addition to, the most recent Node release; called _current_.

JavaScript package managers should allow you to install this package with any version of Node, with, at most, a warning if your version of Node does not fall within the range specified by our `node` `engines` property. If you encounter issues installing this package, please report the issue to your package manager.

## Contributing

Please read our [contributing guide](https://github.com/hyper-expanse/semantic-delivery-gitlab/blob/master/contributing.md) to see how you may contribute to this project.
