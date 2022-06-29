---
title: Hash tables
date: 2022-01-31
layout: layouts/post.njk
---

Part of a series on trying not to fail [Professor Singh's](https://sites.cs.ucsb.edu/~ambuj/) data structures class. Disclaimer: I don't know what I'm talking about, and these derivations are to clarify my thoughts and are probably wrong.

### Hash functions

I've never been completely comfortable with modular arithmetic and probability theory, so hash tables are interesting. The goal of a hash function is to uniformly distribute a set of keys over the set of integers $$ \{ 0, 1, \dots, m - 1 \} $$.  If we're working with an infinite set like the set of integers, then a function like $$ h(k) = \text{mod}(h, m) $$ does so perfectly. Below, we'll make two assumptions.

1. The probability that a key maps, or _hashes_, to some $$ m $$, a _bucket_, is $$ 1 / m $$.
2. The probability that a key hashes to a bucket is not conditional on where the other keys ended up. Each application of the hash function is independent of the others.

Of course, things can go terribly wrong with a hash function in practice. When $$ m $$ shares a divisor $$ d $$ with a subset of the keys, those keys will hash to only $$ m / d $$ buckets. All keys that are a multiple of $$ m $$ hash to the first bucket, and because $$ d $$ divides $$ m $$, $$ d $$ also divides $$ m i + d j  = d j \pmod m $$, $$ i, j \in \mathbb{Z} \ge 0  $$.

### Separate chaining

During a sequence of insertions, the probability that any two keys collide at bucket $$ b $$ is $$ 1 / m^2 $$. Because there is no outcome (a sample point is a hash table with $$ n $$ items in its chains) where the same two keys collide in more than one bucket (a collision is mutually exclusive with other collisions), and there are $$ m $$ buckets, the probability of a collision in _any_ bucket is $$ m / m^2 = 1 / m $$.

There are $$ \binom{n}{2} $$ pairs of keys that can collide. If we let $$ C_{k_i,k_j} $$ be the indicator random variable that records if key $$ k_i $$ collides with key $$ k_j $$, then the expected number of collisions after $$ n $$ insertions is

$$ E(C) = E \left[ \sum_{0 \le i < j \le n} C_{k_i, k_j} \right ]= \frac{n (n - 1)}{2} \frac{1}{m} $$

So if we'd like to expect less than one collision,

$$ n (n - 1) \le 2 m \implies n \sim \sqrt{2 m} $$

From our assumption above, that keys independently hash to a bucket with probability $$ 1 / m $$, if the numbers of keys that hash to bucket $$ i $$ is $$ L_i $$, then the random vector $$ (L_0, L_1, \dots, L_m) $$ follows a multinomial distribution, and $$ E(L_i) = \frac{n}{m} $$.

This is the load factor $$ \lambda $$ of the hash table, the ratio of the the number of keys in the hash table to the table size. For $$ \lambda \le 3 $$, for example, we can expect less than or equal to 3 comparisons when searching for a key in the table on average, which is $$ \Theta(1) $$.

Given that a key hashes to bucket $$ B_i $$, the probability of retrieving a particular key from that chain is $$ 1 / l $$ where $$ l $$ is the length of the chain, if we assume all keys are equally likely to be accessed. The expected number of comparisons to find the key is

$$ \frac{1}{l} \sum_{i = 1}^l i = \frac{1}{2}(l + 1) $$

or about $$ \frac{1}{2} \lambda $$ on average. Of course, the search can take $$ O(n) $$ time in the worst case when all the keys hash to a single list. In fact, this is given by the pigeonhole principle. If the universe of keys is at least $$ n m $$, then the best we can do is put at least $$ n $$ keys in each of the $$ m $$ buckets.

### Probing

For probing hash tables, all of our buckets are super shallow, and are placed one after another in contiguous memory. In fact, the buckets are so shallow that they can only hold one key.

To insert a key, we apply the hash function and drop it in the bucket. If a key already exists there, we choose one of the other buckets at random and try again. There are $$ n $$ keys in the table, so the probability that we collide again is $$ n / m = \lambda $$. No matter, we can do this all day. If $$ P $$ is the number of probes before a successful insertion, inclusive (in other words, the number of array accesses), then

$$ P \sim \text{Geometric}(p = \frac{m - n}{m} = 1 - \lambda) $$

and so

$$ E(P) = \frac{1}{1 - \lambda} $$

If we average this expectation over all the values of the load factor from $$ 0 $$ up to an arbitrary $$ \lambda $$ using an integral we get

$$ \frac{1}{\lambda}\frac{1}{\ln{(1 - \lambda)}}  $$

If we assume all load factors are equally likely, we can view $$ \lambda $$ as a uniform continuous random variable. The above expression is the expectation of the new random variable we get with the transform $$ E(P) $$.

Mathematica tells me this converges for $$ 0 < \lambda < 1 $$, which makes sense. The load factor of a flat table can't me more than 1; in fact, there's a discontinuity in $$ E(P) $$ there. Of course, this performance isn't attainable in reality, because we can't reliably locate randomly placed keys. It would take $$ m $$ tries on average.

It seems reasonable that this is an upper bound on the performance of a flat table then.

Instead, we define a probing function, which tells us which sequence of buckets to use when we collide with a mancala stone.

$$ p(c) = \text{mod}(h(k) + f(c), m) $$

where $$ h $$ is our hash function, $$ k $$ is the key we're trying to place, and $$ c $$ is the number of collisions we've encountered so far. $$ f $$ is the strategy we use to resolve collisions.

- $$ f(c) = c $$ is called _linear probing_. We try buckets $$ 0, 1, 2, \ldots $$ to the right of our initial attempt. Linear probing produces _clusters_, however, because every collision in an unbroken run of full buckets will increase its length by 1 and the probability that a key hashes to it. So the number of comparisons we expect when resolving a collision is conditional on the number of collisions we've encountered so far.
- $$ f(c) = c^2 $$ is called _quadratic probing_. There's a clever proof from _Weiss_ (Theorem 5.1) that says that, "If quadratic probing is used, and the table size is prime, then a new element can always be inserted if the table is at least half empty." Quadratic probing tries to reduce the clustering that affects linear probing, but Weiss said something about _secondary clustering_ being concern. Keys that hash to the same bucket will try to resolve a collision using the same quadratic function, and this repeated checking is overhead.
- $$ f(c) = c \cdot h_2(k) $$ is called _double hashing_. $$ h_2(k) = p - \text{mod}(k, p) $$ where $$ p < m $$ and $$ p $$ is prime. $$ h_2(k) \in [1, p] $$ by design, because evaluating to zero would create an infinite loop. It's important that $$ p $$ is not a multiple of $$ m $$ because this allows only a fixed number of opportunities to place the key. Similarly, it's important that $$ [2, p) $$ is not a multiple of $$ m $$, so usually $$ m $$ is taken to be prime. This technique eliminates secondary clustering by making the resolution strategy a function of both the encountered collisions and the key.

### Perfect hashing and universal hash functions

It's possible to design a hash function that hashes to a given bucket at most 1 in $$ m $$ times, which means the probability that any two keys collide is at most $$ 1 / m $$. This is exactly the assumption we made above. (To self: Can you do better than $$ 1 / m $$?) This means that out of a set of hash functions $$ H $$ the proportion $$ \lvert H \rvert / m $$ is an upper bound on the number of hash functions for which two keys collide.

In particular, we can use our separate chaining derivation above, copied below.

$$ E(C) = E \left[ \sum_{0 \le i < j \le n} C_{k_i, k_j} \right ]= \frac{n (n - 1)}{2} \frac{1}{m} $$

While we made this derivation assuming we could fit multiple keys in a bucket, it works here too, if we discard the hash function that produces the collision and try again. If we let $$ m = n^2 $$, then

$$ E(C) = \frac{1}{2}(1 - \frac{1}{n}) < \frac{1}{2} $$

This means that the probability of one or more collisions using our universal hash function is less than $$ 1/2 $$, by Markov's inequality.

$$ P(C \ge 1) \le \frac{1}{2} $$

If we try a bunch our our hash functions, then, we can expect to find a function that produces _no collisions_ in 2 tries. That's constant time.

If we know the set of keys in advance, we can figure out where they hash to. If there are $$ l_i $$ key in bucket $$ i $$, we provide them with $$ l_i^2 $$ space, their own comfy tiny hash table, cycle through our hash functions, and find one that doesn't produce a collision. The tiny hash table allows us to not have to churn through $$ n $$ keys at a time, which would still take linear time even though we only have to try 2 hash functions on average. We can expect $$ n / m $$ keys. If we choose $$ m = 1 $$, that's 1 key on average.

From our derivation for separate chaining, we know that $$ E(L_i) = \lambda $$. If we want to know how much space we'll need for these tiny tables, we need the second moment of $$ L_i $$, $$ E(L_i^2) $$. The marginal probability density function of a multinomial distribution is a binomial distribution. Scrambling [for formulas](https://en.wikipedia.org/wiki/Binomial_distribution) on Wikipedia, and differentiating the m.g.f. of a binomial distribution twice with Mathematica, we find

$$ E(L_i^2) = \frac{n(n + m - 1)}{m^2} \implies 2 - \frac{1}{n} $$

if we take $$ m = n $$.

$$ E(L) = E \left [\sum_{i = 1}^m L_i \right] = m E(L_i) \implies 2 n - 1 < 2 n $$

$$ P(L \ge 4n) \le \frac{2n}{4n} = \frac{1}{2} $$

by Markov, again.

So no matter the hash function we pick from $$ H $$ to distribute our keys into tiny hash tables, it'll take two attempts to find one that gives us $$ \Theta(n) $$ space, on average.

Of course, this all demands that our set of keys isn't changing. If we insert a key that isn't in the known universe we have to destroy the whole getup and start again.
