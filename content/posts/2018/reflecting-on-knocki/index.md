---
title: "Reflecting on my time at Knocki (Haptic Inc.)"
date: 2018-01-04T11:08:20-05:00
draft: false
showToc: true
---
[Knocki](https://knocki.com) was my first startup, where I served as CTO and
led development of a device that transforms any surface into a smart
controller. Mount it to your bedside table and tap twice to control your
lights, or knock a secret pattern to arm your security system—think of it as
[The Clapper](https://en.wikipedia.org/wiki/The_Clapper) reimagined for modern
smart homes.

{{< img src="knocki-mounting.jpeg" alt="Knocki product photo" caption="Knocki device mounting to a wall." >}}

## The Technical Challenge

Building Knocki presented fascinating engineering challenges. The initial
assumption—that you could simply attach an accelerometer to an MCU and call it
done—proved woefully inadequate. Everyday surfaces experience constant
background vibrations: fridge compressor motors, HVAC systems, footsteps, doors
slamming, and countless other sources of noise that make distinguishing
intentional gestures from environmental interference remarkably difficult.

With unlimited power, sophisticated digital filtering could solve these
problems. But we didn't have that luxury. Operating on battery power meant
carefully managing wake cycles, gradually bringing the device to higher power
states only when necessary to process tap gestures and transmit events to our
servers. The solution required extensive DSP work and meticulous power
management design. When we finally achieved reliable gesture detection on
battery power, it felt like a genuine triumph.

## From Prototype to Production

Knocki was my first journey through the complete product lifecycle: ideation,
prototyping, fundraising, design, production, fulfillment, and customer
support. The early days were characterized by hand-assembled PCBs, blue wire
fixes, and plenty of hot air rework. We launched on
[Kickstarter](https://www.kickstarter.com/projects/knocki/knocki-make-any-surface-smart)
and became the largest successful project from Texas at the time, ranking in
the top 5 worldwide.

{{< img src="knocki-prototype.jpg" alt="Early Knocki prototype" caption="Early prototype. I forgot to add the ground fill layer when I ordered the PCB, so I had to manually wire up each ground connection 😆" >}}

As we scaled, I built and managed a multidisciplinary engineering team spanning
electronics, mechanical design, firmware, systems engineering, and web/mobile
development. I experienced firsthand the challenges of managing customer
expectations during the pre-order phase and the complexities of post-launch
support—troubleshooting bugs and discovering creative (sometimes unexpected)
ways customers used our product.

## Reflection

I've since left the [Haptic](https://haptic.co) team, but the technology lives
on. It's cool to watch as the idea gets integrated into high-tech furniture by
OEMs and integrated in large commercial installations around the world.
