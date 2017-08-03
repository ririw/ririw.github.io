---
layout: post
title: "Cargo cult data science"
date:   2017-07-25 00:00:00
---

Data science as a cargo cult
============================

Data science is becoming an important competitive advantage for companies of all kinds.
With the introduction of a new set practices, there's an increased risk of "[cargo cults](https://en.wikipedia.org/wiki/Cargo_cult_programming)" emerging within organizations. 
Cargo cults describe situations where people emulate a set of behaviours, without understanding their underlying motivations. 
When this happens with data-science, organizations will emulate the _technology_ behind data science, without creating a data-driven organizational culture.
I see this sort of issue pop up from time to time, and I'd like to share my ideas on how to _avoid_ its emergence.

![Cargo cult symbol, credit to Tim Ross, https://commons.wikimedia.org/wiki/File:JohnFrumCrossTanna1967.jpg](https://upload.wikimedia.org/wikipedia/commons/thumb/3/37/JohnFrumCrossTanna1967.jpg/320px-JohnFrumCrossTanna1967.jpg)


Data science can be an immensely powerful tool, and provides two things to organizations:

 - Automated decision making
 - Improved manual decision making.

While many organizations _recognize_ the importance of data science, many of them fail to build effective data science capabilities.
I believe that this is because of a fundamental misunderstanding of data science, which causes them to implement the _technical_ parts of data science, without considering its human side.

This is the same cargo-cult problem that appears in other fields.
In this post, I'll talk about what I see as the best way to _avoid_ cargo-cult data science, and on how to deliver more value from a data science investment.

A typical data-science project
==============================
Too many data science projects follow this trajectory, common to other IT projects:

 - They start out exciting, and full of promise, with strong buy-in from upper management.
 - Initial prototypes look promising, and the project appears to address an important organizational issue.
 - The project starts to slip, failing to deliver on promises.
 - At the same time, the organization starts to loose interest in the project, and it begins to falter.
 - The project doesn't fail, but doesn't deliver the organizational change it promised.

This is particularly problematic for data-projects, because they're typically meant to introduce new forms of management or organizational behavior.
In contrast to many traditional IT projects, which represent improvements in existing processes, data science projects often aim to _change_ the way an organization behaves.

**Why did this project fail?** Most people, especially data scientists, would focus on either the technical execution, or blame management for abandoning it.
However, I believe the failure occurred earlier, when the initial design failed to account for how it would fit into the organization.


![A failed data lake]({{ site.url }}/assets/data-science-orgs/jetty-landing-stage-sea-fog.smol.jpg)

The human side of data science
==============================
In my experience, a "data driven organization" is about far more than the existence of analytics and measurement.
Fundamentally, to be a data driven company, data needs to be part of the internal dialogue spoken by all members.

This is in direct contrast to the project described above, which focused on technology over objectives, a classic sign of a cargo cult.
For example, take the all to common "enterprise data lake project". 
Many organizations now see some sort of data warehouse as a necessity, but the project itself can never deliver a culture of analytics within the company.
The problem here is that the project is viewed in _technical_ rather than _human_ terms. 

Notice that we have a very clear idea of the technical side of this project. 
There's a good chance it involves:
 
 - A large SQL database
 - An ETL tool
 - A frontend, perhaps tableau or similar
 - And dozens of reports that managers don't read.

In contrast, a good data project would be described as "the tool we use to pick phone plans to sell".
Here we know far more about where it fits within the organization:
 
 - We know the organization is probably a telco
 - And that the tool is specifically for their retail operations
 - We know that the tool puts forward recommendations for things to sell
 - It presumably monitors the outcomes of sales and optimizes itself. After all, it's a data project.
   It probably uses a data-warehouse to do so, and maybe it reports with tableau.

So, to avoid a cargo cult of data, organizations should stop chasing _technology_ and start working with experienced technologists who can apply technology to solve organizational problems.

A successful data project
=========================

How might the phone plan tool above come about?

 - The telco engaged an experienced strategist with data experience
 - They identified above average churn in phone plans, and that it was a problem that could easily be tackled with data based tools.
 - They worked closely with the following stakeholders:
   - Executives, to understand how their project relates to company goals, and how success would be reported.
   - Retailers, to understand their needs
   - Retail managers, to understand why retailers are being pushed to oversell plans.
 - With these in mind, they design a small technical MVP that recommends plans and monitors for feedback. Nothing too complicated
 - And finally, they ensure that the retailers work towards the "churn optimal" goal, rather than the "sell the biggest plan" goal.

This project's best feature is that it identified a specific sub-part of the organization, with a clear problem, and then built a solution for it.
It's important to try to find an organizational capability that can be easily partitioned off and then viewed through the data science lens, as I'll explain later.

Spreading data science
======================

I used to think data science would take over every organization, once it was introduced. I was very naive.
My reasoning was simple: anyone with data science on their side would be able to _prove_ that their efforts worked better than their peers.
That would be rewarded, and data-science would quickly become an essential internal requirement for everyone, due to competitive pressure.

> I used to think data science would take over every organization, once it was introduced. I was very naive.

However, that assumes that someone presenting an analytical presentation will be viewed more favourably than someone presenting something softer.
Basically, I had assumed a data-driven culture exists, when in reality businesses are struggling to create that culture in the first place.

That's why the recommendations above talk about picking a small, somewhat isolated part or aspect of the organization.
Doing this makes it easier to move that sub-group to being data driven and analytical.
It also minimizes the problematic boundary between the data-driven subset and the rest of the business, making it easier to view that subgroup according to their metrics.

The other advantage of the "minimal boundary" is that the organizational hierarchy will tend to push the data science down into the subgroup.
With their bosses demanding analytical results, managers will demand analytical results from their peers, and so on, down throughout the subgroup.

Conclusion
==========

Data science is best viewed as a form of company culture, rather than a set of technologies.
However, many firms will try to create that company culture by acquiring data-science technology, rather than working on their culture.

This is understandable: company culture is notoriously difficult to change, but data science is also a little bit different from other cultural matters.
I argue that it's best to spread a data-driven culture from the top of an organization down, by requiring that reports be analytical.
Doing so then means that organizational subunits must become data driven, which causes their sub-units to become data-driven, and so on.
For companies trying to increase their analytical strength, this suggests that solutions that _start_ with technology ("we'll build a data warehouse", or "let's get AI") are likely to fail.

Solutions that help measure and improve the performance of a part of the company ("we'll help you measure marketing ROI", or "we will introduce predictive maintenance), will spread and become enduring organizational strengths.

So in summary:
 
 - Look for solutions that focus on the organization, not the technology
 - In fact, I'd be a little skeptical of tech-first solutions - are they really solving your _real_ problem?
 - And then apply them to problems that can be "boxed up" so that the data-driven solution is actually used to evaluate that problem.




