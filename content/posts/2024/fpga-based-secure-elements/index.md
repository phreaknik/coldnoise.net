+++
date = '2024-07-21T13:31:16-05:00'
draft = false
title = 'FPGA Based Secure Elements'
+++

I recently spoke with a colleague working on an open-source [secure
element](https://en.wikipedia.org/wiki/Secure_element) chip. While the goal is
admirable — enabling customers to verify designs for security vulnerabilities
and [backdoors](https://en.wikipedia.org/wiki/Backdoor_(computing)) — I'm
skeptical of the value proposition despite the hype and investor attention
they've received.

I get why people are so excited by the idea. The tech industry becomes more
interested in security practices with each major incident. People and
businesses alike are increasingly focused on trust models. Companies hire
auditors, run bug bounty programs, and sponsor open-source development to
minimize black-box dependencies. Cryptocurrencies have even made retail
customers security-conscious, scrutinizing their software more and even
reaching for hardware wallets to move their secret keys offline. An open source
secure element would give people a hardware tool to secure their keys, without
requiring them to trust yet another black-box in their tech stack... right?

## Secure Element ASICs Require Trust

As someone who's spent years building hardware wallets, RFID card applets, and
cryptographic libraries, I understand both the appeal and limitations of
open-source secure elements. Unfortunately, no amount of open-source design
files can address the fundamental issue: you cannot verify the integrity of the
semiconductor itself. You must still trust that:

* The RTL sent to fabrication matches the open-sourced version
* The fab doesn't introduce defects or vulnerabilities during production
* Your chip is genuine and from official sources

If you're seeking trustlessness, or if your trust model otherwise precludes
trusting the chip vendor then these trust assumptions are likely also
unacceptable to you.

So who benefits from open-source secure element ASICs? The value proposition is
unclear. Secure elements are designed for customers who accept trust in the
chip vendor—they need protection from remote and opportunistic physical
threats, not from advanced threats or supply chain attacks. For these
customers, whether the design is open-source is largely irrelevant since they
cannot verify the physical chip matches the published design anyway.

## FPGA-Based Secure Elements

Despite my criticisms, I am still an avid advocate for open-source design. The
ability to inspect and review a technology, before I use it, helps us minimize
trust and improve our online security.

This is only possible if I am able to inspect that the finished product matches
the open-source design. For example, if I can verify in any of the following
ways:

* Compile and deploy code from source
* Inspect deployed code against source (as with
[ROM](https://en.wikipedia.org/wiki/Read-only_memory))
* Compare the traces of a PCB to the schematic
* Assemble my own PCB/circuits at home

Unfortunately,
[ASICs](https://en.wikipedia.org/wiki/Application-specific_integrated_circuit)
cannot be fabricated or verified from home. You just have to trust what's in
the black-box.

FPGAs offer a solution. With secure element RTL and a compatible FPGA, you can
compile and deploy the bitstream yourself, ensuring the design matches your
expectations.

Caveats exist—many secure element features cannot be reasonably provided by
FPGAs (such as anti-tamper circuitry or
[PUFs](https://en.wikipedia.org/wiki/Physical_unclonable_function)). However,
core cryptographic operations and key storage—sufficient for most use
cases—work well on FPGAs. [The
Precursor](https://www.crowdsupply.com/sutajio-kosagi/precursor) demonstrates
this approach in practice, using an FPGA to provide secure key storage and
cryptographic operations with user-verifiable firmware.

Of course, FPGAs aren't a perfect solution—you still must trust the FPGA
manufacturer's silicon. However, the trust model differs significantly: you
control the bitstream compilation and deployment, and can verify what's loaded
onto the device. While an FPGA could theoretically contain backdoors that
analyze and modify your bitstream during configuration, the computational
complexity makes this impractical. Parsing and backdooring arbitrary RTL in
real-time would require resources orders of magnitude beyond what's available
in the die area around the gate array.

## Conclusion

Trusted secure elements remain valuable tools, but for those of us focused on
trust minimization, FPGA-based solutions offer a compelling alternative worth
exploring.
