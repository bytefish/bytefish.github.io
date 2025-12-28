title: Creating State Machine Diagrams
date: 2025-12-28 12:21
tags: angular, apps
category: apps
slug: fsm_designer
author: Philipp Wagner
summary: Introduces a tool for designing State Machines.

In this article I am going to introduce fsm-designer, which is a small Angular application 
for creating State Machine Diagrams. It also serves as a quick introduction for modeling 
systems as State Machines.

All code can be found in a Git repository at:

* [https://github.com/bytefish/fsm-designer](https://github.com/bytefish/fsm-designer)

## Table of contents ##

[TOC]

## What are State Machines? ##

A useful tool for implementing and visualizing systems are so called State Machines. It's 
a very simple way to think of your system as so called *States*, *Transitions* and 
*Actions*. I think it's best unterstood using an example.

Let's say your application has just been started and the user is greeted with login screen. In 
terms of a State Machine our application is now in a *State* called `Waiting for Login` or 
similar. 

The user then enters their credentials and clicks the login button. In the background our application 
fires requests to a server and the user is redirected to a Dashboard. In State Machine lingo we could 
say: "The application *transitions* from *State* `Waiting for Login` to *State* `Logged In` by an 
*Action* clicking on the Login Button.

And that's all there is. 

Of course the mathematically correct definition is slightly more complicated:

> A finite-state machine (FSM) or finite-state automaton (FSA, plural: automata), finite automaton, 
> or simply a state machine, is a mathematical model of computation. It is an abstract machine that 
> can be in exactly one of a finite number of states at any given time. The FSM can change from one 
> state to another in response to some inputs; the change from one state to another is 
> called a transition.

## What we are going to build ##

If you are going to implement a State Machine, there are a lot of useful libraries and countless 
of applications. But I always felt them to be overly complicated for simple diagrams. So I've 
developed *fsm-designer*, which is a small Angular application to visualize state machines:

<div style="display:flex; align-items:center; justify-content:center;margin-bottom:15px;">
    <a href="/static/images/blog/fsm_designer/fsm_designer_screenshot.jpg">
        <img src="/static/images/blog/fsm_designer/fsm_designer_screenshot.jpg" alt="State Machine Designer">
    </a>
</div>

## Trying it out ##

You can use the fsm-designer here:

* [https://www.bytefish.de/static/apps/fsm-designer/index.html](https://www.bytefish.de/static/apps/fsm-designer/index.html)

It does not send any data to a server and does not execute any network requests. 

