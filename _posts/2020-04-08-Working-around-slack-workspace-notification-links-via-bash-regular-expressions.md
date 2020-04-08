---
layout: post
title: Working around Slack workspace notification links via Bash regular expressions
tags: [databases,indexes,mysql]
---

When Slack sends workspace notifications, it enforces an annoying[¹](#footnote01)] workflow, if the user just want to open the link in the browser ("Opening link in Slack...") without using the Slack desktop application.

In this post, I will explain you how to work this problem around, using Bash useful features like regular expressions and arrays.

Note that this is not a global workaround - it's targeted at users who launch the browser via a script - however, the features explained are still interesting for those who like shell scripting.

Contents:

- [Terminology](/An-introduction-to-functional-indexes-in-mysql-8.0-and-their-gotchas#terminology)

## Footnotes

<a name="footnote01">¹</a>: `-1` actually addresses the last field, but only when the first parameter in the range is also negative, e.g. `[-3..-1]`.<br/>

## Terminology

