+++
date = '2021-02-16T10:00:00-05:00'
draft = false
title = 'Using the Gray Code to Optimize One-out-of-Many Proofs'
+++
*A performance optimization for zero-knowledge membership proofs based on the
One-out-of-Many committment scheme.*

## Background and Motivation

One-out-of-Many proofs enable a prover to demonstrate membership in a set
without revealing their specific identity within that set. These proofs are
fundamental to privacy-preserving systems, particularly in blockchain protocols
that require large anonymity sets.

The construction by Groth and Kohlweiss provides logarithmic proof sizes and
supports efficient batch verification. However, the computational cost of
verifying multiple proofs scales linearly with batch size, particularly for
scalar multiplication operations. This becomes a bottleneck in systems
requiring verification of numerous proofs simultaneously.

## The Computational Challenge

In the standard One-out-of-Many construction, verification requires computing N
polynomial evaluations (where N = n^m is the size of the anonymity set). Each
polynomial evaluation requires m scalar multiplications, resulting in N × m
total scalar operations per proof.

During batch verification of p proofs:
- Exponentiations over the set can be shared across proofs
- Scalar operations scale to p × N × m multiplications
- For even modest batch sizes, scalar operations become the dominant
computational cost

**In simpler terms**: Imagine you need to verify 100 people's membership in a
group of 10,000. For each person, you must perform a calculation for every
member of the group. While you can share some expensive operations across all
100 verifications (like preparing the group data once), you still need to do
thousands of smaller calculations for each individual proof. These "smaller"
calculations add up quickly—like death by a thousand paper cuts.

## Gray Code Optimization

While working on an implementation of the One-out-of-Many proof scheme, I
recognized that in the process of committing to the membership set, the
algorithm iterated each index of the set, and committed to the binary
decomposition of each integer position. At each step in the computation, a
number of multiplications needed to be performed, proportional to the number of
bits different (a.k.a the [Hamming
distance](https://en.m.wikipedia.org/wiki/Hamming_distance)) between the
current index and the previous.

Once I saw this, I saw an opportunity to use the [Gray
Code](https://en.m.wikipedia.org/wiki/Gray_code) to optimze the process and
remove the number of multiplications required. Gray code is a clever binary
encoding of integers, which guarantees that each subsequent integer in a
sequence can only differ by one bit. If you're following along, you'll start to
see the significance: if I can guarantee that each subsequent integer only
differs by one bit, then I guarantee that each subsequent step in the
computation only requires only one mathematical operation. 

This essentially enables iterative computation where each polynomial can be
derived from its predecessor by:
1. Dividing out a single term
2. Multiplying in a single new term

Using batch inversion techniques, the per-polynomial cost reduces from m scalar
multiplications to just 2 multiplications. This represents an improvement
factor of approximately m/2.

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

{{< img src="without-gray-code.png" alt="Without-Gray-Code" caption="Batch verify 100 proofs with Gray encoding: ~509ms / 100 proofs = ~5.09ms / proof" >}}

{{< img src="with-gray-code.png" alt="With Gray Code" caption="Batch verify 100 proofs without Gray encoding: ~1332ms / 100 proofs = ~13.3ms / proof" >}}

As you can see, I achieved nearly 3x speed-up. Furthermore, these improvements
scale with both batch size and the parameter m, making the optimization
particularly valuable for systems with large anonymity sets.

Concretely, this is roughly equivalent to a privacy transaction protocol
operating at ~200 transactions per second, with each transaction hiding in a
crowd of 4K decoys (a.k.a. ring size 4K).

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

---

*The complete paper with mathematical proofs and detailed analysis is available on [GitHub](https://github.com/phreaknik/papers/blob/master/efficient-one-out-of-many-proofs-using-gray-codes/rendered.pdf).*
