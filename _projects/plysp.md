---
layout: project
title: 'Plysp'
date: 30 Jan 2019
image: /assets/img/projects/hy-push-state.svg
screenshot: /assets/img/projects/hy-push-state.svg
links:
  - title: Source
    url: https://github.com/EricGebhart/plysp
caption: A lisp written in python with some features of clojure.
description: >
  I had just finished doing a small python project which did some data analysis using pandas and
  scipy. It was the first python I had written in a while and I realised I missed programming
  in lisp. But pandas and scipy are kind of fun. So what if I had a lisp that could use
  python libraries ?
  
  I searched around and found various toy lisps in python. Including a complete port of clojure
  which turned out to have poor performance. I also found Hylang which isn't really a lisp IMHO.
  
  So I decided to write one that had scoping, namespaces, python interop, and which might be
  performant if I turned the the internals into Cython.
  
  I used _ply_ for the grammar processor, mostly because I read somewhere that it was impossible
  to write a lisp with a BNR grammar. It seems to be working ok. But it's not done yet. So
  maybe there will be a road block somewhere at which point I'll swap out the grammar.

  This is still a toy. But scoping, namespaces and interop is working. It does
  have immutable data as well. No STM, that would be cool. It would be nice for this to
  handle concurrency and threads as easily as clojure and elixer.
  It still has the macro bits and destructuring into the environments to go.  
  Its still a fun project. The REPL is basically a calculator.

accent_color: '#4fb1ba'
accent_image:
  background: 'linear-gradient(to bottom,#193747 0%,#233e4c 30%,#3c929e 50%,#d5d5d4 70%,#cdccc8 100%)'
  overlay:    true
---
