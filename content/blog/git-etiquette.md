---
title: "Git Etiquette"
date: 2023-04-23T16:03:26+01:00
tags: [git]
draft: false
featured: true
image: "https://i.ibb.co/wp0JVn9/yancy-min-842of-HC6-Ma-I-unsplash.jpg"
alt: "Git Log in a VSCode panel"
summary: "Five simple practices to Git like a pro and make your collaborators (and your future self) happier"
description: "Blog: Five simple practices to Git like a pro."
---

Rationale
---------

Git is a powerful tool that generated an ecosystem of other powerful tools which
made developing software in a collaborative environment much, much easier (does
anybody remember using CVS?). So it would be a pity to use it only for pushing
code on a remote repository when you're done coding for the day.

I have a feeling that the importance of writing good commits is often
overlooked. I've seen lots of projects with commits messages like `Fixed typo`,
`updates` or `trying something new` the latter likely labelling a commit with
hundreds of changes. The following are real commit messages from a very
successful FOSS library: `\o/`, `✨🍰✨`.

I'm always up for a laugh and strongly believe a little dose of silliness can
lighten the mood and improve your workday, but my mood wouldn't definitely be
uplifted by a cake emoji when I'm trying to reconstruct the history of a branch
in order to resolve a merge conflict which is blocking the release of a patch
our team desperately need. So, please be funny, but also be smart.

[![xkcd.com/1597](https://imgs.xkcd.com/comics/git.png)](https://xkcd.com/1597/)

Git commits are more important than most people think and can save you **A TON**
of work if written with common sense. Despite Git itself often being not very
intuitive and borderline scary, writing good commits is actually quite easy!

The Interwebs are full of blog posts on how to write Git commits and merge/pull
requests, but they mostly focus on one single aspect. In this post I've
summarised five simple but fundamental rules that can greatly improve you (and
your collaborator) coding life, with minimal effort.


Five Simple Rules
-----------------

### 1. Write atomic commits

An atomic Git commit should contain all and only the changes involved in a
single unit of work; it doesn't matter if the change is a single character or
spans several lines in multiple files, rather it means that you should be able
to describe your changes with a single, meaningful message. Moreover, you should
be able to revert an atomic commit without any unwanted side effects or
regressions, aside from what you’d expect based on its message. This is crucial
when a change needs to be reverted and can save a very long time when
troubleshooting problems.

Further reading on this topic:
[aleksandrhovhannisyan.com/blog/atomic-git-commits](https://www.aleksandrhovhannisyan.com/blog/atomic-git-commits/)


### 2. Write clear commit messages

Commit messages are an invaluable piece of documentation, as they're "attached"
to the code and can be quickly read by most IDEs. Moreover, if well written,
they can give unique insight on the evolution of a piece of code, so it's
important to include not only **what** was changed but **why** was that changed
and why changed **that way**. Also, don't assume who will read the commit
message understands what the original problem was, don't assume the code is
self-evident/self-documenting and remember that more often than not, commit
messages are **the only documentation**. So, storing any valuable piece of
information about the changes in the message is very important!

Commit messages should be formatted like emails. Think about the first line of
the commit message as its subject (try to keep it within 50 columns) and the
following lines as the body of the message (try to keep it within 72 columns);
leave a blank line between subject and body.

Use imperative and succinct language for the first line (subject) and then write
as many body lines as you need. Bullet points are okay (use hyphens or asterisks)
for the bullet followed by a single space.

50 and 72 columns limits might seem too tight, but considering how much more
information is usually displayed alongside a commit message (e.g. git log tree
view, blame gutters or simply your favourite IDE's side panel) keeping your
messages between those limits makes reading commit messages much easier in
virtually all cases. Don't take my word for it, take
[Tim Pope's](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html).

Great (even though a bit hardcore) examples of very well written commit messages
can usually be found in the [Linux kernel repository](https://github.com/torvalds/linux/commits/master).

More info on clear commit messages:
[freecodecamp.org/news/how-to-write-better-git-commit-messages](https://www.freecodecamp.org/news/how-to-write-better-git-commit-messages/)


### 3. One merge request per concern

Merge requests (or pull requests if you're familiar with GitHub) should cover
only one concern (e.g. adding one feature, fixing one bug) and as with atomic
commits, reverting them should remove all and only the changes related to that
concern. Moreover, strive to keep pull requests "small" as code reviews quality
tends to be inversely proportional to the number of changes to review (i.e. it's
likely that reviewers will tend to skim through pull requests with hundreds of
changed lines, rather of accurately review all of them).

{{< x user="iamdevloper" id="397664295875805184" >}}


### 4. Remember the golden rule of rebasing

**TL;DR** Never rebase onto public branches.

Rebasing is a nice way to keep Git history linear and avoid "annoying" extra
merge commits, but one must keep in mind that rebasing "rewrites" history. For
example, imagine you're working on a branch forked from master, you add some
commits to it but then your fork falls behind master. Now you want to pull those
changes and rebase your fork on master, so you `git pull origin master --rebase`,
well, this will "rewrite" your branch history as it will delete your commits from your
fork (stashing the changes somewhere), fast-forward adding the commits from
origin master, then re-apply your changes as new commits. Your commits will have
same changes, commit messages and time stamp, but their hashes will be new. This
is totally fine if your fork is local, but, if you pushed your commits to the
remote before rebasing, then pushing "re-written" commits will fail. At this
point you could force-push or do another merge/rebase, but in any case merging
your forked branch back to master will duplicate commits. Also, at this point,
if someone else pulled your forked branch before you "rewriting" it, will have a
diverging tip and get very confused when trying to pull/push again. Moreover,
upon merging the fork back onto master, "rewritten" commits will end up as
duplicated. In order to avoid all the above, is good practice to rebase only
local branches (branches that haven't been pushed onto a remote).

More info: [atlassian.com/git/tutorials/merging-vs-rebasing#the-golden-rule-of-rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing#the-golden-rule-of-rebasing)


### 5. Pull frequently to minimize merge conflicts and derived bugs

Pulling new changes off upstream daily, reduces the chances of ending up with
annoying (and potentially painful to fix) merge conflicts when opening a
merge/pull request. Resolving merge conflicts can be a very delicate matter and
even when the code is well understood the chances to introduce bugs are very
high.
