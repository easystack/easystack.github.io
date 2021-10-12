# How to contribute

## Report issues

A great way to contribute to the project is to send a detailed report when you encounter an issue. We always appreciate a well-written, thorough bug report and feature propose, and will thank you for it!

### Issues format

When reporting issues, refer to this format:

- Is this a BUG REPORT or FEATURE REQUEST?
- What happened?
- What you expected to happen?
- How to reproduce it (as minimally and precisely as possible)
- Your Eviroment versions
- Anything else we need to know?


## Submit pull requests

If you are a beginner and expect this project as the gate to open source world, this tutorial is one of the best
choices for you. Just follow the guidance and you will find the pleasure to becoming a contributor.

### Step 1: Fork repository

Before making modifications of this project, you need to make sure that this project have been forked to your own
repository. It means that there will be parallel development between this repo and your own repo, so be careful
to avoid the inconsistency between these two repos.

### Step 2: Clone the remote repository

If you want to download the code to the local machine, ```git``` is the best way:
```
git clone https://your_repo_url/projectname.git
```

### Step 3: Develop code locally

To avoid inconsistency between multiple branches, we SUGGEST checking out to a new branch:
```
git checkout -b new_branch_name origin/master
```
Then you can change the code arbitrarily.

### Step 4: Push the code to the remote repository

After updating the code, you should push the update in the formal way:
```
git add .
git status (Check the update status)
git commit -m "Your commit title"
git commit --amend (Add the concrete description of your commit)
git push origin new_branch_name
```

### Step 5: Pull a request to repository

In the last step, your need to pull a compare request between your new branch and development branch. After
finishing the pull request, the CI will be automatically set up for building test.

### Pull requests format

When submitting pull requests, refer to this format:

- What this PR does / why we need it?
- Which issue this PR fixes?
- Special notes for your reviewer
- Release note


### How to new post

All posts houses under 'content/posts', You can create a new post by creating a new .md file, prefixing it with anything you want. incremental numbers or date string will work well if you prefer.

The post contains two parts, the header part and content part. the header part of this file which starts and ends with 3 horizontal hyphen(---) is called the front-matter and every post that we write needs to be a front matter included here:

- title: This is the title of your post.
- date: This is the time that shows the posting time of your blog. The first portion is in the year-month-date format. You can edit the date and time as you wish.
- description: Add any description you like.
- author: Who is the post owner, we recommend to use your name or github id.
- categories: which category would this post be in. the posts will be listed by the category name. 
- tags: Any tag would this post be under.
- draft: Is this post draft or not, the value is either 'true' or 'false'.

You can refer to the following example and have your modification:

```
---
title: "your post title"
date: 2021-10-09T15:50:43+08:00
draft: false
description: "any description you would prefer to here"
author: "freesky-edward"
categories: []
tags: ["container"]
--- 
```

The content part can be any content following the markdown rules.

