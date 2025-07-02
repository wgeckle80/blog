---
title: NetBSD microvm, Git, and PV Clock
date: 2025-07-01
categories: [Blog]
tags: [blog]
---

Tom and I had a call today discussing commit history pertaining to the NetBSD
microvm implementation, viewing Git logs locally, and next steps regarding the
PV clock driver.


## NetBSD microvm Commits

Tom knew that the user [iMil](https://github.com/iMilnb) is the primary
contributor for the NetBSD microvm implementation. On Github, his commits to
the NetBSD source are found at
<https://github.com/NetBSD/src/commits?author=iMilnb>. They're easy to parse
since, as of today, he has done a relatively small number of commits to the
project, and all of them but one from 2009 pertain to the microvm port.


## Git Logs

To more effectively view Git logs locally, Tom provided the following list of
Git commands:

```sh
git log                      # show everything that has happened in the source tree chronologically
git log .                    # show log for this directory
git log --author="Tom Jones" # show Tom Jones' commits
git log --stat               # show the files changed and their churn for commit log
git log -p                   # show a patch along with commit
git show <commit>            # show log and changes for a commit
git log <file>               # show all changes for a file
```


## PV Clock

It may be worth looking into using PV clock in the microvm build instead
of TSC. We previously disabled it entirely to bypass a TSC `KASSERT`, but my
assumption is that we shouldn't be getting to the check in the first place if
things are properly configured. The PV clock driver implementation in FreeBSD is
located at `sys/x86/x86/pvclock.c`.

If it's at all helpful, we also noted that OpenBSD has a PV clock driver
implementation. A minimal manpage for the driver is located at
<https://man.openbsd.org/pvclock.4>. This may or may not be helpful in
figuring out what PV clock is, as I currently don't know too much about it.
Either way, if changing the configuration doesn't work right away, which it
probably won't, I'll want to look more into what the PV clock actually is.
