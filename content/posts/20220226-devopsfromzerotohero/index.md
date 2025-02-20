---
title: "DevOps From Zero To Hero"
description: "The journey starts with a small step."
summary: "The journey starts with a small step."
categories: ["Programmer"]
tags: [ "myhistory", "devops", "cm-cli"]
#externalUrl: ""
date: 2022-02-26
draft: false
authors:
  - Roque
---

In the beggining there was a powershell script📜...

I wasn't there in the beggining, but the beggining seemed to still be there when I arrived. 

## How I got drafted to DevOps

I started out as a developer on the first implementation of a new module of our product. Being the new guy 🧑‍💻 and the only one that kind of had some knowledge of this new module I was sent to the frontlines of "do something about the release". 

Every project had a slightly different build and pipeline so you wouldn't be able to find help. 

The first thing I did was look at what this release was all about. In essence the release had three parts, clean the database, install the code and run tests. I looked at this and asked how could I add my module with my tests to this release. The database part seemed good enough, so I focused on the other two.

I am not proud, I added the new implementation to the powershell script.

## Redemption

The first thing the devops team did was fix the release process. Guess who was the only one that had any experience with the release process for that product module. You guessed right it was me.

We built an entire dotnet [CLI]("https://github.com/criticalmanufacturing/cli") and completely reshaped the way we work. It was devops full throtle, from tooling to mindset. No more "my version" or "my project". A completely unified way of work and something that everyone could now build upon.

I think everyone has their version of this story, if they work long enough and care about their work. We're continuing the good fight and one thing is for sure, there's always the next problem.