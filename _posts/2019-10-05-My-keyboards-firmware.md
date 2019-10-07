---
layout: post
title: "My keyboard's custom firmware"
description: "The custom firmware I use for my computer keyboard"
date: 2019-10-05
category: code
tags: [C, ergodox, ergonomics, QMK, keyboard]
---

# My keyboard's custom firmware.

I bought my [ergodox-ez](https://ergodox-ez.com) a few years ago because I wanted 
something easier to carry to the cafe than my 
[kinesis ergo keyboard](https://kinesis-ergo.com/shop/advantage2-dvorak/).

These days I take my ergodox with me everywhere I go. The best thing about it, other than the 
thumb keys is that it uses the QMK-firmware for it's programming. 
In this article I'm going to talk about how I ended up programming
the firmware for myself and where to look and how to get started if you want to do that too.
Along the way, I'll explain some of the ways my keyboard is programmed to make typing and 
using a computer easier.

I ended up learning all of this because I needed a portable keyboard that had a dvorak 
layout as close to my kinesis as I could get it.
At the time, the [Online keyboard configurator](https://config.qmk.fm/#/1upkeyboards/1up60hse/LAYOUT_60_ansi) didn't work for me. So I did it all manually. I'm still happy I did.

*NOTE:* Ergodox has a [configurator](https://ergodox-ez.com/pages/oryx) now and 
they are working on auto flashing your keyboard, so no coding would ever be involved 
in reconfiguring your keyboard. I have not tried it. But it looks cool, it is all
based on QMK.

The [QMK-firmware](https://qmk.fm) is how the ergodox is programmed. 
QMK supports [many keyboards](https://qmk.fm/keyboards/) which makes it a lot of fun if you 
like trying out different keyboard layouts and sizes. Which I like.

I'm a little bit of an ergonomics nut. I mostly stick with split keyboards with lots of
thumb keys.  But I'm also currently playing with a 
[virterbi](https://keeb.io/products/viterbi-keyboard-pcbs-5x7-70-split-ortholinear?variant=1302704554014)
and an 
[xd75](https://kprepublic.com/collections/xd75/products/xd75re-xd75am-xd75-xiudi-60-custom-keyboard-pcbkeyboards) which was a super nice keyboard to build, no soldering, just order 
the parts, PCB, key switches, key caps and plates and snap it together.
I'm building a [dactyl](https://github.com/adereth/dactyl-keyboard)
which I like better than anything I have at the moment.

But I digress.

Back when I bought my ergodox the QMK-firmware was a bit messier, but I managed to get 
it programmed to behave just like my kinesis keyboard. It wasn't too special. Just dvorak 
with a few extras. I used it off and on, but never really liked it. It's just not as 
comfortable as my kinesis. Here is the layout, more or less, minus the _F-keys_.

*The kinesis keyboard with dual legend keys. Dvorak / Qwerty*
![dvorak layout for kinesis](dvorak-kinesis.gif)

A few years ago I started travelling. I needed a portable keyboard and like it or not
the ergodox was my best choice at the time. Realizing that I needed to get comfortable with 
it I decided to revisit the firmware and see what I could do. The QMK-firmware had
changed architecture, and to do it well, my layout needed to be completely refactored and
I still had no real idea how I was going to make my layout better. But because I was
typing in french a lot I did know I wanted to be able to type with a 
[bepo](https://bepo.fr/wiki/Accueil) layout in addition to my 
[dvorak](https://en.wikipedia.org/wiki/Dvorak_Simplified_Keyboard) layout and
that there must be other things I could do with all 
[the cool features](https://docs.qmk.fm/#/features) in QMK.

Here are some of them.
 * tap or hold, the key does one thing when you tap it and another if you hold it.
   Repeated key for the tap value still work with a double tap.
 * Sticky keys, for when you don't want to hold something down for the next key.
 * Key combinations on one key, for example; I have keys for _Control-c_ and _Control-v_.
 * Mouse keys.
 * auto-shift on hold
 * tap-dance
 * Up to 31 different layers
 * backlight color control.
 
 There's even more once you start digging around in other people's layouts and you see
 all the creative ways they've used them.

So the way to reprogram the ergodox is to fork the QMK_firmware. I needed a new
version so I restarted with a fresh fork and reread some documentation, and started
salvaging my old layout into a better version which I could more easily play with.
I looked around at other configurations and found some examples that were clearly
better than the rest, and I used their ideas.

*To start I had one thing I knew I wanted.*

I had some remappings I had made on the kinesis. I was accustomed to
having the _End_ key on my left thumb as _Escape_. At the same time I had been using 
[Xmonad]( https://xmonad.org) as my window manager and I really wanted an easier to 
touch _windows/command_ key.  It's location was ok on the kinesis, but not great.

## Into the rabbit hole.

With the new architecture changes, and new desires for my keyboard I was now ready for
round two with the QMK-firmware. 

The QMK_firmware is big. It's a lot of code when you first 
look and it's overwhelming. It really pays off to read the documentation a few times, poke at
the code a little, search for how other people used some feature you just discovered, 
then read some more. The good news is that you don't need to worry about most of the
code that's in there. That might come later.

The [documentation](https://docs.qmk.fm/#/) is really good. Read it a few times. 
Especially the API doc under [features](https://docs.qmk.fm/#/features) .
Let it soak in. Don't worry about getting it all.

I'll explain enough that you can start looking
around and understanding what you are looking at. The important bits are actually pretty easy
to grok. But there are features that will allow you to create keyboard functionality
that might blow your mind. Like an encrypted messaging layer, or a layer for irc messages 
you send all the time, or a layer for gaming, or even for a specific game. 

I leave it to you to discover what you can do. But I'll get you started with an overview.

### first things first. Where do I put my stuff ?

You have a github account and you have a keyboard, those are what determine where your
stuff goes. Aside from the documentation the only two places you
*really* need to know about is where to put your layout's keymap and
where to put your code. This where I deviated from the doc just a bit,
I put almost everything in my _/users_ directory the same as a number
of other people who have more elaborate setups than I do which support
more than one keyboard.

There are only two places to worry about, the rest of the code you can ignore for now.
The good news is that everyone else puts their stuff in the same two places, so you'll be able
to look around at what everyone else has done. 

QMK is structured so there is a place for all your code. it goes into 
`users/<insert your github account name here>`

My code is in *users/ericgebhart*.

Which looks like this.
```
users/ericgebhart
├── config.h
├── ericgebhart.c
├── ericgebhart.h
├── flash-ergodox
├── readme.md
├── rules.mk
└── switch-kbd
```

The layouts go into a specific directory dictated by the hardware keyboard you have.
Many people still put everything here. I don't recommend doing that, it's a pain. 
`keyboards/<your keyboard>/keymaps/<github account>/`

The one for my ergodox can be found in
*keyboards/ergodox_ez/keymaps/ericgebhart* 

```
keyboards/ergodox_ez/keymaps/ericgebhart
├── keymap.c
└── readme.md
```

I'm currently playing with a _viterbi_ and a _xd75re_ while I'm building a dactyl,
So those will have other directories to go into. Just follow the path to your your keyboard.

#### A little about hardware

If you want to play and the ergodox isn't your thing, the
[xd75re](https://kprepublic.com/collections/xd75/products/xd75re-xd75am-xd75-xiudi-60-custom-keyboard-pcbkeyboards) 
is really nice and was super easy to build, I wish my now ancient ergodox had backlighting. 

Of course there is the whole education about keyswitch colors and brands. 
Do you want [Cherry Mx Browns](https://www.cherrymx.de/en/mx-original/mx-brown.html), 
or do you prefer [Gaterons](https://mechanicalkeyboards.com/shop/index.php?l=product_list&c=77)
Or [something else](https://mechanicalkeyboards.com/shop/index.php?l=product_list&c=107)
As well as the profiles of the keycaps, sculpted or flat, DSA, SA? or something else. 
[pimp your keyboard](https://pimpmykeyboard.com)! The many choices we
have with a Mechanical keyboard are awesome.

Building a mechanical keyboard is a very personal thing. And even if you didn't build it
you can still change out keycaps and sometimes even the key switches

### A small example.

One of first things you learn about QMK is that all the keys have names like _LGUI_ for Left GUI, 
which is left _windows/command_, or _KC_ESC_ which is Escape, and that there are functions 
like `GUI_T()` which allow you to define new keys like this one:

`#define GUI_ESC     GUI_T(KC_ESC)  // Gui or escape`

This new key _GUI_ESC_ is a key that you can hold for a _GUI_ key and tap for _Escape_ 

Already my ergodox was nicer than my kinesis.

So that new definition goes in `users/<you>/<you>.h`
If you need to write some code that goes in `<you>.c`.

_config.h_ is some reasonable system wide defaults, and _rules.mk_ is 
for your build.  You'll also want a _README.md_. Anything else is up to you.

In your `keyboards/<your-keyboard>/layouts/<you>/keymap.c` you'll have your
layout definition, and you'll probably want a little bit of a _readme.md_ right
there too.

Another problem for me with the kinesis was the _control_ and _alt_ keys. All the places
I could put them were hard to reach. I found my solution in someone elses layout.
They had changed their _space_ to be _control_ when held. But with the thumb keys 
I had _space_, _enter_, _backspace_ and _delete_. Voila! 
I have _Control_ and _Alt_ on both thumbs and on the easiest keys to reach. 

### Consistency is key, ha.

This stuff can get confusing fast. So many keys with different layers. It's easy
to create a keyboard that isn't particularly usable. A key can disappear on different
layers, or maybe that key does what you want on one layer, but then it needs to do
the opposite on another. It's pretty easy to add a layer that gives you no way to 
get out. You'll see a lot of layouts have a special layer that you can get to with
a key that you almost never use, just so you can get out, or go to an experimental
layer that isn't fully working yet.

Keeping your layers straight and organized with some sort of master plan is going to be really
helpful. I'll explain what I did, and if you look around you'll see a few other people that 
have done something similar.

I defined keys in groups, the left and right rows, leaving the keys on the
outside edges off, that's all the big keys on the ergodox.  
_dvorak-row-1-left_, _dvorak-row-2-right_, the thumb cluster groups,
and so forth.  Then for the extra layers I created special groups like _mouse-buttons-left_,
_mouse-buttons-right_, _vi-arrows_, etc.  The edges and the thumb keys tend to be
layout independent.  You'll want these keys in the same places all the time.
_Tab, shift, caps-lock, ctrl-c, ctrl-v, toggle-to-layer_.

Here's my definitons for the rows for the dvorak layout. These definitions make 
it much easier when it comes time to make the _keymap.c_ file. We'll need to add 
the keys on each end
of the ergodox, all the _big_ keys... And we can reuse the thumbs and bottom
rows to create new base layers. But these take care of the basic dvorak layout.
Keeping things nicely aligned is very important to make this readable. If you miss
a comma, a key, or you have an extra key, it's not going to compile.

```C
#define ___DVORAK_L1___ KC_QUOT,    KC_COMM, KC_DOT,         KC_P,         KC_Y
#define ___DVORAK_L2___ KC_SFT_T_A, KC_O,    KC_LT_MDIA_E,   KC_LT_SYMB_U, KC_I
#define ___DVORAK_L3___ KC_SCLN,    KC_Q,    KC_J,           KC_K,         KC_X

#define ___DVORAK_R1___ KC_F, KC_G,          KC_C,           KC_R,   KC_L
#define ___DVORAK_R2___ KC_D, KC_LT_SYMB_H,  KC_LT_MDIA_T,   KC_N,   KC_SFT_T_S
#define ___DVORAK_R3___ KC_B, KC_M,          KC_W,           KC_V,   KC_Z

#define ___ERGODOX_BOTTOM_LEFT___  LCTL(KC_C),  LCTL(KC_V),  KC_INS,  KC_LEFT, KC_RIGHT
#define ___ERGODOX_BOTTOM_RIGHT___ KC_UP,  KC_DOWN,  KC_BSLASH,  LCTL(KC_V),  LCTL(KC_C)

#define ___ERGODOX_THUMB_LEFT___                \
  OS_RALT, TG(MDIA),                            \
    HOME_END,                                   \
    CTL_BSPC, ALT_DEL, XMONAD_ESC

#define ___ERGODOX_THUMB_RIGHT___               \
  TG(SYMB), OS_RALT,                            \
    KC_PGUP,                                    \
    KC_PGDN, ALT_ENT, CTL_SPC

```
The finished layout in _ergodox_ez/keymaps/ericgebhart/keymap.c_ looks like this.

```c
  [DVORAK] = LAYOUT_ergodox_wrapper(
                             // left hand
                             KC_GRV,     ___NUMBER_L___,   DEF_OS_LAYER_SW,
                             KC_LOCK,    ___DVORAK_L1___,  LCTL(KC_V),
                             TAB_BKTAB,  ___DVORAK_L2___,
                             KC_LSFT,    ___DVORAK_L3___,  TG(MDIA),

                             ___ERGODOX_BOTTOM_LEFT___,
                             ___ERGODOX_THUMB_LEFT___,

                             // right hand
                             MDIA_SYMB,  ___NUMBER_R___,   KC_EQL,
                             LCTL(KC_C), ___DVORAK_R1___,  KC_SLASH,
                             /*    */    ___DVORAK_R2___,  KC_MINUS,
                             TG(SYMB),   ___DVORAK_R3___,  KC_RSFT,

                             ___ERGODOX_BOTTOM_RIGHT___,
                             ___ERGODOX_THUMB_RIGHT___
                                    ),
```



There's also a trick when creating new layers. Transparency. Here's a transparent thumb
cluster for use in the mouse and symbol layers.

```
#define ___ERGODOX_TRANS_THUMBS___              \
  ___, ___,                                     \
    ___,                                        \
   ___, ___, ___                                \

```

You might see that previously I had defined these,

```C
#define ___ KC_TRNS
#define XXX KC_NO
```

They do what you would expect. Here's the code for the mouse layer. You can
see that a lot of the keys are transparent, meaning they work the same as
they do on the layer below. Some keys are XXX which makes them dead keys,
they don't do anything, it didn't make sense to have some letter keys in
this layer. 

```C
  [MDIA] = LAYOUT_ergodox_wrapper(
                          // left hand
                          ___,  ___FUNC_L___,            ___,
                          ___MOUSE_BTNS_L___,       XXX, ___,
                          ___,  ___MOUSE_LDUR___,   XXX,
                          ___,  ___MWHEEL_LDUR___,  XXX, ___,
                          ___,  ___,  ___MOUSE_ACCL_012___,
                          ___ERGODOX_TRANS_THUMBS___,

                          // right hand
                          ___,      ___FUNC_R___,                  KC_F11,
                          KC_VOLU,  XXX,     ___MUTE_PLAY_STOP___, XXX, KC_F12,
                          /*    */  KC_PGUP, ___VI_ARROWS___,      XXX,
                          KC_VOLD,  KC_PGDN, ___MOUSE_BTNS_R___,
                          ___ERGODOX_TRANS_BOTTOM___,
                          ___ERGODOX_TRANS_THUMBS___
                          ),
```

In the mouse layer the home row is the directions _LDUR_, with the mouse on the left hand and
arrow keys on the right. The row below the home rows is the scroll wheels on left
and the 5 mouse buttons on the right. Reversed so button one is always on the index 
finger. On the left the row above home row is all 5 mouse buttons. Having two sets of 
buttons makes it easy to do highlighting and dragging.  The important edge and thumb
buttons don't change from the layer below.

So that's probably enough code to get you started. The docs are really good, and if you
Search around in other people's code you'll find plenty of cool examples of how to do 
different things that you may have never thought of. 

Like having a thumb key that is _Enter_ or _Meta/Alt_ if you hold it.
Or putting shift on your home row pinky keys, How simple and
convenient is that? I don't know what to do with my shift keys now. I
never use them except by accident out of habit every now and then.

## Is it worth it?

I know, it's complicated and a bit of work. Mine was even worse than normal because I wanted
Bepo, and Dvorak on Bepo. That was a real pain. But in all it's really not that bad, and 
now, after all of this, I wouldn't want a keyboard I couldn't program. For me,
it's totally worth it. Now that it's mostly done, It's actually pretty easy even
though I made it more difficult for myself with all these base layers and Bepo.

## Some Features of my configuration

  These are my base layers. 
   * [dvorak](https://en.wikipedia.org/wiki/Dvorak_Simplified_Keyboard)
   * [bepo](https://bepo.fr/wiki/Accueil) 
   * [qwerty](https://en.wikipedia.org/wiki/QWERTY)
   * [norman](https://normanlayout.info)
   * [colemak](https://en.wikipedia.org/wiki/Colemak)
   * [workman](https://workmanlayout.org)
   * Dvorak on Bepo
   
So that was a bit over the top. All I really care about is dvorak and bepo. The rest I could
do without. But after doing bepo it was easy to add the rest so I went ahead and did them.
Feel free to steal the ones you like.

  Additional layers are: 
   * Symbols and number pad with F-keys.
   * Mouse and arrows with media keys and F-keys.
   * A layers layer, for experiments.
   * Symbols and Numbers on Bepo.

  Thumb keys, the bottom row, and edge keys are mostly the same in all
  layers so the _arrows, tab, ctrl, alt, delete, shift_, etc all
  stay put while the interior keys change.

  The mouse layer means I don't need a mouse, and it works great! Honestly I hated it
  for about 3 days. But it got easier and easier and now I don't even think about it.
  I think I'd be really frustrated to lose my mouse keys.
  Although I use the mouse very little by the grace of Xmonad.

  The symbols and numbers mean I amost never reach for the top row of my keyboard.
  Having an embedded number pad is great and all the symbols right around the home row
  is nice too.
  
  There is a single key which _tap-dances_ through all the base layers. Although I always leave it
  in dvorak or bepo.

  Finally there are some interesting things which happen when keys are held down. These are
  actually some of my most favorite features and they are one of the easiest things to do.

  If held down:
   * The 4 big thumb keys are _control_ and _alt_ so they work with either hand. 
   * The home row pinky keys are _shift_.
   * The index finger home-row keys turn on _the symbols and numbers layer_.
   * The second finger home-row keys turn on _the mouse and arrows layer_.
   * _Escape_ is _windows/command_ when held and is located on the left thumb.

   * _F-keys_ are available on the top row of both the symbols and mouse layers.

  I have a few double or multi-tap keys, but _home/end_ is the only one I use very much.
   * The home key is _home_, but _end_ if double tapped.
   * tab is supposed to be shift tab if double tapped, but my pinky doesn't work that well.
   * Some people like caps lock on a double tap of shift. I don't.

### Future improvements.

There must be something to do. 
 * I have several keys I never use. 4 of them are on my thumbs.
 * Some layer toggle keys are unused
 * Maybe I should reuse the shift keys for something.
 * Perhaps another symbols layer with a different layout.
 * Maybe symbols on the top row instead of numbers. Like Bepo.
 * A oneshot layer for commonly typed things. _!!fmt_, hmm. what else.

### Conclusion

For me it's worth it to have all these nice customizations which make
it easier to type and use my keyboard with less strain. I prefer a dvorak
layout and that tips the scales quite a bit toward doing the programming myself
as there just aren't that many choices for dvorak keyboards.

I still tweak my layout every now and then as I realize I don't use a key or something
is too difficult. But it's settling down. Making changes is a piece of cake now
that I have a stable configuration which does mostly what I want.
Even though my ergodox is not as comfortable or as ergonomic as my kinesis 
I still prefer it because of all of the features I've built into it.

Soon I'll be able to apply all this to my xd75re, viterbi and dactyl keyboards as well.





