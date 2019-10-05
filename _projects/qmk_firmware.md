---
layout: project
title: 'qmk-firmware'
date: 25 Dec 2018
# image: /assets/icons/qmk_icon_48.png
screenshot: /assets/icons/qmk_icon_48.png
# screenshot: /assets/img/projects/hy-drawer.svg
links:
  - title: Website
    url: https://qmk.fm 
  - title: Source
    url: https://github.com/ericgebhart/qmk_firmware
caption: Quantum Mechanical Keyboard Firmware
description: >

  Configuration for my Ergo-dox, dactly-8t, verterbi, and xde-75 keyboards.
  This is a huge project which supports many keyboards.  The architecture of the project
  is such that anyone can create their own layouts and integrate them into the project with
  a pull request. Personal code should be in folders named after your github account.
  My code can be found in /keyboards/ergodox-ez/layouts/EricGebhart  and in /users/EricGebhart.
  It is in 'C' and is nicely organized. The layout code is very easy to read, all because of
  the nice code in /users/EricGebhart/.
  
  Some features of this layout are base layers of dvorak, bepo, qwerty,
  norman, colemak, and workman.  additional layers are a symbols and
  numbers layer with a keypad, A mouse, arrows, and media
  layer for navigation. The mouse layer means I don't need a mouse, and it works great.
  
  There is a single key which toggles through all the base layers. Although I usually leave it
  in dvorak or bepo.
  
  Finally there are some interesting things which happen when keys are held down.
  the thumb keys have backspace and delete on one side and space and enter on the other like
  a [kinesis keyboard](https://kinesis-ergo.com/shop/advantage2-dvorak/). 
   * If held down those 4 keys are control and alt so they work with either hand. 
   * The home row pinky keys are <shift>.
   * The index finger home-row turns on the symbols and numbers layer.
   * The second finger home-row turns on the mouse and arrows layer.
   * Escape is on the thumb key but is windows/command if held down.
   * The home key is home, but end if double tapped.

accent_color: '#4fb1ba'
accent_image:
  background: 'linear-gradient(to bottom,#193747 0%,#233e4c 30%,#3c929e 50%,#d5d5d4 70%,#cdccc8 100%)'
  overlay:    true
---
