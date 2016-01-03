---
layout: post
title: "DIY 1Wire advance bus master."
description: ""
category: 
tags: ["automation", "1wire"]
---
{% include JB/setup %}

The One Wire (tm) protocol is very good and flexible when you have low bandwidth
requirements.
And the devices are very low cost and more than good enough for home automation.
But in order to put large amounts of 1wire devices and over long wires you
have to get your hands dirty and create a special master that will handle
the analogue noise.

Fortunately those guys from maxim have a very nice document about this,
[APP 244](http://pdfserv.maximintegrated.com/en/an/REFD244.pdf).

I could have use a specialized chip like [DS2480B](https://datasheets.maximintegrated.com/en/ds/DS2480B.pdf).

But I have to say that is way to complicated to use it and it gives you to
many options that I don't need.
Also I wanted to be more aggressive with the strong pull-up.

Maybe some other time.

So I have made my own 1wire master. The goal was to be dead simple to use.

Here is the schematic.:

![alt text](/assets/schemas/Uart21wire.png)


