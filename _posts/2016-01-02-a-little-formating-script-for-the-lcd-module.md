---
layout: post
title: "A little formating script for the LCD module."
description: ""
category: 
tags: ["bash", "ash", "automation", "lcd"]
---
{% include JB/setup %}

In my last [post](/2016/01/02/_putting-an-hd44-lcd-on-a-tp-link-wr740n/)
I have mentioned that the LCD writing script does no formating.

The formating is isolated in yet another layer.
So you can easily write on the multiple line LCD.

HD44 is made in such a way that the graphics memory hold 80 characters.
Commonly the different LCD sizes uses these layouts:

* 1 row {8,16,20,32,40} columns: first char starts at address 0
* 2 rows {16,20,32,40} columns: first line starts at 0, second line at 40,
no matter the actual number of columns presented on the LCD.
* 4 rows 20 columns: first line starts at 0, second line at 40,
third line at 20 and forth line at 60.

I have rarely seen a 40 columns LCD module used.
So I have done a small formatting script that will assume 4x20 character matrix.

If you have different sizes the lines will be trimmed.


    #!/bin/sh

    read -r Linie1
    read -r Linie2
    read -r Linie3
    read -r Linie4

    Linie1="`echo -n $Linie1"                    " | head -c 20`"
    Linie2="`echo -n $Linie2"                    " | head -c 20`"
    Linie3="`echo -n $Linie3"                    " | head -c 20`"
    Linie4="`echo -n $Linie4"                    " | head -c 20`"

    echo -n "$Linie1$Linie3$Linie2$Linie4"

Basically we take up to 4 lines, each line is padded up to 20 characters and
then trimmed to 20. And lastly, we interleave the lines.

And there you have it, easy.
