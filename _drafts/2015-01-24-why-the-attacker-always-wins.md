---
layout: post
title: "Why the Attacker Always Wins (And What We Can Do About It)"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Introduction
====

Opening- specific news story or overview of attacks in the past
1-2 para

What do we know from all of this? That it's hard to get security right.

In fact, we can't get it right, unless we make some changes.

Computability 101
====

What it means to be computable, Turing thesis, etc.

Types of languages- regular, context free, Turing complete.
Undecidability of TC languages.

So what?
====

Programs accept TC input.

Programs _execute_ in accordance with TC input.

But we can't know if a Turing Machine does exactly what it is intended.
If your input data format is a TC language, then you're fucked.

What do we do?
====

Thankfully, there are subsets of languages which we _can_ make rigorous
claims about. In particular, CFG's and regexes.

If the input language is one of the two, then we can _prove_ 

