---
title: "Week Log: Entry №1"
date: 2019-04-27T07:35:51-05:00
showDate: true
draft: false
tags: ["compilers","log","website"]
---

Welcome to the first of the week log posts, wherein I go over what I got done this week regarding
programming and such. One of these will be posted each Saturday or Sunday, depending on my schedule.

First off, the compiler:

* I've (mostly) completed messing about with the syntax for now, I quite like how its turned out.

* The parser is complete, though it took some time to get there. I started with a state-machine based
  approach that was more complex than necesarry, so I swapped to a stack-based approach which *also*
  ended up being more complicated than necesarry. Ultimately, I ended up just doing a typical recursive
  descent configuration. It works well, but I'm glad to have experimented with the other methods - its 
  never bad to try something new, even it doesn't work out. That's what `git reset` is for, after all.
  
* The compiler source code is now up on GitHub, but not public yet. I would like to get more done and at
  least be compiling basic binaries before I make the code public, which I hope won't take too much longer.
  It depends on whether or not I go for LLVM as a backend or decide to do a custom x64 implementation. I may
  elect to do a hybrid for a while, since work on a custom backend would be dreadfully slow. 
  
Secondly, the site:

* I updated the site this week to this new theme, which you find [here](https://themes.gohugo.io/hugo-theme-sam/).
  I'd been meaning to do this for a while, but hadn't gotten around to it until now.
  
* A blog post about recursive data types in Rust was posted. Writing that post was even a bit of an eye-opener for me,
  as I had never really read the docs for some of the container types until I went to write it. Whether anyone else finds
  the post helpful or not, I certainly learned something in the process - never a bad thing.
  
* I've started porting over some of the old posts from the previous site to this one. This might take a bit, but sooner or
  later they'll all be up.
  
That's it for this week. In the future, I may consider adding in reading and music activities and thoughts here as well,
to give it a bit more of a personal touch, but I'll limit it to programming for now. As ususal, you can always contact me 
at `sjdh AT sjdh DOT us` or find me on [keybase](https://keybase.io/sjdh02).


