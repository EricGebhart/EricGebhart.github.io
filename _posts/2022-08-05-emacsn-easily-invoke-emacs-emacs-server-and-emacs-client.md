---
title: Emacsn, easily invoke Emacs, Emacs server, and Emacs client.
layout: post
TAGS: [emacs xmonad shell]
CATEGORY: Emacs
DESCRIPTION: A shell script to simplify invoking emacs in various ways.
TITLE: Emacsn, easily invoke Emacs, Emacs daemon, and Emacs client.
PUBLISH-TO-PROJECT: eg-com
DATE: [2022-08-05 Fri 17:21]
ON: [2022-08-05 Fri 17:21]
---

- [Emacsn, easily invoke Emacs, Emacs server, and Emacs client.](#orgc03c804)
  - [Introduction](#orgcd43b00)
    - [What did I need ?](#orgb4deafb)
  - [This script is old and well used.](#org82b6205)
  - [Elisp functions that are invoked by some options.](#orgb2d43ab)
  - [The emacsn command options help.](#orgf47c893)
  - [Examples from the help:   Follow any of them with one or more files to load.](#org2a24c57)
    - [Stand alone Emacs](#orgdc4693e)
    - [Using an an unnamed server daemon:](#org069fe46)
    - [Using a named emacs server daemon:](#orgb04258f)
  - [How I use it.](#org49984e7)
    - [Browser command script](#orge27ca60)
    - [Export browser command in .zprofile](#org133f05a)
    - [Xmonad gives me the sessions I need on the desktops I want them on.](#orgfd64780)
  - [The code](#orgdaed2df)
  - [Summation](#orgaab1168)


<a id="orgc03c804"></a>

# Emacsn, easily invoke Emacs, Emacs server, and Emacs client.     :emacs:blog:


<a id="orgcd43b00"></a>

## Introduction

There was a recent conversation on discord about work flows and using named emacs daemons. I realized that my *emacsn* script might actually be helpful to some folks. This is a description of that script.

I´ve been using emacs for a long time. At some point many years ago I wrote this shell script to simplify starting emacs standalone, emacs as a server and emacs client, while running the elisp function desired upon startup. It has grown over the years, with a recent refactor of named server usage and the addition of the EAF Browser. Although it would work fine for any emacs browser should someone want that instead.

The permutations of options between emacs, and emacs client are numerous and conflicting. I just wanted something simple and consistent, that gave me all the power I needed. While also saving me the pain later on of wading through all the options and manual pages for them all just to figure out what I did in the first place. This is a very nice but simple script with verbose help. It normalizes options a bit just to create consistency.


<a id="orgb4deafb"></a>

### What did I need ?

-   run emacs, emacs client, or emacs daemon with the same arguments.
-   run emacs client and use an existing client or start a new one.
-   connect to named or unnamed emacs daemons.
-   start emacs in terminal, or as a window.
-   start mu4e as my mail client
-   start eaf browser as my browser and keep it separate from other things.
-   load a url in an existing emacs session. - my eaf-browser client.
-   Set the window title.
-   Run in terminal mode.
-   Load one or more files on startup.
-   Execute an elisp function on startup.


<a id="org82b6205"></a>

## This script is old and well used.

I dont know how old it is. It could be 20+ years now. The things I've wanted it to do have grown over the years. It started with setting the title and then running an elisp function which splits panes, starts a shell, or runs some special elisp for the usage situation and load the files requested. I also wanted to run in terminal or window mode.

I've had various elisp functions over the years, but now I only use *mu4e* and my *main-window* function which just splits some windows. I recently added *eaf-open-url* which is a custom function wrapping some EAF functions, so that the EAF browser can be fully utilized from outside of emacs.

There are unnamed server daemons and named server daemons. We want to connect to whichever. I want a specific emacs instance running mu4e because I don't want it messing with my other sessions. But maybe I just want to connect to it for an instant. Mu4e can become locked, there is only one xapian connection allowed, so its nice to have it in a server if opening more than one client to access it.


<a id="orgb2d43ab"></a>

## Elisp functions that are invoked by some options.     :elisp:

All of three of the elisp functions I run have a special option in this script which is a short circuit to the *-f* function option.

-   *-e* runs mu4e
-   *-m* runs *main-window*
-   *-b URL* runs *eaf-/open-url*
-   *-f function* runs the function given.

Here are the only two custom elisp functions I use with the *-m* and *-b* options which are just special cases of *-f* which will run any elisp function. The *-e* option just runs mu4e directly which should exist if you have it installed.

```elisp
(defun main-window ()
  (interactive)
  (balance-windows)
  (split-window-horizontally)
  (split-window-horizontally)
  (split-window-horizontally)
  (split-window-horizontally)
  ;;(split-window-below)
  ;;(cb-get-shell)
  )

(require 'eaf-browser)
(defun eaf-open-url (url)
  "Non interactive way to open a browser url in eaf-browser.
   Wraps urls with https:// as needed."
  (eaf-open (eaf-wrap-url url) "browser"))
```


<a id="orgf47c893"></a>

## The emacsn command options help.

Here is the basic help text which explains the command line options.

emacsn - helper for running emacs

*emacsn [options] [files]*

Run emacs as emacsclient or emacs non-client with a title, in a terminal or not, with a lisp evaluation function on startup. Load any files listed.

Client/server -c Use emacsclient -s server/socket name Connect to, or Start, a named daemon -d Start an unnamed server daemon

Run a lisp function on startup. -e, -m, -b, -f -e Title it Email and eval mu4e. -m name Title it name and eval (main-window) -b url Open eaf-browser with url -f lisp-func Evaluate lisp-func on startup.

-h This help text. -T title Give emacs that title. -t *emacs -nw* - run in a terminal. -w New window/frame &#x2013;create-frame. opposite of -t


<a id="org2a24c57"></a>

## Examples from the help:   Follow any of them with one or more files to load.

It´s probably overkill, but it´s nice to have examples.


<a id="orgdc4693e"></a>

### Stand alone Emacs

-   Start a default emacs session *emacsn*
-   Start terminal session. *emacsn -t*
-   Start an mu4e email session (run mu4e) as a standalone emacs *emacsn -e*
-   Start a split frame session (run main-window) as a standalone emacs *emacsn -m MySessionTitle*
-   Start a browser session (run eaf-open-url) as a standalone emacs *emacsn -b duckduckgo.com*


<a id="org069fe46"></a>

### Using an an unnamed server daemon:

-   Start an unnamed server daemon: *emacsn -d*
-   Start a client in new window/frame: *emacsn -cw*
-   Start a client in terminal: *emacsn -ct*


<a id="orgb04258f"></a>

### Using a named emacs server daemon:

-   Start a server daemon with a name of "common": *emacsn -s common*
-   Start a server daemon running mu4e with a name of "mail" :' *emacsn -es mail*

1.  Emacs client with a named server daemon.

    -   Start a client in new window/frame using a named server: *emacsn -cws common*
    -   Start a client in an existing window/frame using a named server: *emacsn -cs common*
    -   Start a client running mu4e in a new window/frame using a named server of email:' *emacsn -ces email*
    -   Open an eaf browser on url in an new client window/frame: *emacsn -cws common -b duckduckgo.com*
    -   Open an eaf browser on url in an existing window/frame: *emacsn -cs common -b duckduckgo.com*
    -   Start a split frame session (main-window) in new frame: *emacsn -cws common -m MySessionTitle*
    -   Start an mu4e email session. *emacsn -cews mail*
    
    *- Start terminal session. /emacsn -cts common*


<a id="org49984e7"></a>

## How I use it.

At the moment, I use two named servers, one for email and one for the browser. Everything else I do is a standalone emacs session, or a client to one of the servers. Basically I have a browser client and an email client, and a bunch of standalone emacs sessions for each project. I can, of course, connect to the servers as I wish, It is instantaneous.

I should mention that I use Xmonad as my window manager, so having a specific 'topic' desktop for browsing and mailing is super easy. When I visit an empty desktop Xmonad automatically starts the applications that belong there in the working directory they belong in. I run telegram next to mu4e, and discord next to eaf-browser. I also have a search in Xmonad, which I use as my main interface outside of emacs to EAF-browser.

I don't like to mix my emacs sessions too much. Most of the time I run a standalone emacs for a specific project, the servers allow me to keep tasks that can have a lot of buffers like the browser and email in one place where they don't clutter up my project work environment. I am beginning to think of running servers as topics like QMK, or clojure, or other commonalities, but I haven't tried that yet.

I really only run three elisp functions on startup, main-window, mu4e or eaf-open-url. The *main-window* function just divides the frame/window into separate windows/panes. This is how I start most programming sessions. Specifying mu4e runs mu4e as we would expect.

Currently I run two named servers, *common*, and *email*. I use *common* for EAF browser, and *email* for mu4e. I start them up with everything else in my StartX.

Note: *Sometimes a server will fail to start when the X server hasn't started up completely.* *I put my servers at the end of my StartX script for this reason. It mostly* *doesn't fail. If it does, I start it manually. It hasn't been annoying frequently enough to change.*

I can use the EAF Browser in any emacs session, but at the moment I think it makes sense to have a server for it. All the loaded web pages are in one server so they don't clutter up my buffer space in other sessions. With the server I can easily start another client and have access to all the pages loaded by the browser no matter which client loaded them. It's all the same session since its a named emacs daemon . But maybe I´m wrong. I do occasionally open the browser in my current session too. So it is all very flexible. I might have to try it without the server just to see. I have noticed that I tend to kill browser buffers and not leave them around. Unlike Vivaldi where I end up with 50 tabs open.

For everything else, I run a basic emacs which invokes *main-window*. This splits my emacs into a few windows/panes and maybe starts eshell. Of course any files listed will be loaded. Xmonad topics sets the root directory on startup which makes everything easier. I simply visit the desktop and whatever should be there starts up with it´s home in the right place.

I put my browser command in a script named <span class="underline"><span class="underline">emacsbrowser</span></span> The script allows me to easily assign the emacs command to the default browser in my shell. The behavior is that it opens the urls in the current emacs client, if there is one, or create a new one in the current desktop if there is not one somewhere. All connected to the emacs daemon server named *common*. This allows my Xmonad Search to give it´s url to the eaf browser so I never have to visit a search engine.

Here is the extra code bits that connect all the dots.


<a id="orge27ca60"></a>

### Browser command script

This uses existing client if possible, connected to the *common* server. Adding the *-w* option would create a new window everytime&#x2026; Not sure that would be good. And of course within emacs *open-url-at-point* is available, which just opens a browser buffer right there.

I put this in a file named *emacsbrowser* in my personal bin directory. \#+BEGIN<sub>SRC</sub> shell emacsn -cs common -b $1 \#+END<sub>SRC</sub> shell


<a id="org133f05a"></a>

### Export browser command in .zprofile

\#+BEGIN<sub>SRC</sub> shell export BROWSER=emacsbrowser \#+END<sub>SRC</sub> shell


<a id="orgfd64780"></a>

### Xmonad gives me the sessions I need on the desktops I want them on.

Examples of Xmonad topic entries for the MyQMK, Web, and Comm desktops.

\#+BEGIN<sub>SRC</sub> haskell , TI "MyQMK" "play/myqmk/users/ericgebhart" (spawnInTopicDir "emacsn -m MyQMK README.md") , TI "Web" "" (spawnInTopicDir "emacsn -cws common -b duckduckgo.com" >> spawnInTopicDir "discord") , TI "Comm" "" (spawnInTopicDir "telegram-desktop" >> spawnInTopicDir "emacsn -cews mail") \#+END<sub>SRC</sub> haskell


<a id="orgdaed2df"></a>

## The code

Here´s all the code minus most of the help text. [Get the full version here on my github.] (<https://github.com/EricGebhart/dotfiles/blob/master/bin/emacsn>) I find this code to be an excellent template for other simple shell scripts.

```shell
#!/usr/bin/env zsh

runemacs=`which emacs`

eval_prefix=" --eval '("
eval_suffix=")'"
runfunc=""
background=" &"
title=""
terminal=""
daemon=""
emacsclient="no"
email_elisp='mu4e'
edit_elisp='main-window'
browser_eval_prefix=' eaf-open-url "'
browser_eval_suffix='"'

help(){
    print 'emacsn - helper for running emacs'
    print ' '
    print 'emacsn [options] [files]'
    print ' '
    exit
}

# $opt will hold the current option
local opt
while getopts cetdhwm:f:T:b:s: opt; do
    # loop continues till options finished
    # see which pattern $opt matches...
    case $opt in
        (d)
            #eval "emacs --daemon"
            if [ -z "$daemon" ]; then
                daemon="--daemon"
            fi
            ;;
        (s)
            daemon="--daemon="$OPTARG
            socket="--socket-name="$OPTARG
            ;;
        (c)
            runemacs=`which emacsclient`
            emacsclient="yes"
            ;;
        (e)
            runfunc=$eval_prefix$email_elisp$eval_suffix
            title="-T Email"
            ;;
        (m)
            runfunc=$eval_prefix$edit_elisp$eval_suffix
            title=' -T '$OPTARG
            ;;
        (b)
           runfunc=$eval_prefix$browser_eval_prefix$OPTARG$browser_eval_suffix$eval_suffix
            ;;
        (f)
            runfunc=$eval_prefix$OPTARG$eval_suffix
            ;;
        (T)
            title=' -T '$OPTARG
            ;;
        (t)
            terminal=' -nw ' $terminal
            background=''
            ;;
        (w)
            terminal=' -c ' $terminal
            ;;
        (h)
            help
            ;;
        # matches a question mark
        # (and nothing else, see text)
        (\?)
            print "Bad option:" $*
            print " "
            help
            return 1
            ;;
    esac
done
(( OPTIND > 1 )) && shift $(( OPTIND - 1 ))
#print Remaining arguments are: $*

# switch the -s arg if we are running emacs client.
if [[ ${emacsclient} == "yes" ]]; then
    daemon=$socket
fi

mycommand=$runemacs' '${daemon}' '${terminal}' '${title}' '${*}' '$runfunc' '${background}

print "Emacs With Command:" $mycommand

eval $mycommand
```


<a id="orgaab1168"></a>

## Summation

I hope that someone will find this useful. The command lines for emacs and emacsclient are a bit tricky with all of these options working together. I haven't had to think about it in a long time. This script gives me the freedom to just think about how I'd like to use emacs, emacs server, and emacs client in the bigger sense without worrying or thinking about the implementation details.