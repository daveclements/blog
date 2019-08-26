---
layout: post
title:  "Code Coverage"
# date:   2019-08-02 13:37:44 +0100
---
# What is Code Coverage?
Taken from [Wikipedia](https://en.wikipedia.org/wiki/Code_coverage): `In computer science, test coverage is a measure used to describe the degree to which the source code of a program is executed when a particular test suite runs`

Normally, this boils down to a percentage figure of how much of the code you have written is covered by unit tests

# Aim for 100%?
So a higher number is better. Right?

In my opinion, no.

I've worked with people that argue for 100% coverage and i've worked at companies where your code must have x% coverage or your PR won't be approved, or coverage dropping below x% will break the build and I think all of these practices are harmful

The problem with this kind of thinking is that it gives a false sense of security around how 'good' the code is and actively promotes writing point;ess tests

