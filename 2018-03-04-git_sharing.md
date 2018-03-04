---
layout:  post
title: "Sharing Git Commits with Your Friends Made Easy"
comments:  true
published:  true
author: "Zach Burchill"
date: 2018-03-04 10:00:00
permalink: /git_sharing/
categories: [git,"git commit",GitHub,tip,short]
output:
  html_document:
    mathjax:  default
    fig_caption:  true
---



Have you ever wanted to make a small pull request to improve an open-source project that you have a heavily modified version of?  For example, say you have a personal version of a repo that you've changed a bunch with a particular aspect you think the main project would find useful, but you don't want to make them pull all your custom code?

I've run into this type of problem a bunch times, so I'm making a really short post on how to make a pull request to a project for just a few specific commits.

<!--more-->

## Background

One would _almost_ think that this "shared" blog is just my brother Andrew mooching off my work to make quality content, given that he has yet to make a single post. One would only think so, huh?

Anyway, I just made a post that required me to [add new CSS to the site]({{ site.baseurl }}{% post_url 2018-03-03-myoozam_survey %}), and if it was to be displayed on Andrew's GitHub Pages blog correctly, I needed a way to get him that code. Because his website is a forked version of mine, the new CSS I made should work perfectly on his site if it were implemented.

Now, I _could_ just email him the new files and the instructions of what to change, but that method is time-consuming and error-prone. Instead, I can use the magic of git and GitHub's pull requests. In this tutorial, I'll tell you how to do just that.[^1]

## Tutorial

This quick tutorial assumes a basic level of knowledge on how to use git. You should already have a repo working and know how to `pull` and `push`, etc. Let's imagine that you are either trying to share some select few changes with a project you forked, or a project that forked you.

Imagine that you have 500 commits in your repo that you don't want shared and 5 recent commits that you do.

### Step 0: The plan

In a nutshell, what we're going to be doing is loading the other project's repo into a branch of your project, transfering those 5 specific commits you want shared onto that new branch, pushing that to GitHub and then using that branch to make the pull request.

### Step 1: Adding the remote

The first thing you need to do after making the commits you want shared is establishing some way of loading the other project's current git data---i.e., [adding a "remote" repository](https://help.github.com/articles/adding-a-remote/). A "remote" repos is one that is hosted on the internet somewhere, i.e., on GitHub.

If the repo you want to make a pull request to is one that you forked _your_ repo from, then we call [that repo "upstream"](https://stackoverflow.com/questions/2739376/definition-of-downstream-and-upstream) from yours. We need to tell git which project this is with:

```
git remote add upstream https://github.com/user/repo.git
```

where `https://github.com/user/repo.git` is the URL of the git file of the repo you forked, like `https://github.com/excentris/compass.git` for me.

If you're making a pull request to a "downstream" repo (like Andrew's is to mine) or just a completely unrelated project, you can do:

```
git remote add <project_name> https://github.com/user/repo.git
```

where `<project_name>` is whatever name you want to call this new remote repo you're linking to. I called mine `AndrewMaster` since I was linking it to his `master` branch.

## Step 1.5: Verify the remote

Do `git remote -v`. You should see something like:

```
<project_name>  https://github.com/user/repo.git (fetch)
<project_name>  https://github.com/user/repo.git (push)
```

## Step 2: Fetching the data from the remote repo

You need to load the information about the remote repo. You can update this info with:

```
git fetch --all
```

## Step 3: Making a new branch to hold the commits

You need to make a new branch that will hold _only_ the new commits you want to share. Presumably you want to do a pull request to the remote repo's `master` branch, hence the `/master` below:

```
git checkout -b <new_branch_name> <remote_repo_name>/master
```

where `<new_branch_name>` is what you want to name this new branch (I picked `andrew_use_this`), and where `<remote_repo_name>` is the name of the remote repo (e.g., `upstream` or `AndrewMaster`).

## Step 4: Adding in the specific commits

Go to your GitHub repo, or check `git log` to find the hashes of the specific commits you want to share. For each of these commits, do:

```
git cherry-pick <hash>
```

which will "cherry-pick" that commit and put it into the current branch.

**Note:** Do **not** add commits that reference files that aren't created through the new commits or don't already exist on the remote repo. Commits are just _changes_, and if you `cherry-pick` a commit that modifies a file that doesn't exist, things can get weird.

I accidentally did that here, which edited a `google_maps.html` that Andrew didn't have:

```
ZBurchill:burchill.github.io zburchill$ git cherry-pick 374d4b313a7fa18f1d1ec62d43fee29283b10d65
error: could not apply 374d4b3... shoulda committed this earlier
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'
```

After doing a `git status`, I got:

```
ZBurchill:burchill.github.io zburchill$ git status
On branch andrew_use_this
Your branch is ahead of 'AndrewMaster/master' by 3 commits.
  (use "git push" to publish your local commits)

You are currently cherry-picking commit 374d4b3.
  (fix conflicts and run "git cherry-pick --continue")
  (use "git cherry-pick --abort" to cancel the cherry-pick operation)

Unmerged paths:
  (use "git add/rm <file>..." as appropriate to mark resolution)

	deleted by us:   google_maps.html
```

I fixed this mistake with `git rm google_maps.html`.

## Step 5: Get it on GitHub

To make a GitHub pull request, the code has to be on GitHub. Do:

```
git push -u origin <new_branch_name>
```

## Step 6: Make the pull request!

Now that your code is on GitHub, navigate to the project you want to modify, and click "New pull request," making sure that you reference the `<new_branch_name>` branch of your current project. Depending on the setup, you might need to make the "base fork" the project you want to change and the "head fork" the branch you just pushed.

Simple as that!


<hr />
<br />

### Footnotes:

[^1]: Much of this tutorial is built on Joseph Silber's answer to [a question on stackexchange](https://stackoverflow.com/questions/34027850/how-to-pull-request-a-specific-commit).
