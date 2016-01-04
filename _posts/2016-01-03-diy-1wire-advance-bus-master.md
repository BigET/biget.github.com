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

So for any byte that you send you should receive exactly one byte.

Here is the schematic.:

![alt text](/assets/schemas/Uart21Wire.png "Schematic for Uart <-> 1Wire")

It is kind of a steal from that APP244.

The goal is to allow a UART that does 9600 8n1 to communicate on the 1 Wire bus.

Because one bit in 9600 8n1 is about 104 microseconds, it means that we have
time to:
* read the bit from the host
* write it on 1wire
* read the resulting bit on the 1wire
* write the resulting bit to the host

Another thing that will be very helpful is to be able to search.
To do a search we have to communicate 3 1wire bits. We make a close shave here
and push a 1wire bit to 66 microseconds.

3 * 66 = 198 microseconds, less than 2 bits on 9600bps.

So when we are searching we receive from the host a byte and we care only for
the even bits (0,2,4,6).
These bits are tiebreaker in case the search can go either way.

The byte that the host receives is the 1wire response for the bit and negated
bit that the 1wire bus will have on a 1wire search.

So for every even bit that the host sends we have this transition table:

* Bit from host, first bit read from 1wire, second bit read from 1wire => third bit that will be written to 1wire
* 0, 0, 0 => 0
* 0, 0, 1 => 0
* 0, 1, 0 => 1
* 0, 1, 1 => 1 <- the search failed
* 1, 0, 0 => 1
* 1, 0, 1 => 1 <- the search failed
* 1, 1, 0 => 1
* 1, 1, 1 => 1 <- the search failed

When the search has failed it means that for the remaining bits in the search
the host will get only "1"s.

To be able to distinguish between normal and search bytes we institute the
control byte.

The control byte has this meanining:

* the first nibble: is a bunch of flags.
  * bit 0: set to 1 if you need a reset pulse now.
  * bit 1: set to 1 if the following sequence of bytes is prefixed by one normal byte.
  * bit 2: set to 1 if the following sequence of bytes are search bytes.
  * bit 3: set to 1 if we don't want a strong pull-up.
* the second nibble: represents the raw length of the sequence of bytes.
To calculate the length of sequence of bytes you:
  * take the raw length
  * increment it if the bytes are search bytes
  * increment it if you have a prefix byte

The response to the control byte is:

* 0x7F: if host requested a reset pulse and there was a presence response.
* 0xFF: otherwise.

So how to use the search.

You start to search for a device by using a sequence like:

0xF4 (no reset, no prefix, search, want strong pull-up) 0x00 0x00 0x00 0x00
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00.

This sequence, if there is a device on the bus, will get you:

* the address of the device with the lowest address value.
* a virtual address that can be used to do another search that will get you the
next address, like a search token.

The found address is decoded by looking at the even bits in the response.
And if the search has not failed you have your address.

The search toke is obtained by:

* take the response
* find the last bit pair in the sequence that is "0" "0" and the host bit for
that pair was also "0"
  * if you find this pair then change it to "1" "1" and then set the
remaining bits to "0".
  * if you don't find it then you have found the last device, the search token
should be now all "0" so you can start over with a new search.


