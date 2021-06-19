---
title: "Getting Dependabot and semantic-release to Play Nice"
author: "Matt Rathbun"
slug: "semantic-release-dependabot"
tags:
  - semantic-release
  - dependabot
---

The other day, GitHub sent me a notification that there were a couple security vulnerabilities in the dependencies of one of my personal projects, [castaway](https://github.com/mattbun/castaway). Having used Dependabot at work, I figured I'd enable it and get that vulnerability taken care of.

After I merged one of Dependabot's PRs, I was shocked (😱) to find that semantic-release hadn't made a release for it. Dependabot prefixed the commit with `build(deps):`, and semantic-release only creates releases for commits with prefixes `fix` and `feat`.

What's the point of a security update if it doesn't actually build a new image? Looks like I've got some work to do.

<!--more-->

# Approach #1 - Change Dependabot Commit Messages

You can change Dependabot's commit messages! And not only that, you can change the commit prefix depending on whether its a dev or production dependency.

I configured dependabot to use the commit prefix `fix(deps)` for production dependencies with the following in `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    commit-message:
      prefix: "fix"
      prefix-development: "build"
      include: "scope"
    allow:
      - dependency-type: "production"
```

This works! semantic-release bumps the patch version and triggers a new build when production dependencies are updated. Dev dependencies are still prefixed with `build`, which semantic-release ignores.

## But there's a catch!

Dependabot comes in a couple different flavors:

1. GitHub Dependabot - this is enabled by having a `.github/dependabot.yml` file in your repository
2. GitHub Dependabot Security Updates - this is enabled in your repository's "Security & Analysis" section

The dependabot configuration from above only affects the first one. Security updates [cannot be configured](https://github.community/t/how-to-get-dependabot-to-trigger-for-security-updates-only/117257/5). This also means that the configuration from above will enable _all_ dependency updates, not just the ones with vulnerabilities.

Now this is probably fine for plenty of projects. Keeping dependencies up-to-date is noble. If you're fine with that, you're done! No need to read the next section.

But the project I'm adding this to isn't worth all that hassle. I'm fine with making some security updates here and there, but I don't want to spend all of my time updating dependencies unnecessarily.

I guess I have to give up the ability to configure Dependabot to get what I want 🤦.

# Approach #2 - Configure semantic-release to Understand Dependabot Commits

We can't change how Dependabot Security Updates works, so we'll have to work with its default commit format: `build(deps)`.

## commit-analyzer Changes

First, we'll need to configure semantic-release to see dependabot commits as "patch"-worthy updates. That can be done by adding a `releaseRule` to the commit-analyzer configuration, like the following:

```json
"release": {
  // ...
  "plugins": [
    [
      "@semantic-release/commit-analyzer",
      {
        "preset": "conventionalcommits",
        "releaseRules": [
          {
            "type": "build",
            "scope": "deps",
            "release": "patch"
          }
        ]
      }
    ],
    // ...
  ]
}
```

Note that dev dependencies use a different scope, `deps-dev`, so those updates will continue to be ignored by semantic-release.

## release-notes-generator Changes

With just the commit-analyzer changes above, Dependabot updates will create releases, but the release notes in the GitHub release will be blank. That's not particularly helpful. To fix that, we'll need to make some configuration changes to semantic-release's release-notes-generator.

First off, you'll need to add [conventional-changelog-conventionalcommits](https://www.npmjs.com/package/conventional-changelog-conventionalcommits) to your dev dependencies, if you don't already have it:

```shell
yarn add -D conventional-changelog-conventionalcommits
```

Then configure the `release-notes-generator` plugin like this:

```json
"release": {
  // ...
  "plugins": [
    // ...
    [
      "@semantic-release/release-notes-generator",
      {
        "preset": "conventionalcommits",
        "presetConfig": {
          "types": [
            {
              "type": "feat",
              "section": "Features"
            },
            {
              "type": "fix",
              "section": "Bug Fixes"
            },
            {
              "type": "build",
              "section": "Dependencies and Other Build Updates",
              "hidden": false
            }
          ]
        }
      }
    ]
    // ...
  ]
}
```

This puts any `build` commits in a new "Dependencies and Other Build Updates" section. That `"hidden": false` is important! If you don't include it, it'll default to `true`.

Note that this change will put other `build` commits in your changelog as well, though they may not trigger a release on their own.

### Do I have to use the conventionalcommits preset?

I can't speak for the other presets, but `angular` doesn't support this as far as I can tell. The `conventionalcommits` preset has an [extensive configuration spec](https://github.com/conventional-changelog/conventional-changelog-config-spec/blob/master/versions/2.1.0/README.md) while [the angular one doesn't seem to have any options?](https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-changelog-angular#readme)

## Can I see all these changes in one place?

Yep, [see it in action here](https://github.com/mattbun/castaway/blob/bc4595ff2f52bf2f7d8b929f3c386f07472b2342/package.json#L36)
