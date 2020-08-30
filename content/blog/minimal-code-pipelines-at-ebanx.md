+++
title = "Minimal code pipelines at EBANX"
date = 2020-03-31
template = "blog_post.html"
[taxonomies]
tags = ["software engineering"]
+++

EBANX, as a fast growing company, faces a lot of challenges. 
Some of them are related to that old balance on how much 
we will take from the outside and how much of our problems 
will be solved by in house products.

Recently we had a new demand coming to the IT team about 
how we manage some of our compliance flows. 
The responsible team decided we should have an in house 
alternative (or extension) to some service providers.
And that can actually be quite a task.

At EBANX we strive to deliver services that will allow 
people access to services that are not immediately available 
to the general population for several reasons, most of them 
deriving from an in development economy and low banking maturity. 
Some of the services include: integration to common ecommerce 
platforms, a refunds platform, a prepaid credit card, 
payment processing for companies and individuals from Brazil 
and outside that want to painlessly offer services or products 
in the Latim America market, and the list goes a little bit longer.

But let’s go back to that compliance demand. 
This is a challenge in several different ways, starting from 
the structure of the team. EBANX is made of some major areas 
and if we want to integrate products of them all we will need 
collaboration and developer time from all of them. There is no 
helping it. These developers would also probably not be the 
newest ones, we would need experienced people to make some 
integrations. More often than not, this precious workforce 
cannot be redirected from their home responsibilities. 
And surely, as the main team behind the project we will 
have to live with some rotation of these developers. 
Again, there is no helping it.

Even if they were to be with us for a longer period of time 
they would end up distancing themselves from their home projects, 
losing their close knowledge of how things work there, 
and by consequence losing that precious value they have for us.

Surprisingly, this factor that is not a technical one is 
one of the big reasons behind our system architecture.

Some of our technical requisites are:

 - Being able to integrate all of EBANX’s products, coming 
 from all the different areas, and more to come
 - Being able to integrate several external providers, and more to come
 - Work with the above mentioned personal situation
 - Extensibility for business logic
 - Initially, having a flow easy to understand by non-developers
 - Futurely, having a flow configurable by non-developers

Let’s dive then on a system architecture that can comprise 
all of these worries.
The solution

Since the beginning the idea of this flow, including the concept 
of how the components and the pipeline should work, seemed to us 
like a good application for a Finite State Machine, 
and that was a POC we would like to test.

First of all we defined as an application the fact that a given 
company or person would like to use one of EBANX’s products and 
therefore would have to be evaluated by the Compliance Team 
auxiliated by our solution.

In a finite state machine the state of the processing is kept 
on the concept of a state, where this state can be as simple 
as an integer or string field in a struct representing that 
application. Progress in the processing of that application 
would be signaled by a signal or input that together with the
current state would define the next state for that application.

And a simple table would be capable of describing the whole 
flow of providers an application of a given type would pass by.

But we wanted to try something even simpler. Not that these 
concepts would be hard to explain to another programmer, but 
there was no need not to try and make things even simpler 
given our team’s multi area coordination challenges.

We have since then a degenerated state machine. There is no concept of signal or input, a single mutate(application_id, current_state, next_state) function is exposed to the components as a way of changing state and denoting progress. This is the minimal concept we got as a way of coordinating our work. And of course, we kept that idea of a nice looking table to demonstrate our flow.


```Python
pipelines['product_pipeline_1'] = OrderedDict([
   ('000_PIPELINE_START', mutate_step('000_PIPELINE_START', '000_COMPONENT_1')),
   ('000_COMPONENT_1', Component1.process),
   ('999_COMPONENT_1', mutate_step('999_COMPONENT_1', '000_COMPONENT_2')),
   ('000_COMPONENT_2', Component2.process),
   ('999_COMPONENT_2', mutate_step('999_COMPONENT_1', '999_PIPELINE_END')),
])
```


Let’s walk through some other concepts here. There are components in the software that are not related to any integration with compliance providers, they are responsible for receiving applications through an API or collecting them from the CRM systems used throughout EBANX. These components will register these applications in the initial state 000_PIPELINE_START.

From that point on the system needs someone to call a /mutate endpoint that will gather all applications that are in one of the states known by the state machine, and apply on them the defined function. Perceive also that there is a mutate_step function that simply mutates things from one state to another giving progress to the pipeline.

Let’s expand on this point for a bit to explain something. You may ask yourself: “So, do you want to roll out a system that depends on a cron to work?” Well, maybe. At first, for communication with the external world our system uses AWS SNS. An AWS Lambda can be plugged on the SNS to call /mutate on every notification of state change providing a feedback to the state machine and a faster processing flow to the system. Without sacrificing our structure and giving us plenty of space for maneuvering on how we want the system to work and when we want it to work.

But back to the pipeline components: every pipeline component has at least two states. A 000_start and a 999_done symbolizing that one application is ready to be processed by that component, and that the component has done the job it should, respectively. It can, and should have more if that fits it’s inner workings, but that is completely up to whoever is designing that component.

If a component depends on an asynchronous response from the component that would work like this:

1. The code that send the request mutates to a state unknown 
(and by consequence, not managed) by the FSM
2. The endpoint that receives this response should mutate 
back to a state inside the pipeline

But there are advantages to working with APIs that allow a 
pooled approach. Something like the following would be preferable:

1. The function handling the 000 state sends a 
creation request, and mutates to a 100 state
2. The function that handles the 100 state requests 
the result. If it is pending stays on 100 for a future 
retry, if not moves on to 999

The point is that this structure is generic enough for 
anything we have tried so far. It would not be difficult 
to convert this into a fully featured finite state machine 
where the components use a signal API instead of calling 
mutate directly. But that adds little value to us. Surely 
the mutate function gives to the component too much power 
allowing it to change the state indiscriminately, but we 
solved this internally with a simple whitelist on what state 
changes would be allowed and keeping track of this policy 
with code review.

We may still see downsides on all of this, but so far 
it has worked and has been more of a tool than a nuisance 
for us.