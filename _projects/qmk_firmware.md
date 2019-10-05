---
layout: project
title: 'qmk-firmware for my keyboard'
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


  This is the firmware for my Ergo-dox, dactly-8t, verterbi, and xde-75 keyboards.
  This is a huge project which supports many keyboards.  The architecture of the project
  is such that anyone can create their own layouts and integrate them into the project with
  a pull request. Personal code should be in folders named after your github account.
  
  &NewLine; 
  
  My code can be found in */keyboards/ergodox-ez/keymaps/ericgebhart*  and in */users/ericgebhart*.
  It is in 'C' and is nicely organized. The layout code is very easy to read, all because of
  the nice code in */users/ericgebhart/*.

  &NewLine; 
  
  Some features of this layout are base layers of 
   * dvorak
   * bepo 
   * qwerty
   * norman
   * colemak
   * workman  

  &nbsp; 
   
  Additional layers are: 
   * Symbols and number-keypad with F-keys
   * Mouse and arrows with media keys and F-keys
   * A layers layer, for experiments.

  &nbsp; 
   
  Thumb keys, the bottom row, and edge keys are the same in all
  layers so the arrows, the tab, ctrl, alt, delete, shift, tab all
  stay put while the interior keys change.

  &nbsp; 

  The mouse layer means I don't need a mouse, and it works great,
  The home row is the directions, the row below is the scroll wheels and the row above is
  all 5 buttons. The buttons are repeated on the other hand as well, for drag and drop.
  Although I use the mouse very little by the grace of Xmonad.

  &nbsp; 
  
  There is a single key which toggles through all the base layers. Although I usually leave it
  in dvorak or bepo.

  &nbsp; 
  
  Finally there are some interesting things which happen when keys are held down.
  the thumb keys have backspace and delete on one side and space and enter on the other like
  a [kinesis keyboard](https://kinesis-ergo.com/shop/advantage2-dvorak/). 

  &nbsp; 

  If held down:
   * the 4 big thumb keys are _control_ and _alt_ so they work with either hand. 
   * The home row pinky keys are _shift_.
   * The index finger home-row turns on _the symbols and numbers layer_.
   * The second finger home-row turns on _the mouse and arrows layer_.
   * _F-keys_ are available on both the symbols and mouse layers.
   * _Escape_ is on a thumb key but is _windows/command_.

  &nbsp; 

  I have a few double or multi-tap keys, but _home/end_ is the only one I use very much.
   * The home key is _home_, but _end_ if double tapped.

accent_color: '#4fb1ba'
accent_image:
  background: 'linear-gradient(to bottom,#193747 0%,#233e4c 30%,#3c929e 50%,#d5d5d4 70%,#cdccc8 100%)'
  overlay:    true
---
