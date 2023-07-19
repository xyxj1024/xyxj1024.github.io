---
layout:     post
title:      "Technologists' Opinions on Kubernetes"
category:   "Cloud Native Computing"
tags:       open-source kubernetes devops
permalink:  /blog/technologists-opinions-on-kubernetes
---

It is quite fun watching people exchanging thoughts and ideas on Twitter, even though from time to time people no longer care about their intellectual humility and turn the discussion into a fight. Everyone, more or less, wants to be heard. The Internet has taught me that information sharing, as a human instinct, eventually makes social progress possible. This is especially true for OSS ("open source software") in that to facilitate open discussions and to have faith in collective intelligence are what we have luckily done right so far.

<!-- excerpt-end -->

I decide to document here a recent debate on [Twitter](https://twitter.com/) about [Kubernetes](https://kubernetes.io/), which prompts me to ask myself:

*What might be called a good attitude towards an emerging OSS technology?*

## Table of Contents
{:.no_toc}
* TOC 
{:toc}

## Pete Cheslock

[Pete Cheslock](https://pete.wtf/) posted a [Twitter thread](https://twitter.com/petecheslock/status/1671536499748118532) on June 21, 2023:

> Kubernetes has single-handedly set our industry back a decade. Companies are going to die because they spend more time "managing Kubernetes" than building a product.
>
> Engineers brought Kubernetes to their jobs in the ultimate expression of "resume-driven development". But sure let's take Google's failed project and adopt it widely across the industry.

Pete's phrasing is eye-catching, indeed. [Scott Lowe](https://blog.scottlowe.org/) seems to be one of those that share the same feelings with Pete:

> There's a bit of hyperbole in this thread, but there are also some very valid points. Pete's point about "resume-driven development" is, I feel, spot on. That said, I do believe there are some use cases where the functionality Kubernetes offers outweighs the complexity it adds.

## Kelsey Hightower

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

And then, there is [Gabe Monroy](https://twitter.com/gabe_monroy?lang=en)'s reply to Kelsey:

> This was exactly the premise of [Deis](https://github.com/deis) way back when Kubernetes was first becoming popular: developer-oriented UX that drives a declarative system like Kubernetes under-the-hood. The challenge is balancing imperative vs. declarative without creating unmaintainable snowflakes.

[Matt Boyle](https://mattjamesboyle.com/)'s [argument](https://twitter.com/MattJamesBoyle/status/1668518042085597189):

> People like to criticize Kubernetes for being too complicated, but the reality is if you want to write your application in multiple languages (which I do nearly every time) there is not many easier/better/cheaper ways to deploy it than to use a managed k8s cluster.

## Joe Beda

[Joe Beda](https://hachyderm.io/@jbeda)'s [long thread](https://twitter.com/jbeda/status/993978918196531200) back in 2018 (this is gold!):

> First off: Kubernetes *is* a complex system. It does a lot and brings new abstractions. Those abstractions aren't always justified for all problems. I'm sure that there are plenty of people using Kubernetes that could get by with something simpler.
>
> As an example my son (9yo) wanted to me to teach him Kubernetes but I started with simple imperative Docker on the command line on a singleton GCE instance. Once he gets those concepts nailed we'll start talking k8s.
>
> That being said, I think that, as engineers, we tend to discount the complexity we build ourselves vs. complexity we need to learn.
>
> When you create a complex deployment system with Jenkins, Bash, Puppet/Chef/Salt/Ansible, AWS, Terraform, etc., you end up with a unique brand of complexity that *you* are comfortable with. It grew organically so it doesn't feel complex.
>
> But bringing new people on to help on an organically grown system like this is difficult. They may know some of the tools but the way that you've put them together is unique.
>
> This is a place where, IMO, Kubernetes adds value. Kubernetes provides a set of abstractions that solve a common set of problems. As people build understanding and skills around those problems they are more productive in more situations.
>
> There is still a steep learning curve! But that skill set is now valuable and portable between environments, projects and jobs.
>
> Beyond this, Kubernetes allows for "operations specialization". Not everyone has to be an expert in all parts of the stack. Things like operating a cluster can be handed off to specialists&mdash;either on staff or via a cloud provider.
>
> The story of computing is creating abstractions. Things that feel awkward at first become the new norm. Higher layers aren't *simpler* but rather better suited to different tasks.
>
> Modern JavaScript, for example, is incredibly complex. There are a ton of concepts and ideas that alien to, say, a C++ 3D game engine programmer. When we created Kubernetes we aimed to create the *right* abstractions for modern scalable server side apps.