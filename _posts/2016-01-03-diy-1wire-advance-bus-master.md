---
layout: post
title: "DIY 1Wire advance bus master."
description: ""
category: 
tags: ["automation", "1wire", "PIC"]
---
{% include JB/setup %}

# The need

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

So I have made my own 1wire master.

# The requirements.

The goal is to allow a UART that does 9600 8n1 to communicate on the 1 Wire bus.

## The goal was to be dead simple to use.

My bus driver has a very simple state.

1. After the reset it will wait for a 0x00 from the host to activate,
this is done so that when the tp-link UART is initially set to 56kbps,
and it's logging the boot procedure, it will be ignored.

2. After the receiving of the 0x00 it will be in the operating state, in this
state it receives commands, there is only one type of command.

That's it.
There is no state that passes from one command to the next.

## The bus driver has to be able do do a strong pull-up.

Here is the schematic.:

![alt text](/assets/schemas/Uart21Wire.png "Schematic for Uart <-> 1Wire")

It is kind of a steal from that APP244.

## The driver has to be easy to reset/resync.

No matter in what state the driver is, the host can issue 18 null bytes (0x00).
It will receive 17 or 18 bytes back, after that, the driver is expecting a brand
new command.

This is possible because 0x00 is a valid command that does nothing on the 1wire
bus, except issuing a strong pull-up.

Also by issuing 0x00 we minimize the side effects that we might send on 1wire.

# The command.

A command is composed from 1 to 18 bytes.
The host will receive one byte for each byte that the host sends.
The delay between the sent byte and the received one is less than one bit.

This means that the host cannot overrun the internal buffer in the driver.

Also the driver has no buffer to overrun :) .

A command has one command byte and a sequence of normal or search bytes.

## Command byte

The control byte has this meanining:

* the first nibble: is a bunch of flags.
  * bit 0: set to 1 if you need a reset pulse the 1wire bus.
  * bit 1: set to 1 if the following sequence of bytes is prefixed by one normal byte.
  * bit 2: set to 1 if the following sequence of bytes are search bytes.
Otherwise the following sequence of bytes are normal bytes.
  * bit 3: set to 1 if we don't want a strong pull-up.
* the second nibble: represents the raw length of the sequence of bytes.
To calculate the length of sequence of bytes you:
  * take the raw length
  * increment it if the bytes are search bytes
  * increment it if you have a prefix byte

The response to the control byte is:

* 0x7F: if host requested a reset pulse and there was a presence response.
This means that there are devices on the bus.
* 0xFF: otherwise.

## Normal byte.

Because one bit in 9600 8n1 is about 104 microseconds, it means that we have
time to:

* read the bit from the host
* write it on 1wire
* read the resulting bit on the 1wire
* write the resulting bit to the host

## Search byte.

Another thing that will be very helpful is to be able to search.
To do a search we have to communicate 3 1wire bits, 2 reads and 1 write.
We make a close shave here and push a 1wire bit to 66 microseconds.

3 * 66 = 198 microseconds, less than 2 bits on 9600bps.

So when we are searching, we receive from the host a byte and we care only for
the even bits (0,2,4,6).
These bits are tiebreaker in case the search can go either way.

We also do this to give the driver time to report back the 2 1wire bits that
it is reading on the search, which yields 2 bits for the host.

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

# Functionality.

On 1wire you want to write, write 0xFF to read a byte or search something.

## Writing/reading bytes on 1wire

Just segment the sequence in segments of at most 16 bytes.

* For each segment of 16 bytes you write:
0xFC (or 0xFD if you want to send a reset pulse also) and those 16 bytes.
* For each segment of less then 16 bytes you write:
0x{number of bytes}8 (or 0x{number of bytes}9 with reset pulse request) and the data bytes.

Note: you can, for instance write 0x0C for writing a single byte, instead of
0x18 and this is ok.

## So how to use the search.

Usually, before you do an actually search, you write a search byte command on the 1wire bus.
Like 0xF0. That is why the prefix byte comes in handy.

For instance if you like to select the first device on a 1wire bus you issue.

    0xFF 0xF0 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00

The way you get your response is helping you to enumerate the devices on the bus.

## How to do device enumeration.

You start to search for a device by using a sequence like:

    0xF6 (no reset, prefix, search, want strong pull-up) 0xF0 (or another search ROM command) 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00.

This sequence, if there is a device on the bus, will get you:

* the address of the device with the lowest address value.
* a virtual address that can be used to do another search that will get you the
next address, like a search token.

The found address is decoded by looking at the even bits in the response.
And if the search has not failed you have your address.

The search token is obtained by:

* take the response
* find the last bit pair in the sequence that is "0" "0" and the host bit for
that pair was also "0"
  * if you find this pair then change it to "1" "1" and then set the
remaining bits to "0".
  * if you don't find it then you have found the last device, the search token
should be now all "0" so you can start over with a new search.

# Closing thoughts

This driver will not do the heavy lifting for you. It will not calculate the
CRC8 of an address or anything else.

It is made to be simple and allow you to actually communicate at 9600bps over
1wire with your devices.

The asm [source code](http://hub.darcs.net/ETB/PIC_Apps/browse/Uart21Wire/Uart21Wire.asm).
