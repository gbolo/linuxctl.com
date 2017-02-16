---
categories:
  - "development"
  - "source control"
keywords:
  - "git"
tags:
  - "git"
date: "2016-03-21"
title: "Git - preserve history when moving files"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-github.jpg"

# url of old site
aliases:
  - "/git-preserve-history-when-moving-files"

---

When moving files/directories around in a git repo, the version history of the file gets lost! Let's fix that.
<!--more-->

While working on my [vSummary](https://github.com/gbolo/vSummary) github repo, I came to a point where I had decided to change the folder structure of it. At this point I had about 60-ish commits to it. So I began by using the `git mv` command to move and rename directories then finished off with the usual commit and push commands. However, when I took a look at some of the individual files in the repo I noticed that the commit history for them were gone. While github still had all my commits, the history for individual files/directories seemed to have been erased with the restructuring commit. I did not expect that, but according to github, this is the expected behaviour: [Why does github not track renames](https://git.wiki.kernel.org/index.php/GitFaq#Why_does_Git_not_.22track.22_renames.3F)

Fortunately, I was able to fix this by rolling back my latest commit and using this script to preserve the history of my restructuring:
<script src="https://gist.github.com/emiller/6769886.js"></script>

Once you are finished with your history rewrites, you can force a push like: `git push origin master --force`
