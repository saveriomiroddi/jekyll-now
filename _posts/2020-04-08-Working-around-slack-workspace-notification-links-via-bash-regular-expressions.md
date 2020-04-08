---
layout: post
title: Working around Slack workspace notification links via Bash regular expressions
tags: [databases,indexes,mysql]
---

When Slack sends workspace notifications, it enforces an annoying[¹](#footnote01)] workflow, if the user just want to open the link in the browser ("Opening link in Slack...") without using the Slack desktop application.

In this post, I will explain you how to work this problem around, using Bash useful features like regular expressions and arrays.

Note that this is not a global workaround - it's targeted at users who launch the browser via a script - however, the features explained are still interesting for those who like shell scripting.

Contents:

<!-- [Terminology](/An-introduction-to-functional-indexes-in-mysql-8.0-and-their-gotchas#terminology) -->

- [The problem](#the-problem)
- [The regex approach](#the-regex-approach)
- [Bash implementation of the match, capture, and transformation](#bash-implementation-of-the-match-capture-and-transformation)
- [Bash arrays](#bash-arrays)
- [Putting everything together](#putting-everything-together)
- [Footnotes](#footnotes)

## The problem

https://exercism-team.slack.com/x-01234/archives/56789/0ABCD

https://exercism-team.slack.com/messages/56789/0ABCD

## The regex approach

`https://exercism-team.slack.com`

`/x-01234/archives/`

`56789/0ABCD`

...

`^https://[[:alnum:]-]+.slack.com`

`/x-.+/archives/`

`.+$`

...

`^(https://[[:alnum:]-].slack.com)/x-.+/archives/(.+)$`

## Bash implementation of the match, capture, and transformation

```sh
if [[ $address =~ ^(https://[[:alnum:]-].slack.com)/x-.+/archives/(.+)$ ]]; then
  address=${BASH_REMATCH[1]}/messages/${BASH_REMATCH[2]}
fi

firefox "$address"
```

## Bash arrays

```sh
myarray=(foo "bar baz")
echo $(IFS=,; echo "${pizza[*]}") # foo,bar baz

myarray+=(qux)
echo $(IFS=,; echo "${pizza[*]}") # foo,bar baz,qux

myarray[1]=sav
printf '%s,' "${myarray[@]}" # sav,bar baz,qux,

for entry in "${myarray[@]}"; do
  echo "$entry"
done

for (( i = 0; i < ${#myarray}; i++ )); do
  echo "${myarray[i]}"
done
```

## Putting everything together

```sh
#!/bin/bash

args=("$@")

for (( i = 0; i < $# ; i++ )); do
  if [[ ${args[i]} =~ ^(https://[[:alnum:]-]+\.slack\.com)/x-[[:xdigit:]]+/archives/([[:xdigit:]]+/[[:xdigit:]]+)$ ]]; then
    args[i]=${BASH_REMATCH[1]}/messages/${BASH_REMATCH[2]}
  fi
done

firefox "${args[@]}"
```

## Footnotes

<a name="footnote01">¹</a>: `-1` actually addresses the last field, but only when the first parameter in the range is also negative, e.g. `[-3..-1]`.<br/>

