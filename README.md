# pir-8
A swashbuckling adventure in making a custom computer! YAAAGH!!!

For a long time I've had the unsettling urge to make a computer, I mean _really_ make it. I don't want to just slap a few bits of modern stuff together, I want to go deeper! Sure, using ICs is all well and good, but that doesn't quite do it for me. I want to go down to the level of making it from individual transistors!

## Status/Phase

I'm currently working through fleshing out the ISA, this includes details what registers there will be, what instruction I will offer.

## Goal

Erm... this is mostly just a bit of fun, so I don't really have a fixed end goal. I'd like to get a 'working' system built in hardware, fully out of transistors. How powerful that computer will be, I don't know, let's just see what happens. Of course, the classic benchmark of running DOOM isn't going to hapen any time soon; a more modest aim would be at least Turing complete.

## Simulation

I started talking about this project with some friends, one of them happened to be bored enough to start working on a [simulator](https://github.com/TheCatPlusPlus/pir8) for what I have specified.

## Inspiration 

One project that I'm glad exists is the [Megaprocessor](http://megaprocessor.com/index.html). This huge construction has an aim of trying to show quite visually how all the logic within is put together for educational purposes. In contrast, I shall be looking to shrink parts of the logic down, both interms of the transistor count and the phyiscal layout. 

Another great example of low level CPU design is [Ben Eater's YouTube channel](https://www.youtube.com/channel/UCS0N5baNlQWJCUrhCEo8WlA). He has a nice mini serioes where he builds up a very basic CPU on breadboard, though he uses nast ICs. It's a great example of a neat and tidy system, and really helps you understand how the interanl control signals within the CPU can work.

There are lots of other small projects I've seen that have all helped influence me here. I've also taken a look at existing comercial designs. RISC-V to some extent, the iconic 8080 and even PIC.

## Contributions

If you see anything wrong, like I've accidentally specified two operations using the same binary code, or I've a design flaw in the adder wrong, I would massively appreciate you giving me a message ASAP. Down the line, when I start looking to write code, contributions would be cool, once I have real hardware I could maybe video it executing; amazing.
