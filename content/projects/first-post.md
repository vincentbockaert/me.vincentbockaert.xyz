+++
title = 'How I "created" this very site'
date = 2023-12-11T19:10:50+01:00
draft = false
+++

## Introduction

As this is my first post, I figured, why not write about how I set this very thing up to begin with?

So that's exactly what I did; in the next few paragraphs you will get a step-by-step process of how you can build this too.

## Prerequisites

- [GitHub account](https://github.com/)
- [hugo extended edition](https://gohugo.io/installation/)
- [Cloudflare account](https://dash.cloudflare.com/login)
- [Git](https://git-scm.com/downloads)
- A domain managed in Cloudflare

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
baseURL: https://vincentbockaert.xyz/ # replace with your baseUrl, can be fake for now
languageCode: en-us
title: It's-a Me, Vinnie!
theme: "PaperMod"
params:
  defaultTheme: auto

  homeInfoParams:
    Title: Welcome to my personal little corner on the internet :)
    Content: Here you can find links to my "socials" as well as posts about hobby-activities, DevOps or otherwise related.
```

The above will ensure that a home-page is served by hugo, thus we can now make hugo serve our "website".

```bash
hugo server -D # -D is for draft mode, which will reload the pages as we make edits
```

## Setting up deployment

The deployment of your hugo generated static site is extremely straight forward to configure.
First off we need to ensure that our repository exists in GitHub.

This shouldn't need to be explained :D but as an example, we can use the below to create a remote for our local repo:

```bash
git remote add origin https://github.com/<your-gh-username>/<repository-name>
git branch -M main
git push -u origin main
```

Afterward, we can continue with the configuration in the cloudflare dashboard.

### Cloudflare configuration

To deploy our site as a _Cloudflare Pages_ application we need to undertake the following procedure:

1. Sign in to the [Cloudflare dashboard](https://dash.cloudflare.com/)
2. Using the webui, starting from Account Home go to _Workers & Pages_
3. Now proceed by selecting _Workers & Pages > Create application > Pages > Connect to Git_
4. Select the created repository (you will first need to authenticate with GitHub to grant Cloudflare access)
5. Configuration options to set:
   - Production branch: your main branch
   - Build command: `hugo --minify` 
     - Ensures full canonical URLs will still work after deployment, see more on the subject [here](https://developers.cloudflare.com/pages/framework-guides/deploy-a-hugo-site/#deploy-with-cloudflare-pages)
   - Build directory: public 

Once the first deployment has finished, we will get a unique `*.pages.dev` domain assigned to our application.
For every commit made to the main branch of our repository, a new build and deployment will take place.

Additionally, pull requests (PRs) can be used to view preview deployments in order to view the changes "live" (without impacting) 
the main-branch's deployed version.

#### Configure custom domain

Of course, this isn't using our fancy personalized domain, so as the final configuration change we rectify this.
On the _Workers & Pages_ overview page, we can our deployed _Page app_.

Using the title of our deployed _Pages app_ we can configure some extra details, among them the custom domains.

_See as guidance the below images, with first an overview of my deployed app_

![image showing overview of the worker pages app with the title hyperlink encircled in red](/img/projects/first-post/1.webp)

_...and finally another image which shows where to add a custom domain_

![image showing where to add the custom domain, using encircle in red button](/img/projects/first-post/2.webp)