+++
date = '2020-01-14T14:06:23-05:00'
draft = false
title = "Reflecting on my time at GridPlus"
+++
I recently left [GridPlus](https://gridplus.io/) after 3 years, where I served 
as tech lead building hardware cryptocurrency wallets that now secure assets for 
some of the world's largest crypto custodians. I wanted to take a moment to 
reflect on what was truly an exceptional journey.

{{< img src="gridplus-suite.webp" alt="Gridplus product photo" caption="The full Gridplus product suite." >}}

## The Technical Deep End

Working at GridPlus meant diving headfirst into some of the most challenging
and rewarding technical problems I've encountered. My days were filled with
hardware hacking, electronics prototyping, and porting cryptography libraries
to embedded firmware. I developed Java applets for chip-and-pin cards, designed
anti-tamper circuitry capable of detecting intrusion and destroying sensitive
keys, and helped build the first secure display featuring a dedicated embedded
GPU entirely enclosed within the device's anti-tamper zone. This work spanned
embedded graphics driver development, distributed networking, and extensive
blockchain wallet development at both firmware and application levels.

One of my favorite challenges was developing firmware that could interpret
contract data and render it clearly for user approval -- a critical security
feature that helps users understand exactly what they're signing.

{{< img src="gridplus-exploded.webp" alt="Gridplus Lattice1 exploded view" caption="Exploded view of the GridPlus Lattice1." >}}

## The Evolution of Our Vision

GridPlus started with an ambitious vision: building an automated agent to
perform blockchain transactions on behalf of users. The original use case was
compelling -- servicing our retail electricity provider business in Texas,
where we aimed to enable peer-to-peer electricity trading directly on the
blockchain. The GridPlus Lattice would remain always-online, automatically
signing transactions and negotiating energy deals behind the scenes for our
customers

To make this vision a reality, we needed to solve a fundamental problem: how to
build an always-online wallet that was still secure. We built a state-of-the-art
hardware wallet with configurable signing policies, spending limits, and 
anti-tamper circuitry designed to minimize the risk of both remote and physical
attacks. The result was a wallet far more sophisticated than anything else on 
the market.

Unfortunately, regulatory challenges prevented us from realizing our dream of
peer-to-peer electricity dealings. On the bright side, however, our hardware 
wallet had gained considerable traction in its own right. We pivoted fully into 
that space and successfully shipped a world-class hardware wallet that continues 
to secure cryptocurrency for some of the largest crypto custodians today.

## Reflections and Looking Forward

GridPlus was truly one of a kind. My time there came with all the challenges
and pivots you'd expect from a startup, but ultimately we found product-market
fit and shipped something I'm genuinely proud of.

I have high hopes for GridPlus's continued success, and I'm grateful for the
experience and the talented team I had the privilege of working with. I'm
excited to bring these lessons forward to my next role, working in Tesla's
Autopilot-Hardware group.
