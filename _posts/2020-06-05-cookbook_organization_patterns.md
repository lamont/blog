---
layout: post
title:  "Cookbook Organization Patterns"
tags: chef cookbook
author: jcook
---

We've worked with dozens of clients and their varying ways of organizing Chef policy. 
Every team's needs and organizational structure dictates the shape of their chef policy repository structure and 
cookbook dependency tree.  With that uniqueness in mind, here's an example of a structure
we setup recently.

The clients requirements/comfort points which influenced this layout were:
* Migrating existing Chef code from AWS OpsWorks
* Mostly app developers with a good working knowledge of VCS workflows/repo dependency tracking
* No desire to run their own Chef server
* Small handful of unique runlists to manage, perfectly reflected upwards through higher environments
* Everything should be driven/checked/enforced via CI

We’re using three types of cookbooks in this world:
* Library cookbooks - like fr_deploy, fr_rsyslog, fr_nodejs…etc
* Application cookbooks - like app-foo, app-bar
* Environment cookbooks - env-development, env-staging, env-production

Library Cookbooks
===

Library cookbooks come in two flavors:
* They generically install one supporting services like rsyslog or nodejs
* They provide custom resources that enforce an installation pattern for your applications like the old OpsWorks deploy

Library cookbooks sometimes don't have recipes but their kitchen and chefspec tests demonstrate a 
generic example of the cookbook's use, demonstrating the custom resource `deploy` or the generic `rsyslog` install. 
These tests run with almost no site-specific details required. 

These cookbooks ideally have minimal dependencies.  Library cookbooks can call other library cookbooks to make sure common base functionality is installed if the pattern requires it.  An example is `deploy` has recipes that include recipes and use custom resources from `nodejs`.

Application Cookbooks
===
Application cookbooks build on the library cookbooks by combining the custom 
resources, recipes, default attributes, and pull data from a dynamic, external 
source like AWS's System Manager's Parameter Store to build a client application 
like `foo` or `bar` from our list above. The bulk of the business logic and smarts 
around installing and operating the application are stored in this layer.

Most tests exist in these cookbooks.  This is where we test the cookbooks build the application we expect.  In some cases, we have written _kitchen.rb recipes to launch and build instances in AWS using default attributes that we can test for. Usually you can store all of that test functionality
in a `kitchen.yml` which describes a minimal yet functional generic instance of the application in question. 

Environment Cookbooks
===
Environment cookbooks in most cases will only every set the 
node['ssm']['environment'] attribute and then call a series of application 
cookbooks…but it also gives you a place to override the default attributes 
with your own environment specific attribute value while using the existing 
deploy pattern to achieve a new result.

In the absence of a Chef server and the abstraction layer that Chef Environments provide, 
we're using a cookbook to realize those required features:
* Overlaying environment-specific attributes (e.g. choosing a specific git branch for deployment, 
or wiring the application to a different external data source)
* Locking a known good set of application and library cookbooks versions 

We could write tests here, but unless you are adding in logic that will 
determine the value of an attribute or wrap a specific resource block in 
an if/else statement…all you would be doing is testing that the framework 
works as advertised. If the cost of a failed/bad deployment was so high that you 
want Maximum Testing this would be a very reasonable place to add kitchen integration tests
that attempt to run in the actual named environment, but generally we don't end up with a
lot of integration testing at this layer.

Where to put new features?
===
We look at these levels of cookbooks as going from very general -> environment 
specific.  As a developer, you have many options as to where you want to make 
your change.  Is this change happening on every box?  It's probably a library 
cookbook change.  Is it for all applications of this type?  
It's probably an application cookbook change.  If we only want prod to act this 
way…environment cookbook change.

