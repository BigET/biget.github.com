---
layout: post
title: "Practical use of HITACHI HD44 compatile LCD"
description: ""
category: 
tags: ["lcd", "automation"]
---
{% include JB/setup %}

So HD44 is, [as wikipedia put it](https://en.wikipedia.org/wiki/Hitachi_HD44780_LCD_controller), "the de-facto standard" for alphanumeric dot matrix liquid crystal.

You can see at the wikipedia article the command reference.

Also there is a very nice javascript [simulator](http://www.dinceraydin.com/djlcdsim/djlcdsim.html) for it.

The main problem with a component is the state management of it.
What happens if because of a power spike or whatever reason the internal module
state gets messed up.

* You can, if you really want, query the state of the LCD.
And from time to time you can see that the state of the module is the one you expect.
* Another way is to do like the ostrige and berry your head in the sand.
Ans just hope this will never happens. The main problem is that problems do
happens and state corruption is very visible on the display.
* The middle way is to just make the display process stateless.
The cost here is that you have to write more to the LCD and also there are some
visible transition artefacts. With a little more work you could hide the transition.
Make it less destructive.

So here is the sequence of commands that I am using on every LCD info modification.

First of all I am putting the module in 4 bit mode no matter the state of
the module, reset the module and then put it in my usual state ready to receive
the character data.

To do that we have to fix some of the hardware.

Basically we ground the pins for:

* R/W
* Bit0
* Bit1
* Bit2
* Bit3

In this setup we mess only with:

* RS
* Bit4
* Bit5
* Bit6
* Bit7

For setup phase we assert RS to 0.

Also I will only write the nibble that we send.

First step put the module is to resynchronize the bit mode.
You can see the long story in the wikipedia article.

        0 0 1 1
        0 0 1 1
        0 0 1 0

Now that we have synchronize the communication we put the LCD in 2 lines, 5x8 font.

        0 0 1 0
        1 0 0 0

Clear display.

        0 0 0 0
        0 0 0 1

The cursor is increasing the position and the display will not scroll.

        0 0 0 0
        0 1 1 0

Finally we turn the display on, hide the cursor and stop the cursor blink.

        0 0 0 0
        1 1 0 0

Now we assert RS to 1 so we can populate the display.

This setup will not create a very pretty picture or production state,
but the display will output exactly what you write. After the write you don't
need to care about the state. The state will be reset on the next write anyway.




