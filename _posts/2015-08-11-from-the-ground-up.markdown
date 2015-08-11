---
layout: post
title:  "From the ground up"
date:   2015-08-11 10:49:55
categories: general
---
In software, we are often concerned with _getting things done_. At Mozilla this is especially true, where [Bonus Driven Development][bonus-driven-development] rewards completing deliverables in time.

However, often small matters overlooked today, mean big problems for tomorrow.

When you build a skyscraper, you build different foundations to a house. *No* detail is too small to be overlooked. It is the same with software.

For me, this means things like having API metadata that can be interpreted by clients, used to generate documentation, provide run-time validation of inputs/outputs, and be easily versionable/extensible.

These things give you small gains in the early days, but with each storey that is added to your building, the higher these gains become.

Keeping a well defined separation of concerns is also a principle which becomes increasingly important, the higher your building rises.

Ultimately, it is a matter of how high you wish to go. I want Taskcluster to be developed long into the future, to be usable by a wide variety of users and products, and to be easy to build on top of for a long time to come. This means we need exceptional quality _from the ground up_. We need to build the architecture _to serve the development lifecycle process_. 

Challenges
----------

High aspirations can only be realised if we are prepared to build highly extensible and exceptional quality software from the ground up. It requires solving architecural issues _as soon as they are first realised_, rather than avoiding them, or accepting them as a consequence of the project's history. All architectural decisions have to be evaluated based on their merit, and their merit alone. If there is a better way, you should adapt. Accepting a poor design choice limits your ability to continue building on top of your product, and ultimately it will cost you the success of your product. Regardless of the amount of work to fix it - you can only overcome a limitation by fixing the problem at source. Furthermore, any limitation you discover _will_ hinder you later on, and ultimately will determine how far your product can be developed before it reaches the point that it is not feasible to be developed any further. You limit the height of your skyscraper.

Things that inhibit our growth trajectory
-----------------------------------------

I've talked a bit about why I believe it is important to tackle software problems in an obsessively perfectionistic way. Ultimately I believe it is the only way to build something with legs to survive. However, there is a reason I have chosen this sticky topic. I believe there are some areas of Taskcluster which already need addressing. These are areas of technical debt, deeply rooted, that will already be a huge effort to rework. However, as time progresses, they will only become harder to solve, so I believe it is right that we solve them now.

* We currently rely on scripting languages such as node.js and python for core components. Scripting languages do not offer the beneifts of compile-time checks, and make it quite possible to create code that can easily swallow errors, coerce data types incorrectly and introduce whole classes of problems that are quite avoidable in other languages (such as error-swallowing). The more that taskcluster undergoes development, the greater the problem this will be. We should use programming languages that have compile time safety checks, such as rust, go or java.
* We are increasingly focussed on _getting things done_ to support mozilla use-cases, taking our focus away from building a generic open source platform that can serve a larger community. This also reduces interest in Taskcluster from the community, which is an antipattern to success. We need to make serving the broader community a high level goal, with the intention of bringing in contributors.
* Installation of taskcluster needs to be trivial, and well-documented
* Documentation needs to encompass all aspects of interacting with Taskcluster at levels appropriate to the various user groups
* We need to keep a clean separation of Taskcluster from Mozilla-specific integration with Taskcluster
* We need to solve the Pulse dependency issue, and get Taskcluster working with vanilla RabbitMQ

[bonus-driven-development]: https://drive.google.com/a/mozilla.com/file/d/0B9useiHoaCWaUTV6WG5KRFN6TVRCZGd4eVRJRWFNNlJvRUxv/view?usp=sharing
