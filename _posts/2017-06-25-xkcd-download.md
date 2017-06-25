---
layout: post
title:  "Download all xkcd comics"
date:   2017-06-25 20:05:00 +0100
author: Paul Jähne
---

I quite like Cinnamon's [xkcd desklet created by rjanja](https://cinnamon-spices.linuxmint.com/desklets/view/5) which always greets me with some funny stuff. It also stores all the viewed comics with a JSON file containing the metadata on your computer. Recently, I wanted to download all the comics to shorten a train ride. So I searched and found a [gist by wjh](https://gist.github.com/wjh/d99ec012ac09281c53fe) which does exactly this. Unfortunately, it wasn't updated in quite some time and therefore doesn't include some changes and subsequently failed to download some comics.

So I looked into the script and updated it to make it work again for all comics except the ones without images (some aren't images but interactive). I also added a check functionality so that it shows which comics failed to download. You can find the [updated script](https://gist.github.com/SethosII/20db363aef55d3252dd1c47c960e16a2) on Gist.
