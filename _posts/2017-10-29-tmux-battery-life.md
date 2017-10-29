---
layout: post
title:  "Battery life as hearts in tmux"
date:   2017-10-29 19:42:00 +0100
author: Paul Jähne
---

Ever wanted to have your battery charge visible in your favourite terminal multiplexer? What about this:

![Battery life shown as hearts in tmux](https://github.com/SethosII/tmux-batterylife/raw/master/example.png)

Looks awesome? Then look no further than this [little script](https://github.com/SethosII/tmux-batterylife) I created! It shows you the current charge as red hearts, empty as white hearts, and dead as black hearts. Dead means the difference between the design charge and the last full charge. The script checks common locations of battery information and extracts the data from there. It should also work on OS X, although, I haven't tested it in a while because of a lack of Mac products ;)

The [repository](https://github.com/SethosII/tmux-batterylife) also contains an example [tmux configuration file](https://github.com/SethosII/tmux-batterylife/blob/master/example_tmux.conf) to show how to integrate it.

The script was originally inspired by another [blog entry](https://aaronlasseigne.com/2012/10/15/battery-life-in-the-land-of-tmux/).
