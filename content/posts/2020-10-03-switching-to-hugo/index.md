---
title: "Switching to Hugo"
author: "Matt Rathbun"
tags:
  - hugo
  - github-actions
  - ssg
slug: "switching-to-hugo"
---

Welp, I switched this website from [Gatsby](https://www.gatsbyjs.com/) to [Hugo](https://gohugo.io/).

This reminds me of my distro-hoppin' days, when I used to spend nights installing different Linux distributions and then meticulously configuring them to my liking. "Oh man, I can get _so_ much done with this distro!", I'd say. I spent so mush time trying to find that perfect setup, that I didn't spend much time actually _using_ them to get things done.

Well, I guess I haven't changed much. After writing a single post, I decided to try something else for this website. Let's at least talk about the notable changes in a _second_ post...

<!--more-->

# Go-ing There with package.json

Hugo doesn't need a `package.json` but I couldn't bring myself to delete it. Old habits die hard I guess. I added a couple things to it to make it feel worthwhile:

* I use [yarn workspaces](https://classic.yarnpkg.com/en/docs/workspaces/) to install dependencies for the theme when I run `yarn install` from the repo's base directory. All I had to do was add this to the package.json:

    ```json
    "workspaces": [
      "themes/terminal"
    ],
    ```


* I added [this cool npm module that installs a hugo binary](https://www.npmjs.com/package/hugo-bin) to the dev dependencies. If I want to work on this website on a new computer, I only have to run `yarn install`!

# Github Actions: Rewind It Back

With Gatsby, I used [this action](https://github.com/enriikke/gatsby-gh-pages-action) to deploy it. But this time around, I felt like I had a better understanding of what was going on here. With a little help from [this article](https://medium.com/zendesk-engineering/a-github-actions-workflow-to-generate-publish-your-hugo-website-f36375e56cf7), I (mostly) wrote it by hand. The process is simple:

1. Checkout the website's git repository
2. Run `yarn install` to install the theme's npm dependencies
3. Run `hugo` to build the website
4. Publish the contents of the `public` directory (where `hugo` builds to) to the `gh-pages` branch.

Those `package.json` changes I mentioned earlier came in handy here. Here's what it looks like:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0

      - name: Use Node.js
        uses: actions/setup-node@v1
        with:
          node-version: 12.x

      # This also installs Hugo since it's in my dependencies!
      - name: Install dependencies
        run: yarn install

      - name: Build
        run: yarn build

      - name: Create CNAME file
        run: cp CNAME public/CNAME

      - name: Publish
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          user_name: 'github-actions[bot]'
          user_email: 'github-actions[bot]@users.noreply.github.com'
          commit_message: ${{ github.event.head_commit.message }}
```

# Using Timestamps in Folder Names

I really like the idea of prefixing post names with `YYYY-MM-DD` so they're listed in date order when listed alphabetically. Like this:

```
$ ls content/posts
2020-08-02-creating-this-website/
2020-10-03-switching-to-hugo/
```

Hugo has a way to get the creation date from those timestamps! See the `:filename` date handler [here](https://gohugo.io/getting-started/configuration/#configure-dates). In my `config.toml`, I have:

```toml
[frontmatter]
date = [":filename", ":default"]
```

You can also get the date that it was last updated from git by enabling [gitinfo](https://gohugo.io/variables/git/):

```toml
enableGitInfo = true
```

I don't think I've ever been so excited about creation and updated dates before! These changes give me the file structure I want, keep the creation dates in a single place, and make the updated dates entirely automatic! 💃

#### Remove Timestamp from Title

At this point, if you run

```bash
$ hugo new posts/2020-10-03-switching-to-hugo
```

It gets the creation date from the folder name as expected, but the new post has the title "2020 10 03 Switching To Hugo". How do I get that timestamp out of there? After plenty of trial and error, I finally came up with this template for the post archetype:

```
title: "{{ replace ( replaceRE "^\\d+-\\d+-\\d+-" "" .Name 1 ) "-" " " | title }}"
```

That template removes the timestamp, then replaces hyphens with spaces, then capitalizes the first letters of each word. Note that you can nest templates with _parentheses_.

Now when I run the same command, the title is "Switching To Hugo". Ahh much better.

# That's All Folks

That's it for now, we'll see how Hugo works out. If I switch again, I'll be sure to post some updates!
