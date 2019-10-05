---
layout: project
title: 'Xmonad-setup'
date: 25 Dec 2018
image: /assets/img/projects/hy-push-state.svg
screenshot: /assets/img/projects/hy-push-state.svg
links:
  - title: Website
    url: https://xmonad.org/
  - title: Source
    url: https://github.com/EricGebhart/xmonad-setup
caption: My Xmonad configuration
description: >
  I am not a strong haskell programmer, but I've been using Xmonad for several years now.
  It is by far the best window manager I have ever used.
  
  This setup has the panel from xfce, dmenu and denzen.
  This configuration uses grid select for many things but also has hot key selections
  and context sensitive help for sub-menu hotkeys. 

  The context sensitive help is unique but also very helpful to remember all of the
  possible hotkeys and what they do. 
  I wrote it completely in awk, It uses denzen to display the help.

     * workspaces with autoloading applications.
     * named scratchpads.
       * termite terminals
       * bc - calculator
       * ghci - for haskell development
       * htop
     * Hot key Search
       * man
       * hackage
       * wikipedia
       * wiktionary
       * reverso
       * arch linux packages and AUR.
       * many others.


accent_color: '#4fb1ba'
accent_image:
  background: 'linear-gradient(to bottom,#193747 0%,#233e4c 30%,#3c929e 50%,#d5d5d4 70%,#cdccc8 100%)'
  overlay:    true
---
