---
layout: post
title:  Coq tactics (2)
date:   2020-05-04 22:25:30 -0700
categories: 
    - Programming
---
# A Few More Handy Tactics

+ clear _H_: Delete hypothesis _H_ from the context.
+ subst _x_: For a variable _x_, find an assumption _x = e_ or _e = x_ in the context, replace x with e throughout the context and current goal, and clear the assumption.
+ subst: Substitute away all assumptions of the form _x = e_ or _e = x_ (where _x_ is a variable).
+ rename... into...: Change the name of a hypothesis in the proof context. For example, if the context includes a variable named _x_, then rename _x_ into _y_ will change all occurrences of _x_ to _y_.
+ assumption: Try to find a hypothesis _H_ in the context that exactly matches the goal; if one is found, behave like _apply H_.
+ contradiction: Try to find a hypothesis + in the current context that is logically equivalent to False. If one is found, solve the goal.
+ constructor: Try to find a constructor _c_ (from some Inductive definition in the current environment) that can be applied to solve the current goal. If one is found, behave like _apply c_.