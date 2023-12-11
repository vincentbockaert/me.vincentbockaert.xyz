+++
title = 'How I created this very page'
date = 2023-12-11T19:10:50+01:00
draft = true
+++

## Introduction

As this is my first post, I figured, why not write about how I set this very thing up to begin with?

So that's exactly what I did; in the next few paragraphs you will get a step-by-step process of how you can build this too.

## Pre-requisites

- [GitHub account](https://github.com/)
- [hugo extended edition](https://gohugo.io/installation/)
- [Cloudflare account](https://dash.cloudflare.com/login)
- [Git](https://git-scm.com/downloads)

*Note, a GitLab or Bitbucket account should also work but that's out-of-scope.*

## Setting up the local environment

To set up the local environment we must perform the following steps:

1. Create a hugo directory (initialize as git repository)
    - We use the yaml format for configuration because that's more readable when compared to TOML (in my opinion)
2. Clone the theme used as a submodule into the project

In other words, we need to execute the following commands:

```bash
hugo new site me.vincentbockaert.xyz --format yaml
cd me.vincentbockaert.xyz
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod 
```

Now with that out of the way we can verify that our site works by modifying the `hugo.yaml` file adding the following:

```yaml
baseURL: https://me.vincentbockaert.xyz/ # replace with your baseUrl, can be fake for now
languageCode: en-us
title: It's-a Me, Vinnie!
theme: "PaperMod"
params:
  defaultTheme: auto

  homeInfoParams:
    Title: Welcome to my personal little corner on the internet :)
    Content: Here you can find links to my "socials" as well as posts about hobby-activities, either IT or otherwise related.
```

The above will ensure that a home-page is served by hugo, thus we can now make hugo serve our "website".

```bash
hugo server -D # -D is for draft mode, which will reload the pages as we make edits
```

## 