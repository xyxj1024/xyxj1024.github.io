---
layout:     post
title:      "Technologists' Opinions on Kubernetes"
category:   "Cloud Native Computing"
tags:       container kubernetes devops
permalink:  /posts/technologists-opinions-on-kubernetes
---

It is quite fun watching people exchanging thoughts and ideas on Twitter, even though from time to time people no longer care about their intellectual humility and turn the discussion into a fight. The Internet has taught me that information sharing, as a human instinct, makes social progress possible. This is especially true for OSS ("open source software") in that to facilitate open discussions and to have faith in collective intelligence are what we have luckily done right so far.

<!-- excerpt-end -->

I decide to document here a recent debate on Twitter about Kubernetes.

[Pete Cheslock](https://pete.wtf/) posted a [Twitter thread](https://twitter.com/petecheslock/status/1671536499748118532) on June 21, 2023:

> Kubernetes has single-handedly set our industry back a decade. Companies are going to die because they spend more time "managing Kubernetes" than building a product.
>
> Engineers brought Kubernetes to their jobs in the ultimate expression of "resume-driven development". But sure let's take Google's failed project and adopt it widely across the industry.

Pete's phrasing is eye-catching, indeed. [Scott Lowe](https://blog.scottlowe.org/) seems to be one of those that share the same feelings with Pete:

> There's a bit of hyperbole in this thread, but there are also some very valid points. Pete's point about "resume-driven development" is, I feel, spot on. That said, I do believe there are some use cases where the functionality Kubernetes offers outweighs the complexity it adds.

Here is [Kelsey Hightower](https://github.com/kelseyhightower)'s response:

> If you don't need Kubernetes, don't use it. What is being described here was already happening. Companies are spending too much time managing CI/CD pipelines, IaC, random bash scripts, and a whole collection of custom tooling no one wants to talk about.
>
> Containers were about adopting a new abstraction and decoupling your application from the machine. Bundle your application and dependencies so you can spend less time messing with OS and configuration management tools. Docker and Kubernetes are optional.
>
> Kubernetes is an infrastructure framework for building your own platform. You layer it on top of bare metal, virtual machines, or better yet, an IaaS provider of your choice, and you get an opinionated way for deploying containers to servers and exposing them to the network.
>
> Where people get into trouble, and the point I believe [@petecheslock](https://twitter.com/petecheslock) was trying to make, is when they start cosplaying a cloud provider, and going overboard with operators just so they can attempt to run everything in Kubernetes.
>
> Kubernetes allows you to declare your infrastructure concerns, including storage requirements, load balancing and service discovery, and enough extension points to integrate your policies into the platform itself. This is 10x better than what enterprises were doing before.
>
> The problem is we asked developers to do all that. Kubernetes is not a tool for developers. They can use it, but we have to be honest, Kubernetes is low-level infrastructure and works best when people don't know it's there. Shifting YAML left was a mistake.

[Matt Boyle](https://mattjamesboyle.com/)'s [argument](https://twitter.com/MattJamesBoyle/status/1668518042085597189):

> People like to criticize Kubernetes for being too complicated, but the reality is if you want to write your application in multiple languages (which I do nearly every time) there is not many easier/better/cheaper ways to deploy it than to use a managed k8s cluster.