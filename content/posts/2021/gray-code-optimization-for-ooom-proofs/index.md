+++
date = '2021-02-16T10:00:00-05:00'
draft = false
title = 'Using the Gray Code to Optimize One-out-of-Many Proofs'
+++
I recently discovered [an
optimization](https://github.com/phreaknik/papers/blob/master/efficient-one-out-of-many-proofs-using-gray-codes/rendered.pdf)
to the One-out-of-Many (OOM) proof protocol, and wanted to explain it in a less
formal setting. In this work, I use the Gray Code to drastically reduce the
mathematical complexity of the proving scheme. By extension, this optimization
applies to the many other proof systems built on OOM, including, but not
limited to: [MLSAG](https://eprint.iacr.org/2015/1098),
[CLSAG](https://eprint.iacr.org/2019/654),
[Omniring](https://eprint.iacr.org/2019/580),
[Lelantus](https://eprint.iacr.org/2019/373), [Lelantus
Spark](https://eprint.iacr.org/2021/1173),
[Triptych](https://eprint.iacr.org/2020/018),
[Arcturus](https://eprint.iacr.org/2020/312), and many others.

## Background and Motivation

[One-out-of-Many proofs](https://eprint.iacr.org/2014/764.pdf), by Groth and
Kohlweiss, is a widely adopted membership proof protocol. The OOM proof system
enables a prover to demonstrate membership in a set without revealing their
specific identity within that set (a.k.a the anonymity set). These proofs are
fundamental to many privacy-preserving systems, blockchain payment protocols,
mixnets, voting systems, or any application wishing to make user actions
indistinguishable from those of a large set of other users.

The OOM construction provides logarithmic proof sizes and supports efficient
batch verification. However, the computational cost of verifying multiple
proofs scales linearly with batch size, particularly for scalar multiplication
operations. This becomes a bottleneck in systems requiring verification of
numerous proofs simultaneously.

> It's worth noting that [recent advancements in
> zk-SNARKs](https://eprint.iacr.org/2019/1021.pdf) will likely render this
> class of membership proofs obsolete. I share my work as an improvement to
> existing systems, but I don't expect many new systems to employ OOM proofs.

## The Computational Challenge

In the standard One-out-of-Many construction, the bulk of the computational
complexity (for both prover and verifier) centers around creating a
cryptographic commitment to the anonymity set. This process requires computing
N polynomial evaluations (where N = n^m is the size of the anonymity set) to
compute the factors for each index in the set. Each polynomial evaluation
requires m scalar multiplications, resulting in N × m total scalar operations
per proof. These factors are then exponentiated with the elliptic group
elements corresponding to each member of the set, to create the set commitment.

During batch verification, the expensive exponentiations can be re-used for
each proof in the batch, but the number of scalar multiplications increases
from `N × m` for one proof to `p × N × m` for `p` proofs. **For even modest
batch sizes, scalar operations become the dominant computational cost.**

## Gray Code Optimization

While working on an implementation of the One-out-of-Many proof scheme, I
attempted to optimize the set commitment by transforming it to an iterative
computation. Rather than computing each membership commitment from scratch, I
would iteratively build each member commitment as a transformation of the
previous member commitment. Since each member commitment includes a commitment
to each bit for its position in the set, subsequent computations can re-use the
commitment for each bit that hasn't changed. For indices with only one bit
difference, the savings was huge. For other indices, where nearly every bit
changes, there was little/no incremental performance savings.

This connection to between bit transitions and computational complexity led me
to reconsider the binary encoding of each index in the set. I was reminded of
the [Gray Code](https://en.m.wikipedia.org/wiki/Gray_code), an alternative
encoding which guarantees that each subsequent integer differs from the
previous by only one bit. Consequently, each step in the iterative computation
need only compute the commitment for one bit, allowing maximal reuse of all
other bit commitments from the previous iteration.

For illustration, I put together a table showing the bitwise comparison of
binary coded vs Gray coded integers, and highlighted each bit transition in
red. Each red cell represents new work that has to be performed for that
position in the set, while the white cells can be reused from the previous
iteration. Notice the Gray code has significantly fewer red cells:

{{< img src="bit-transitions.png" alt="Bit Transitions Comparison PNG" caption="Comparison of binary and gray coded decimal numbers. The red cells indicate bits that have to transition when iterating from the previous integer. Notice the Gray code only ever has one bit transition per integer." >}}

I'd previously used this to optimize the layout of transistors in a
semiconductor design, but I'd never applied this as a computational
optimization. The computational savings was immediately obvious, but I needed
to be sure that the change in encoding would not impact the security of the
proof system. Fortunately, this turned out to be simple. Since the security
proofs depend only on the set indices and not their encodings, I am able to
apply the Gray code to the protocol construction without impacting the security
proofs 🎉


## Implementation Details

The modification requires minimal changes to the existing protocol:

1. **Index Encoding**: Replace standard indices with their Gray code
   equivalents using a mapping function gray_n(k)
2. **Iterative Evaluation**: Compute polynomials sequentially, leveraging the
   single-term difference property
3. **Batch Inversion**: Pre-compute all required inversions for approximately
   the cost of a single inversion

The resulting protocol maintains identical proof sizes and formats—only the
internal computation method changes.

## Performance Results

Testing with Curve25519 on standard hardware demonstrates substantial
improvements. For reference, I give the baseline performance of a
One-out-of-Many proof implementation **without** the Gray Code optimization,
and then for comparison, I show the same implementation modified to incorporate
the optimization. For this test, I am using a 12 bit anonymity set (4K members)
and a batch size of 100.

{{< img src="with-gray-code.png" alt="With-Gray-Code" caption="Batch verify 100 proofs with Gray encoding: ~509ms / 100 proofs = ~5.09ms / proof" >}}

{{< img src="without-gray-code.png" alt="Without Gray Code" caption="Batch verify 100 proofs without Gray encoding: ~1332ms / 100 proofs = ~13.3ms / proof" >}}

As you can see, I achieved nearly 3x speed-up in this scenario. This is roughly
equivalent to a privacy transaction protocol operating at ~200 transactions per
second, with each transaction hiding in an anonymity set of 4K other
transactions.

Thse improvements scale with both batch size and the set size, making the
optimization particularly valuable for systems with large anonymity sets and/or
high "batchability".

## Comparison with Alternative Approaches

It's worth noting the distinction between this approach and others. The Beam &
Firo projects [achieved a similar
speed-up](https://forum.firo.org/t/gray-code-optimization-for-lelantus/930/14)
in their implementations of Lelantus (built on One-out-of-Many proofs), but
they achieved this through a time-memory-tradeoff: their recursive approach
increases the number of heap allocations proportional to the computational
savings they achieved. In practice, this is acceptable for small sets, but the
increased memory requirements will begin to impact performance at larger set
sizes.

My proposed optimization using a Gray code achieves that same speed-up, but
without the memory tradeoff. This ensures the performance benefit wont degrade
at large set sizes.

## Security Considerations

The Gray code transformation is a bijective mapping that simply reorders the
elements in the computation sequence. From the protocol's perspective, this is
equivalent to a permutation of the anonymity set members. Since the original
One-out-of-Many construction's security doesn't depend on element ordering, all
security properties and existing proofs remain valid.

In essence, since the Gray code is just an encoding difference, it doesn't
impact the algebraic equations. The proofs remain unchanged, but the
implementation becomes easier.

## Applications and Future Work

This optimization technique applies to any One-out-of-Many proof implementation
and potentially extends to other cryptographic protocols involving polynomial
evaluations over ordered sets. Current applications include:

- Privacy-preserving cryptocurrencies (notably Firo's Lelantus protocol)
- Ring signature schemes
- Other zero-knowledge proof constructions with similar polynomial structures

The approach demonstrates that classical computer science concepts can provide
practical optimizations for modern cryptographic protocols.

## Conclusion

By applying Gray codes to the polynomial evaluation structure in
One-out-of-Many proofs, we achieve significant performance improvements in
batch verification scenarios. The optimization reduces scalar multiplication
operations by a factor of approximately m/2, resulting in up to 60% reduction
in verification time for realistic parameters.

This work provides a practical improvement to an important class of
zero-knowledge proofs, making large-scale privacy-preserving systems more
feasible without compromising security or increasing memory requirements.

The complete paper with mathematical proofs and detailed analysis is available
on
[GitHub](https://github.com/phreaknik/papers/blob/master/efficient-one-out-of-many-proofs-using-gray-codes/rendered.pdf).
You can also find my (for research/evaluation purposes only) implementation of
One-out-of-Many proofs with this Gray Code optimization on
[GitHub](https://github.com/phreaknik/one-of-many-proofs).
