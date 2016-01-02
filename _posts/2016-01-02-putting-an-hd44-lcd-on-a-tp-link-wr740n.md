---
layout: post
title: "Putting an HD44 LCD on a TP Link WR740N"
description: ""
category: 
tags: ["bash", "ash", "automation", "lcd"]
---
{% include JB/setup %}

I was planning the automation solution for my house.
To keep the long story short, this is the solution that I came up with:

* I will use the low cost 1Wire devices for sensors and command in my house.
* In order to command the 1Wire bus I will use a low cost router that can be
redone with openwrt.
* I will connect the router to a HD44 LCD module to display the house status.

I am horrified of the prices that now the home automation are.
So I needed a low cost central control unit for my home automation.

TP Link WR740N fit the bill.

* It is low cost : 15$
* It has a UART that can be used to manage a 1Wire bus.
* It has 8 GPIOs.
* It has a "usable" usb 2.0 that you can get to using this [mod](https://forum.openwrt.org/viewtopic.php?id=33991), I didn't go that far.

You can get more information on this router on the openwrt [page](https://wiki.openwrt.org/toh/tp-link/tl-wr740n).

Also it makes no sense to use more than 2 GPIOs to communicate with the LCD
module, you don't have to many around anyway.
So I have hijacked the LED for the lan2 as the serial data and
the LED for the lan3 as the serial clock.

At the HD44 module side, I have used [this](http://www.circuitvalley.com/2011/12/two-wire-serial-lcd-16x2-graphics.html)
solution to be able with 2 wires to talk to the module, again low cost.

As the software part I was using only ash scripting.
Because we already have busybox in openwrt and the speed is not an issue.

So to setup the GPIOs I issue:

```
#!/bin/sh
echo none > /sys/class/leds/tp-link:green:lan3/trigger
echo none > /sys/class/leds/tp-link:green:lan2/trigger

ln -s /sys/class/leds/tp-link:green:lan2/brightness /tmp/date
ln -s /sys/class/leds/tp-link:green:lan3/brightness /tmp/ceas
```

Now I need a little scrip that will write the stdin to the lcd, no formating.
You have to know in advance what kind of LCD this is (1x16,1x8,2x16,4x20, etc.)

So here it is, it will also reset the LCD on every call.

```
#!/bin/sh
# lan2 alb date
# lan3 verde ceas

echo 1 > /tmp/date
echo 1 > /tmp/ceas

scrieNible(){
# stergere registru.
    echo 1 > /tmp/date
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas

# scriem enable
    echo 0 > /tmp/date
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas

# scriem r/s
    [ $1 -eq 1 ] && echo $1 > /tmp/date
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas

# scriem 3 2 1 0
    [ $2 -eq $1 ] || echo $2 > /tmp/date
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    [ $3 -eq $2 ] || echo $3 > /tmp/date
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    [ $4 -eq $3 ] || echo $4 > /tmp/date
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas
    [ $5 -eq $4 ] || echo $5 > /tmp/date
    echo 0 > /tmp/ceas
    echo 1 > /tmp/ceas

# finalizare, ne asiguram ca E are fron negativ
    [ $5 -eq 1 ] && echo 0 > /tmp/date
    echo 1 > /tmp/date
}

scrieNible 1 1 1 0 0
scrieNible 1 1 1 0 0
scrieNible 1 1 1 0 0
scrieNible 1 1 1 0 0
scrieNible 1 1 1 0 1

scrieNible 1 1 1 0 1
scrieNible 1 0 1 1 1

scrieNible 1 1 1 1 1
scrieNible 1 1 1 1 0

scrieNible 1 1 1 1 1
scrieNible 1 1 0 0 1

scrieNible 1 1 1 1 1
scrieNible 1 0 0 1 1

hexdump -v -e '/1 "%02X"' | sed -e 's#.#\0\n#g' | sed -e '
s#0#1 1 1 1#
t
s#1#0 1 1 1#
t
s#2#1 0 1 1#
s#3#0 0 1 1#
s#4#1 1 0 1#
s#5#0 1 0 1#
s#6#1 0 0 1#
s#7#0 0 0 1#
s#8#1 1 1 0#
s#9#0 1 1 0#
s#A#1 0 1 0#
s#B#0 0 1 0#
s#C#1 1 0 0#
s#D#0 1 0 0#
s#E#1 0 0 0#
s#F#0 0 0 0#' | while read a b c d ; do scrieNible 0 $d $c $b $a ; done
```

