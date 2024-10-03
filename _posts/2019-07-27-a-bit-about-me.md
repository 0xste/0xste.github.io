---
title: A bit about me
author: 0xste
date: 2019-07-27 20:00:00 +0000
categories: [Blogging]
tags: [software,architecture]
---

After an inordinate amount of procrastination I've finally bitten the bullet and am putting together a series of blog posts on modern cloud-native applications, platforms and design concepts. I will be drawing on my experience across a few different roles and providing examples, templates and design literature with an aim to use this series to have some interesting conversations on the topics covered.

As my fantastic photoshop skills may suggest... the initial posts will be based around (largely) GitOps and general platform engineering tooling, guides and examples.

## My Experience

Prior to working at Pollinate I spent some time as a Lead Backend Engineer at Atom Bank, working (largely) on building, instrumenting and deploying GO microservices for the (then) new Instant Access Savings product in GCP as well as being part of migrating their legacy infrastructure to the cloud. A scenario which I'm sure many of you reading this will be familiar.

At Pollinate, I'm working in the developer experience team, largely involved in building tooling and automation to improve the path-to-production for engineers to build and run our merchant acquiring platform for our clients (banks).

I often found it difficult to explain what exactly Pollinate do when I first joined, but this snippet on the the Pollinate website states it a lot more eloquently than I ever could.

> Pollinate gives banks a modern toolkit for small businesses. Cloud based, and designed to take data feeds from existing bank and third party systems, it gives a small business a single place to manage their business, and is built with bank-grade privacy and security for peace of mind.

Simply put, we run a common platform for different banks (currently Tyl by Natwest and soon National Australia Bank) that enable businesses to accept payments online, via terminal among other things.

As you can imagine, there's a lot of banking regulation and contractual work that means essentially our core business is running a lot of infrastructure for a lot of banks given various global and local banking regulations as well as banks personal risk tolerances. 

## Overview of coming content
I've always been strongly of the belief that a software engineer should be responsible and and accountable for the software from design to production. In that breaking down silos between teams (although I'm yet to see the utopian "Netflix" model anywhere in practice) is the means in which to do so. In recent years with the advent of containers and infrastructure as code, this has largely become within reach for many organisations, it is now largely a people problem not a technical one. The most successful implementation I've seen thus far is to empower developers through tooling in the platform tier to entirely instrument everything from the codebase up through code, in practice means engineers are responsible for containers and configuration, but the build, validation and deployments are abstracted as much as possible so the underlying infrastructure concerns. We'll cover this concept more detail as the blogs progress.

Given day-to-day I'm working largely with Azure (for now at least) many of these solutions covered will be implemented using Azure tooling, but the concepts and principles could be applied elsewhere, in terms of how we separate the "platform" at pollinate we've got two teams working together with differing aims.

Solutions will vary in language and implementation, but I'm a big advocate of best-language-for-the-job, largely these solutions will be Terraform HCL with sprinklings of GO and Bash.

## Wrap Up
Coming from a background where running a single production environment is often an significant challenge, moving to a company in which striving to run many as it's core business is a welcomed challenge.
