---
layout: post
author: Valentin Rothberg
title:  "Mail Filters for GitHub Notifications"
date:   2018-08-29
tags: github mail email filters open source
---

In this blog post, I want to describe how I believe having tamed the beast of daily mails coming into my inbox from GitHub and how I use mail filters to instantly see which issues, pull requests, comments and discussions I need to address first.  Note that I try to keep this blog post short by focusing more on the solution rather than on the problem.

## GitHub Notifications

GitHub provides a rather simple and pragmatic way of dealing with notifications.  At the time of writing, GitHub knows two categories: **participating** and **watching**.  According to the [documentation](https://help.github.com/articles/about-notifications/) I will receive *participating* notifications "when I am directly involved in activities or conversations within a repository or team that I am a member of".  The documentation further states that I will receive *watching* notifications "for updates in repositories or team discussions that I am watching [and that do not qualify as participating notifications]".

GitHub notifications can be watched in a nice web interface sorted by project on GitHub and, at wish, they can also be send via email to a specified email address.  Shortly after I started using GitHub more professionally, I switched to using email-only as the web interface just did not scale for me, especially with a growing number of projects that I am involved in.  Using mails for notifications has many benefits for me but the most important ones are:

  * I have **mail threads** which give me a context that allows me to catch up more easily.

  * I can watch the notifications on **any device** that does email.

  * I can **control** which kinds of notifications may or may not be **important for me**.

  * I can **reply via email** which scales much better.

## Mail Filters

After having changed to mail notifications, I was exposed to a new threat: mails, thousands of them, more than I ever had on the Linux Kernel mailing list.  I really hate distractions and obviously I could not be interested in or scale to hundreds of emails per day about all kinds of topics in projects that I am involved in. The solution was to use **email filters** to have *full control* over what needs my attention first and what may be important to me.

To make it short, I use the following set of mail filters:

 **Forward filters**: any email coming from GitHub (i.e., `notifications@github.com`) will skip my inbox and be placed in a dedicated mail folder.  Currently, I have only one folder for all GitHub mails but I could create more subfolders for projects or groups of projects to further tackle the noise.

**Tagging filters**: to discriminate *important* notifications (e.g., somebody asks me to review something) from less important ones, I have a few filters to *tag* or *mark* emails as important.

 - **Own actions**: any action performed by me, such as opening a new issue or pull request, will be marked as important.  I need to catch up on things and want to see immediately if something new flew in for those actions.  GitHub makes this task easy as such emails will be sent by `Valentin Rothberg <notifications@github.com>` instead of just `notifications@github.com` as for "normal" notifications.  Certainly, you should change the sender's address to reflect your real name or your user name, depending on which data you gave to GitHub.

 - **Requests**: a very GitHub-ish way of requesting something from someone is by referencing the user name, in my case via `@vrothberg`.  Such requests are generally asking for feedback, for reviews, for attention, an so on.  If somebody requests my attention, I really want to be reliable and reactive, so I better discriminate those mails as well and I do this by looking for "`@vrothberg`" in the mail contents.

 - **Important projects**: I have some cases, where I consider any activity on a given project to be important.  Again, GitHub does a wonderful job here, as the email notifications include for which project they're being send.  Let's assume, I am interested in the project `github.com/lovely/project`.  All notifications from this project will contain the following strings that can be used for filtering:
```
List-ID: lovely/project <project.lovely.github.com>
List-Archive: https://github.com/lovely/project
```

And that's all I do to keep my sanity in this context.  To some readers this may sound simple or like a rather trivial or obvious solution, and if that's the case, I consider you to be one of the few people in the wild that use mail filters (for GitHub).  I have seen many people struggling with GitHub notifications and mails in general, and I intend this blog post to be some source of inspiration for how the beast can be tamed. Certainly, there is lots of space for improvement in my described setup but it is currently working for me.  It seems to make me scale well across many projects with hundreds of GitHub notifications per day.
