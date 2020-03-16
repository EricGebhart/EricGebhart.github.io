---
layout: post
title: "Arch Linux; creating an automated configuration and personal setup."
description: "A repository configuration to enable automated installation and setup of a personal Arch Linux box."
date: 2020-03-15
category: code
tags: [Arch Linux, pacman, AUR, emacs, xorg, Xmonad, Make, shell, Wacom, Mobile studio pro]
---

It always starts with something simple.
I wanted to start over with a fresh installation of
Arch Linux on my old Wacom Mobile Studio pro. It had been working mostly fine
with a few glitches that were probably due to the age of the current install
and it's lack of being used.

It really was just fine until I installed a fresh Arch Linux and it stopped booting.  I'm not sure how
many times I installed Arch Linux on that machine in the last weeks. In the process
of doing that, I decided that I wanted a reproducable state of being for future
installs.

So, in the usual fashion for me, I did both. While I worked to solve my computer's 
problems with the recent kernel updates, I also worked toward creating a personal 
install process that would result in a computer that was mine, just the way I like it.

# Here we go.

I've installed Arch Linux following
[the installation guide](https://wiki.archlinux.org/index.php/Installation_guide#Localization)
many times. I know it by heart. What I don't remember
are some of the details, which packages I like to use, what I had to do to
get my High DPI screen working properly with a visible mouse cursor, which of
the packages that I like to use are in the official package repo and which are in the
[AUR](https://aur.archlinux.org).
Some things that I forget are a bit painful, some are just another dumb thing to check 
on and follow through, or back up, do something else, then go forward again.  But it's 
a time consuming tedious task, I'm not learning, I'm just going through the paces. Most of
it is stuff I don't really want to keep in my head.

One of the recommendations in the Arch Linux community is to make your own
[meta-packages](https://wiki.archlinux.org/index.php/Meta_package_and_package_group)
for installation with pacman and possibly even host your own
[package repository, locally](https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks#Custom_local_repository), or online.
I started down that rabbit hole.
I even thought about [hosting my repo on ipfs](https://github.com/victorb/arch-mirror).
I did find 3 nice articles about hosting your own repo on an S3 bucket by Michael Daffin.

#### Automating Arch Linux by Michael Daffin.
* [Hosting an Arch Linux repo in an S3 Bucket](https://disconnected.systems/blog/archlinux-repo-in-aws-bucket)
* [Managing Arch Linux with meta packages](https://disconnected.systems/blog/archlinux-meta-packages/)
* [Creating a custom Arch Linux installer](https://disconnected.systems/blog/archlinux-installer/)

That was more than I wanted to do, so then I thought why not make a
[local repo](https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks#Custom_local_repository)
and use that. Even that turned out to be too heavy weight, time consuming and
complex, and it didn't really solve my problems.

## An install script

I spent a fair amount of time going down various rabbit holes while wondering if I
should make a repo of my own from [the meta packages I had just created](http://github.com/ericgebhart/arch-pkgs).
So I experimented with package groups and building packages from the AUR. I learned a lot.

Meanwhile I was doing repeated installs and experimenting with various kernel options
and versions which may or may not have booted on my Wacom mobile studio pro. Something
changed in the kernel between 01/12/2019 and 01/01/2020 which caused linux to not boot. I needed
_ACPI=off_ which took a while to figure out, and a little longer to decide I didn't care.

The install became very repetitive during this time so I copied
[Michael Daffin's script](https://disconnected.systems/blog/archlinux-installer/)
and made it my own. Another facet of this project was born. And a source of more twiddling.

## Complexities

The complexities come in various forms. First, was that the process of building
a local repo takes a fair amount of processing time. But then the complexities
creep in, and a repo doesn't actually solve any of them.

The key problems I had were these:

* Packages cannot have dependencies for [groups](https://www.archlinux.org/groups/).
* Packages cannot have dependencies for anything in the [AUR](https://aur.archlinux.org).
* It's difficult to create a package from things which are in the AUR.
* You must install dependencies manually for things in the [AUR](https://aur.archlinux.org).
* My personal configuration of my user account, my dot files, emacs and Xmonad are their own repos
  which install to user space. That just seems like the wrong thing to do with a meta package.

### What I wanted

I wanted something that would install what I needed, without me needing to maintain what
packages were actually in the package groups I wanted. I wanted something that could
install official packages or packages from AUR. I don't have many AUR packages, but my
web browser [vivaldi](http://vivaldi.com) is one of them and it needs an _ffmpeg_ package
to be installed after vivaldi is installed.

My personal repositories for dotfiles, emacs, and xmonad were intertwined and
I needed them to be installed correctly, with the bits and pieces I needed for
various configurations. A High DPI monitor being one example of the variables.

## A package installer

While I waited for the installs, I wrote a script to install my packages and configurations,
It mostly worked, and I learned about the peculiarities of the problem. When the script was
finally working it was ugly in it's form and function, but it worked.
I wanted a list of associative arrays. You can't do that in shell. The dependencies
were getting complicated.  My emacs setup needed _emacs, isync, languagetool and hunspell_ from
the official package repository and
[mu-git with mu4e](https://www.djcbsoftware.nl/code/mu/mu4e.html)
from the AUR.  Then it needed to install my personal
[emacs-setup](http://github.com/ericgebhart/emacs-setup) repository in git.

My script had a nice checklist menu via 
[dialog](http://linuxcommand.org/lc3_adv_dialog.php), 
but the rest was not very elegant.

### makepkg, Pacman, [yay](http://github.com/jguer/yay) [groups](https://www.archlinux.org/groups/) and dependencies.

After lots of experimenting and 2 or 3 different implementations, I decided that
my git repository of meta packages and a Makefile was the best and simplest solution for me.
There is absolutely no need to create an Arch Linux package repo of any sort. PKGBUILDS
are lightweight and the process of building/installing them is easier than turning them
into a repository that I have to put somewhere.

I decided that I should just install the package groups I wanted directly
and let my meta packages list their dependencies for individual official packages.
All that is needed for that is to run pacman for the groups and `makepkg -si` for each
meta package and voila, all my official packages are installed.

But then I also had packages from the AUR, my personal configuration repositories and
the _xmonad-log-applet_ which needed an _autoconfig, make and make install._

And there were all the intertwined dependencies between them all.

## Makefiles are genius.

As I worked on the Makefiles in my configuration repositories I realized I should
have Makefiles everywhere. After all, this project had turned into two shell scripts
with some submodules which were all other git repos. Each should stand on it's own.

I made a Makefile to build/install my meta packages in
[my arch-pkgs repo](http://github.com/ericgebhart/arch-pkgs),
and then I made a Makefile for the top level.
The package install script became nothing more than a menu that ran make.
The Makefiles are as simple as can be and track all the dependencies almost like magic.
It is, afterall, what _make_ was intended to do. Use the right tool and things become simple.

The install-packages script became a somewhat pretty face for the Makefile,
[dialog](http://linuxcommand.org/lc3_adv_dialog.php) is many
things but pretty isn't really one of them.
Suddenly my solution was elegant and maintainable.

# Making Meta Packages.

A Meta package is, in it's simplest form a
[PKGBUILD](https://wiki.archlinux.org/index.php/PKGBUILD)
file in a folder, The only thing
it really needs other than the meta data at the top is a list of packages it depends upon.

Here's my Xmonad meta package.

    # Maintainer: Eric Gebhart <e.a.gebhart@protonmail.com>
    pkgname=Xmonad-setup
    pkgver=1
    pkgrel=1
    pkgdesc="Everything needed by my Xmonad Setup"
    arch=(any)
    url="https://github.com/ericgebhart/arch-pkgs"
    license=(MIT)
    groups=(eagebhart)

    ### XMonad with accessories and haskell.
    depends=(
        xmonad xmonad-contrib ghc dbus haskell-dbus
        dmenu dzen2 network-manager-applet xfce4-panel)

    rootdir=$PWD

Building a meta-package with `makepkg -si` results in the installation of all those dependencies.
So this [PKGBUILD] (https://wiki.archlinux.org/index.php/PKGBUILD)
will install _xmonad xmonad-contrib ghc dbus haskell-dbus dmenu dzen2
network-manager-applet and xfce4-panel_  as needed.

But remember, I have some package groups I'd like to install from the official repository and
putting those in a meta package's dependency list doesn't work unless I want to list every
single package in the group.  I'd rather let the maintainers of the groups do that.

The Makefile takes care of everything.

```make
    packages := $(shell ls -d */ | sed 's,/,,')

    # groups cannot be installed via dependencies in PKGBUILD
    groups := xorg xorg-apps xorg-fonts alsa xfce4 xfce-goodies

    all: $(packages) $(groups)

    print-%  : ; @echo $* = $($*)

    clean:
            rm -f */*.xv
            rm -f */*.xz
            rm -rf */src
            rm -rf */pkg

    .PHONY: $(packages) $(groups)
    $(packages):
            cd $@; makepkg -si --noconfirm;

    $(groups):
            sudo pacman -S --noconfirm $@


    # not necessary to list them, but it's clearer.
    necessities:
    dotfiles:
    yay:
    emacs: necessities
    X11: xorg xorg-apps xorg-fonts X11-apps
    X11-apps: audio
    Xfce: xfce4 xfce-goodies
    audio: alsa
    Xmonad:
    devel:
    natural-language:
    mobile-studio-pro:
    tablet:

    base: X11 X11-apps audio necessities dotfiles Xmonad yay
```

This is one of the simplest Makefiles you will ever see. We get a list of
all the meta packages with an `ls`.  We create a list of package groups
manually, there aren't that many.

We have an _all:_ rule to build/install everything.  We have two rules, one for the
meta packages which does a `makepkg -si` and another rule which runs `pacman` to install
any groups that are triggered.

At the end, I've listed some of the build targets, but that isn't really necessary unless
they have dependencies like _X11_ and _Xfce_.  Both of which are meta packages, making
them results in all of their dependecies being installed before they are installed themselves.

That's it. You can build any target with `make <target name>` like `make X11` or
`make xorg`.  If there are dependencies those get built too.

# Making everything together.

At this point all the submodule repos have working Makefiles. Now there needs to
be a Makefile for the entire setup with dependencies on configuration repos, and
packages from the AUR. This Makefile is just like the last one, but with a bit more
to do, although it doesn't need to know much other than how to call make.

```make
packages := $(shell cd arch-pkgs; ls -d */ | sed 's,/,,')
aur-packages := yacreader vivaldi vivaldi-codecs-ffmpeg-extra-bin mu-git
repos := xmonad-setup emacs-setup dotfiles bc-extensions
everything := $(packages) $(aur-packages) $(repos) hidpi xmonad-xsession xmonad-log-applet


all: $(everything)

# make print-packages, etc. - to see what's in a variable.
print-%  : ; @echo $* = $($*)

.PHONY: $(everything)

clean:
	cd arch-pkgs; make clean; \
        cd xmonad-log-applet; make clean

$(packages):
	cd arch-pkgs; make $@

$(aur-packages):
	yay -S --noconfirm $@

$(repos):
	cd $@; make install

hidpi:
	cd dotfiles; make hidpi

xmonad-xsession:
	cd xmonad-setup; make xsession

xmonad-log-applet: Xmonad
	cd xmonad-log-applet; ./autogen.sh && make && sudo make install

necessities: yay
dotfiles: dotfiles bc-extensions
emacs: necessities emacs-setup natural-language mu-git
yay:
X11: X11-apps
X11-apps: yay yacreader vivaldi vivaldi-codecs-ffmpeg-extra-bin
Xfce:
xmonad-setup: Xmonad xmonad-log-applet
Xmonad:
devel:
natural-language:
mobile-studio-pro: hidpi
tablet:

base: yay X11 X11-apps necessities emacs dotfiles Xmonad
account: dotfiles emacs Xmonad

```

As you can see it is very similar to the meta-packages makefile.
We get a list of the target packages by doing
an `ls` on the arch-pkgs repo.  The AUR packages and the configuration repos are
listed out.  We now have 3 make rules which handle the meta-packages to
be installed with [pacman](https://wiki.archlinux.org/index.php/Pacman),
the AUR packages to be installed with [yay](http://github.com/jguer/yay)
and the repos to be installed with [make](https://www.gnu.org/software/make/).
There are additional specific rules
for _hidpi_, _xsession_, and _xmonad-log-applet_,
each of which have something specific about their invocation.

The beauty of this is that we can track our dependencies and build/install everything
as needed. Notice that I put _yay_ as a dependency on X11-apps, base and emacs since all 
of those install packages from the AUR. It wasn't really necessary, since I always
install _necessities_ but you never know when someone might skip a step.

So looking at the _emacs_ rule. We can see it has these dependencies

* necessities - [yay](http://github.com/jguer/yay), vi, nano, fonts, silver-searcher...
* emacs-setup - my git repo of elisp code.
* natural-language - my meta package with languagetool, hunspell, etc.
* mu-git - an AUR package for email.
* emacs - my meta packages for emacs which includes isync.

So when we run `make emacs` 
* Necessties depends on _yay_ so my yay package gets installed first.
  It calls `make yay` on the arch-pkgs Makefile which calls
  `makepkg -si --noconfirm` in the yay package folder of arch-pkgs. 
* It installs my necessities meta-package with `make necessities`,
  which becomes `makepkg -si --noconfirm` in the arch-pkgs Makefile.
* It runs `make install` in my emacs-setup repo to install my elisp code. 
* Next is `make natural-language' in my arch-pkgs.
* Then it does a `yay -S --noconfirm mu-git` to install the _mu-git_ package from the AUR. 
* Finally it installs the emacs meta package with `make install emacs` which results in
  `makepkg -si --noconfirm` in my arch-pkgs repo.

Perhaps I should rearrange that. No problem. Things like that are simple.
Maybe this instead.

`emacs-setup: necessities emacs natural-language mu-git`

I think that's much better. Now I need to change the menu from _emacs_ to _emacs-setup_.

I could sprinkle some more dependencies here and there to insure safety in the
installation order. But mostly I run this thing once.  After that I only reinstall
my configurations as they change. 

The one problem here is that because we have
no real targets, we have to use the *.PHONY* rule to force make to do things 
when we ask.  That means it always does them because it can't tell if it's done
them before or not.  So a little prudence is probably necessary when it comes to
making rules.  We don't want to install _yay_ 12 times in a row. Although pacman
might save us some work.

_Pacman_ and _yay_ take care of all my upgrades (_yay -Syu_), so this is really a 
one shot thing. I probably don't need all the dependencies I have here. I always 
install _necessities_ first so there really isn't a big danger of it going missing 
when it's needed. _yay_ is a part of the necessities rule so there's not much danger 
there either.  

### install-packages

At this point, this script is just a checklist menu that runs make. It's totally
unecessary, but it's nice, so I kept it.

# In conclusion

This has been a fun journey and I can already tell that I will be using this
[Arch-Setup](http://github.com/ericgebhart/Arch-Setup) project for a long time.

I have a Makefile which handles all my software installation dependencies and it's
all very simple. Adding new things to my system is a breeze wether they are packages,
AUR packages, some dot file or some random git repo.

I also have an install script which adds a few things to the normal
installation procedure. It installs some extras I always seem to forget, including
the things I need to install my packages. It creates
my account in wheel with zsh, it installs and enables
[Network manager](https://wiki.archlinux.org/index.php/NetworkManager).
so I can easily
get an internet connection with `nmcli` or `nmtui` once I login to my new machine.
I don't have to remember to start and enable the Network manager daemon before I can connect.
It's not a big deal to type `sudo systemctl --now enable NetworkManager`.
But I always seem to forget the first and only time I need to do it.

Now I don't have to remember so much. I just have to start by following
[the installation guide](https://wiki.archlinux.org/index.php/Installation_guide)
. Once I have internet on my Arch Linux Live USB boot I can do

```shell
curl https://raw.githubusercontent.com/ericgebhart/Arch-Setup/master/install-arch > install-arch
chmod a+x ./install-arch
./install-arch -h
```

Then go from there. I tend to like to do my own partitioning and formatting, but it will do it
for you if you like. `./install-arch -h` will tell you what you need to know if you know what
you are doing. 

### If you are not sure. 

Ok then, go to [the installation guide](https://wiki.archlinux.org/index.php/Installation_guide)
and follow all the steps. You can use _install-packages_ later once you've got your 
system running.

## But wait there's more

The cool trick after all this basic stuff is that the install also
does a `git clone`  of my Arch-Setup repo, and runs _nmtui_ and
_install-packages_ automatically upon the first login after reboot.
All I have to do is check off the boxes I want and wait. logout, login, et voila, my
system, my way, ready to go.

## Make it yours.

Please don't hesitate to fork my repos and create your own, or add to what I have
and do a pull request. There's no reason this couldn't be more universal than it is.
