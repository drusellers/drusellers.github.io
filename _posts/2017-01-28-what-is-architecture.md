---
layout: post
title:  "What is Software Architecture?"
date:   2017-01-28 05:58:21
categories: Architecture
published: false
disqus_id: 150e5c91-183a-42a5-8a03-a60e01b20851
---

Architecture. Such a grand term. It evokes sweeping views of buildings, the intricate plans of a building. Wiring diagrams and blueprints. Its a romantic idea, and one I would love to be a part of. The trick is, I build software systems.

Software systems aren't known for being particularly grand. Most people, when they run into a software developer talking about software architecture, often run into someone that wants to build something. In order for their architecture to be achieved an amount of effort must be spent _building_ the underlying framework.

That is one of the areas that I disagree with, that underlying framework, that is "infrastructure". Architecture should instead be thought of as designing the system. Its the noodle power that goes into it before you begin to build. Its asking hard questions and exploring areas of the business that might be "no brainers" to others.

When I start working at a new company I tend to ask a lot of questions. I'm going to try and reproduce those here as a means of "making the implicit, explicit".

One of the first things that I want to do when I'm working for a company is understand what their "Operating Models" is as defined in [Enterprise Architecture as Strategy](https://www.amazon.com/Enterprise-Architecture-Strategy-Foundation-Execution/dp/1591398398). This book presents four operating models: Unification, Coordination, Replication, and Diversification. This is a high level idea, and it will take longer to read the book and digest its contents then it will to determine a company's operating model. Its a simple concept, that can radically change how we approach system selection and design.

From there I like to discuss what the "Value Chain" of the company is. How does my company "create value"? Sometimes I get nerdy about it and try to describe it as an F# function signature. I think what are my "inputs" into the business function? `(input: ?) -> Output`. I find it humorous how simple this function tends to be in any business as well. At a previous employer we came up with `(account: Account) -> (principal: Money) -> (fedRate: InterestRate) -> (schedule: PaymentSchedule) -> (fundingSource: FundingLoan) -> Loan`. We boiled down the entire loan making process to 6 key elements. Who are we making the loan to? How much is the loan for? At what interest rate is the Fed offering money at? When was the loan due? How were we going to fund it? Given those things we could generate a loan.

That was it. _Everything_ else the bank did was about speeding that whole process up. The bank had established many additional and common business processes to operate at its level in the business, but those processes served only one goal. To make the above process faster and safer. The core of the business was still the same, with or without those extra processes.

Lets try this again, at my latest job we came up with something very similar. We call it the "4 box model". It looks roughly like this for the majority of the company.

<pre>
------------------------------------------
| Product | Commerce | ERP | Fulfillment |
------------------------------------------
</pre>

And in my nerdery, `(ProductInfo) -> (SalesOrder) -> Sale`. In the end you have to have product info first, then you can pass in a sales order which will product a sale. All of the ERP tooling and fulfillment tooling that we have simply accelerate this process. But the reality of the situation is that there is a lot of capital flowing through those last two boxes so we expanded it from my handy F# style definition to one a bit more expanded.

Once I have a model like the two above, I circulate this model to other people in my organization. I validate my model and let people challenge its construct. Usually, in that process I will uncover something that forces the model to be tweaked, and sometimes it needs to be rebuilt. Usually, this only takes a few test runs on people before you settle on something that resonates with people.

## Ok, so now what?

With this model in hand, I follow a basic template that has worked well across multiple organizations. I make a code repository for each part of the business function and start shuffling code into those buckets as business velocity allows. What I mean by that is I don't stop what I'm doing and go sort the code into these buckets. Instead I have created a space for this code to live and as I touch various systems I start to move and consolidate that code into the buckets.

This slow sorting approach gives me and the other contributors time to absorb those changes. Time to ask hard questions about what is going on and why we are doing it. With each move things generally become clearer and service boundaries become clearer. By far the best conversation at the development level is always about dependencies, and how much more obvious they become. In the Retail world people were surprised how many connections the Product section had, but on reflection it made a lot of sense, because it is the base of every other system.





---

### Drafty stuff

At my latest job, I've been exploring a type of architecture that requires little pre-established building. It exists as little more than an idea. The idea is that we need to group software together. Ok, nothing too genius there. We have been grouping software together for a long time. We even have a word for it: _cohesion_.

We want like bits to be together. But what makes something, _like_ something else. Well, lets take a look at the system I am working on. Its a retail system involving the sale of motorcycle parts. Let's start at the highest level of the business. It looks roughly like this.





So, we started our architecture with a simple idea. We need for application buckets: Product, Commerce, ERP and Fulfillment. With those for repositories we can build the entire company, including how we integrate with our third party software. Each box above has one major system that is backing it and my work is about integrating those systems together to behave as one healthy system.
