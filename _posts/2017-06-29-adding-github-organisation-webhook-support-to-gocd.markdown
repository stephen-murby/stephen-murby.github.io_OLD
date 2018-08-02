--- 
layout: post 
title: Adding GitHub Organisation Webhook Support to GoCD 
date: 2017-06-29 
author: Stephen Murby
---
The bulk of our active codebases, over time, have made their home our GitHub Enterprise server. We also have a [GoCD](https://www.gocd.org/) (continuous delivery pipeline) server that is polling these repositories to work out if it has something to do. The upshot of this is that, every minute, for each of these codebases, GoCD polls Github for changes. This consumes a lot of unneccessary CPU cycles (especially because some of these sources haven't been updated recently) and is one of the reasons our GoCD server is slower than we'd like it to be. This blog post will talk about how we improved this and my experiences while contributing code back to the open source community.

We had an appetite to trigger our pipeline builds instead by a GitHub push notification. However, the only way of doing this was to use the [GitHub GoCD service integration](https://github.com/github/github-services/blob/master/docs/gocd), secured with basic authentication for each repository. To do this across 2,500 pipelines we thought would be unmanageable as each team would have to remember a) to do this for each of their projects and b) enter their own credentials.

<img style="float:right; width: 15em;" src="{{ site.github.url }}/images/2017-06-29/gihub-to-gocd.svg">

## Early days
I talked to ThoughtWorks [(here)](https://github.com/gocd/gocd/issues/217) and proposed a community plugin to support GitHub organisation webhook messages. Other people had gone down the route of writing a custom proxy to handle the disparity between the messages emitted for a GitHub organisation, and those expected by GoCD `notify` endpoint. I felt it would be more beneficial to write something that benefited everyone rather than just our own use case. After a small code spike and a little digging around in the GoCD core, I found that it was not possible to do this with a plugin. Instead I was going to have to make a change to GoCD core and send them over a pull request.

The core team are based over in India, so the only real way of collaborating was over their 
[(gitter)](https://gitter.im/gocd/gocd) channel. This presented some difficulties as I only spend a small amount of time on this project each week (in my personal development time at work). By the time that I get going, they have almost always finished for the day, so communication and getting answers to questions could sometimes take up to a week or two.

I made a good first stab at a solution by trying to hook into the existing `PostCommitHookImplementer` mechanism. Doing this created complexity around the matching of GitHub repositories to GoCD cruise materials, which could be configured to have one of a number of forms of URL:

```
 https://github.com/organisation/repository
 https://github.com/organisation/repository.git
 http://github.com/organisation/repository
 http://github.com/organisation/repository.git
 git://github.com/organisation/repository
 git://github.com/organisation/repository.git
 git@github.com/organisation/repository
 git@github.com/organisation/repository.git
``` 
GoCD expects a custom header on all of the existing API request URLS. While I was testing the `/notify` endpoint using [Postman](https://www.getpostman.com/postman) this was no problem at all, but when I came to test it with a dummy GitHub webhook request I noticed I couldn't configure such a thing. I raised this with ThoughtWorks, and for the time being we relied on the fact Rails combined all the path parameters and form-encoded data together, passing it down into the service layer in a `$data` variable. This allowed me to carry on unblocked while ThoughtWorks figured out how they wanted to approach a solution to this issue. So I finished off the implementation I was going with and went back to the discussion with ThoughtWorks. They had come up with a best fit solution for now. That was to move the task of accepting vendor webhook notifications to their own set of new API endpoints, which sat outside of the current authentication filter. 

Interestingly GoCD has a mix of languages used in its implementation. The service layer is written in Java and the API layer written in Ruby on Rails, running on JRuby. Until now I had only been making changes to the Java service layer, which I am comfortable with. Creating the new API endpoints meant delving into the Ruby controllers. I cobbled together something that seemed to work, mostly by copying the way the other controllers had been written. However, I was really struggling with how I was supposed to write the tests. Unfortunately there wasn't anything similar in the project to go off. 

I battled with this for almost a whole day, while the core team were tucked up in bed, but before I left for the day dropped a message to ThoughtWorks asking if it was possible to pair with someone. Amazingly, by the time I got home, I had had a reply (the support core team don't sleep at all it would seem). "Yes", a core team member suggested he could pick it up with me the following morning - super contributor support from them! It did mean a 5am start for me though. No pain, no gain, right?

I pair almost every day at work, but I had no idea how this was going to go with someone half-way around the world. It turns out the team do this quite often and have a tool dedicated for it. Although it wasn't working at the time resulting in us using a tried and tested... Google Hangout! The only disadvantage was that while I could share my screen, I couldn't share keyboard input etc. 

Very quickly we had a working endpoint **and** tests! They are really easy to write when you know how. Seeing the light at the end of the tunnel, we discussed putting my PR through the build pipeline that day and getting merged and into a release the following day. One job remained - to implement some kind of authorisation for these new endpoints. 

GitHub only really provide one mechanism for doing this - they will sign the entire request payload with a secret. Looking around at the other vendors like Gitlab and BitBucket, there didn't appear to be a common solution that would work for each of them. Therefore supporting other vendors webhooks is going to require a similar developer effort to get something working. This in mind, the GoCD product lead suggested we add a single `webhook-secret` to the config xml, which is generated by GoCD and used to sign the incoming payloads. It was quick to implement and they could gather feedback from the community as to which direction to go next.

Lastly, I needed to update the docs for GoCD to describe the new feature and usage. I encountered a number of problems with ruby gems while trying to build the docs project, but with excellent support from the core team I was able to use Docker to get them up and running locally. [Here they are,](https://api.gocd.org/current/#github-webhook) and if you want to have a look at the code [this is what I have done](https://github.com/gocd/gocd/pull/3437).

Furthermore, my change is released! I'd like to offer a huge thanks to ThoughtWorks for supporting the contributing community like they do, and to Auto Trader for giving me the time to give something back to the Open Source Software we use every day. 

## The experience
This has been a great project for me, it has enabled me to see some other approaches to solving some of the 'enterprise' API problems that we all have. While we have lots of developers here at Auto Trader, I tend to work with the same few quite often. This means my exposure to alternative approaches is sometimes a little lacking, so from a professional development point, it has worked wonderfully. In addition to this, it's my first real contribution to Open-Source Software. I have been able to see how this open and collaborative way of building software works. While GoCD is open source, it does have a dedicated team of ThoughtWorks core developers, and they have been invaluable to me getting my contribution over the line. If you are inspired to make your own contribution, they would love to hear from you on [gitter](https://gitter.im/gocd/gocd). As a thankyou for my contribution the core team sent over some GoCD swag.

<img src="{{ site.github.url }}/images/2017-06-29/gocd-swag.JPG">
