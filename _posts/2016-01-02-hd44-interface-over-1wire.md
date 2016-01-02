---
layout: post
title: "HD44 interface over 1Wire"
description: ""
category: 
tags: ["bash", "ash", "automation", "lcd"]
---
{% include JB/setup %}

One avenue that I have taken, but then abandoned, is the [owfs](http://owfs.org) way.

In the [description of the DS2408 1wire chip](https://datasheets.maximintegrated.com/en/ds/DS2408.pdf)
Figure 18, you have the schematic of using this chip to drive an LCD module.

It is very simple.

Also owfs has a [special implementation](http://owfs.org/index.php?page=ds2408) for ds2408,
that comes with support for the HD44, provided that you respect the wiring.

Owfs is a nice package for prototyping and if you use a standard bus master
it is very easy to setup and work.

However I had created my custom 1wire bus driver so I have abandoned this avenue.
I post this so I will not forget this try.

Provided that you are in the directory of the device.
If you the following script, it will erase and show the message, it will not
use the high level interface for the LCD module provided, but use the bare
ds2408 interface.


    #!/bin/sh
    echo 1 > strobe
    echo 0,0,1,1,1,0,0,0 > PIO.ALL
    echo 0,0,1,1,1,0,0,0 > PIO.ALL
    echo 0,0,1,1,1,0,0,0 > PIO.ALL
    echo 0,0,1,1,1,0,0,0 > PIO.ALL
    echo 1,0,1,1,1,0,0,0 > PIO.ALL

    echo 1,0,1,1,1,0,0,0 > PIO.ALL
    echo 1,1,1,0,1,0,0,0 > PIO.ALL

    echo 1,1,1,1,1,0,0,0 > PIO.ALL
    echo 0,1,1,1,1,0,0,0 > PIO.ALL

    echo 1,1,1,1,1,0,0,0 > PIO.ALL
    echo 1,0,0,1,1,0,0,0 > PIO.ALL

    echo 1,1,1,1,1,0,0,0 > PIO.ALL
    echo 0,0,0,0,1,0,0,0 > PIO.ALL

    NIBL=`hexdump -v -e '/1 "%02X"' | sed -e 's#.#\0\n#g' | sed -e '
    s#0#1,1,1,1#
    t
    s#1#0,1,1,1#
    s#2#1,0,1,1#
    s#3#0,0,1,1#
    s#4#1,1,0,1#
    s#5#0,1,0,1#
    s#6#1,0,0,1#
    s#7#0,0,0,1#
    s#8#1,1,1,0#
    s#9#0,1,1,0#
    s#A#1,0,1,0#
    s#B#0,0,1,0#
    s#C#1,1,0,0#
    s#D#0,1,0,0#
    s#E#1,0,0,0#
    s#F#0,0,0,0#'`

    for nene in $NIBL ; do echo $nene,0,0,0,0 > PIO.ALL ; done


