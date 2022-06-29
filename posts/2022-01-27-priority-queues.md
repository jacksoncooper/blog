---
title: Priority queues
date: 2022-01-27
layout: layouts/post.njk
---

Part of a series on trying not to fail [Professor Singh's](https://sites.cs.ucsb.edu/~ambuj/) data structures class. Disclaimer: I don't know what I'm talking about, and these derivations are to clarify my thoughts and are probably wrong.

All of the queues discussed below maintain the _heap invariant_ that the key at a given node is _less than or equal to_ the keys in its subtrees. We care about the performance of 5 operations on priority queues.

1. `enquque()` or `insert()`
2. `deque()` or `remove_minimum()`
3. `shrink_key()` and `grow_key()` which comprise `change_key()`
4. `merge()`
5. `find_minimum()`

`change_key()` assumes that we don't have to go hunting for the key. If our queue supports random access, we're usually given its offset from the root in memory.

### Binary heaps

The vanilla binary heap is a random-access representation of a complete or perfect binary tree. The latter is the _shape invariant_. If you number the nodes from $$ 1 $$ to $$ n $$ from top to bottom, left to right, you'll see ascending powers of two down the left edge of the tree. The $$ log_2 $$ of these labels will yield their depth, the number of edges you must traverse to reach them from the root. This implies that the height of a binary heap is $$ \lfloor \log_2(n) \rfloor $$.

`enqueue()`: There's only one place to insert without destroying the shape invariant of the heap. If the new key is less than the parent, we restore the heap invariant by swapping it with its parent.

Worst case: We make $$ \lfloor \log_2(n) \rfloor + 1 = \Theta(\log n) $$ comparisons and the key surfaces at the root. This occurs when we insert a key into a perfect binary heap.

Average case: Our keys are real numbers, so $$ P\{k_\text{new} < k_\text{parent}\} = 1/2 $$. If $$ C $$ is number of comparisons we perform, $$ C \sim \text{Geometric}(p = 1/2)$$ and $$ E(C) = 2 = \Theta(1) $$. So inserting into an infinite heap takes constant time.

`dequeue()`: To preserve the shape invariant, we replace the key at the root with the key labeled $$ n $$. We make 2 comparisons to exchange the parent with its smallest child and restore the heap invariant.

Worst case: $$ 2 \lfloor \log_2(n) \rfloor = \Theta(\log n) $$  comparisons.

Average case: $$ E(2C) = 4 = \Theta(1) $$. Like swimming a key, the probability that the parent is larger than its smallest child is $$ 1/2 $$. Each of these comparisons requires an additional comparison to locate the minimum child.

`change_key()`: `shrink_key()` and `grow_key()` require swimming and sinking the key as in `enqueue()` and `dequeue()`, respectively, so they're both $$ \Theta(\log n) $$ in the worst case and $$ \Theta(1) $$ on average.

`merge()`: Constructing a new perfect heap with $$ n = 2^{h + 1} - 1 $$  keys, inserting one after the other, requires $$ \sum_{i = 0}^h i 2^i = \Theta(n \log n) $$ comparisons. (To self: Is it reasonable to expect merging with insertion performs similarly if half of the heap is already constructed?) Constructing a new perfect heap by starting with $$ 2 ^ h $$ heaps of size 1 and sinking $$ 2 ^ {h - 1} $$ parents, and then $$ 2 ^ {h - 2} $$ parents, and so on, requires $$ \Theta(n) $$ comparisons. So heap construction is $$ O(n) $$.

`find_minimum()`: The smallest key is at the root, $$ \Theta(1) $$.

### _d_-Heaps

_d_-heaps are a random-access representation of a complete or perfect _d_-ary tree, $$ d \ge 1 $$. If $$ d = 2 $$, you get a binary heap. While swimming a key still requires only one comparison, sinking a key requires $$ d - 1 $$ comparisons to find the smallest child and $$ 1 $$ comparison to determine if the parent is larger. So while `enqueue()` is faster than binary heaps, `dequeue()` is slower. If you hold the number of nodes $$ n $$ fixed, and take the partial derivative of $$ d \log_d(n) $$ with respect to $$ d $$, the cost increases for $$ d \ge 2 $$.

`enqueue()`: Requires $$ \lfloor \log_d(n) \rfloor = \Theta(\log n) $$ comparisons in the worst case. $$ \Theta(1) $$ on average.

`deque()`: Requires $$ d \lfloor \log_d(n) \rfloor = \Theta(\log n) $$ comparisons in the worst case. $$ \Theta(1) $$ on average, because it takes $$ d - 1 $$ comparisons to locate the minimum child, a constant cost.

`change_key()`: $$ O(\log n) $$ and $$ \Theta(1) $$ on average.

`merge()`: $$ O(n) $$.

`find_minimum()`: $$ \Theta(1) $$.

### Leftist and skew heaps

If a leftist heap has $$ r $$ nodes in its right edge, then by definition no other path from the root to a leaf can contain fewer than $$ r $$ nodes. This means a leftist heap must have at least $$ 2 ^ {h + 1} - 1 = 2 ^ r - 1 $$ nodes, and so $$ n + 1 \ge 2 ^ r \implies \lfloor \log_2(n + 1) \rfloor \ge r $$. If we have two leftist trees with right paths containing $$ r_1 $$ and $$ r_2 $$ nodes, then

$$
\begin{aligned}
r_1 + r_2 \le 2 + \log_2(n_1) + \log_2(n_2) &= O(\log n_1 + \log n_2) \\
&= O(\log(n_1 n_2)) \\
&= O(\log (n_1 + n_2)) \\
&= O(\log n) \\
\end{aligned}
$$

For the last equality, please see [Jeff](https://cs.stackexchange.com/questions/10056/leftist-heap-determining-time-complexity).

`enqueue()`: Merge a heap of size 1 with the existing heap, $$ O( \log n )$$.

`dequeue()`: By the definition of a leftist heaps, all subtrees are leftist. Remove the root and merge the two leftist subtrees trees, $$ O(\log (n_\text{left} + n_\text{right})) = O(\log n) $$.

`change_key()`: Sinking and swimming doesn't work here because the leftist shape invariant isn't as strong as that of a binary heap. The heap could be a string bean in the worst case, $$ \Theta(n) $$. Imagine inserting a descending series like $$ 6, 5, 4, \ldots $$ The `merge()` algorithm only touches 2 nodes during each insertion, but the swap needed to restore the leftist shape invariant creates the worst-case leftist heap, a strand running along the left edge of the tree. If we wanted to shrink the key $$ 6 $$, we'd have to perform exchanges all the way up the tree.

`shrink_key()`: If we shrink the key, the heap invariant at that node will not be violated, although we've already established we can't swim it up the heap. Instead, we cut the subtree out of the heap. The parent will have a null path length of zero. The parent's right subtree is still leftist, so if necessary we only have to restore the shape invariant. If the null path length of the empty left subtree (-1) is less than that of the right subtree, we swap the subtrees. However, the shape invariant may still be violated at the parent's parent. For every subsequent swap, the null path length of the parent increases by one. If we stop after $$ s $$ swaps, the final corrected subtree has a null path length of $$ s - 1 $$. This path is no shorter than its right path, which means the subtree must have at least $$ n_l = 2 ^ s - 1 $$ nodes, so $$ s \le \lfloor \log_2(n_l + 1) \rfloor = O(\log n) $$. Then merge the two trees you obtained.

`grow_key()`: `shrink_key(∞)` followed by `deque()` followed by `enqueue()`.

`merge()`: This algorithm touches only the nodes on the right paths of the two parameters. We merge the tree with the larger root into the right subtree of the tree with the smaller root. The merge produces a root with two leftists subtrees, but the null path length of the right subtree may now be greater than the null path length of the left subtree. In this case, we swap the subtrees to restore the shape invariant. The null path length of the root is now one more than the null path length of the right subtree. Because we only follow the right path during each recursive call, we touch $$ r_1 + r_2 = O(\log n) $$ nodes. More precisely, it requires at most $$ r_1 + r_2 - 1 = O(\log n ) $$ swaps.

`find_minimum()`: Still at the top, thank goodness, $$ \Theta(1) $$.

Skew heaps are leftist heaps that assume that a merge has violated the shape invariant of the heap, and perform a swap. In other words, they have no shape invariant. In the worse case then, a merge is $$ \Theta(n) $$. (To self: Skipping the comparison _somehow_ gains them amortized $$ \Theta(\log n) $$ operations, except for `change_key()`, which the lecture slides say is $$ \Theta(n) $$, probably because we lose the shape invariant when pruning. But what about the amortized prune? A proof comes later in Chapter 11 of _Weiss_.)

### Binomial queues

A binomial queue is a forest of trees. Every tree in the forest has $$ 2 ^ h $$ nodes, $$ h \ge 0 $$, where $$ h $$ is the height of the tree, and no two trees have the same number of nodes. In other words, a binomial queue can be represented as a positive integer in base 2. Binomial queues get their name because the number of nodes at depth $$ d $$ in a binomial tree of height $$ h $$ is the binomial coefficient $$ \binom{h}{d} $$.

`enqueue()`: Add a tree of height 1 to the forest, and `merge()`. $$ O(\log n) $$. If we view the existing forest as a bitstring, the probability that we encounter $$ t $$ ones before the first zero is $$ 1 / 2 ^ t $$, assuming the independence of our observations. It's equivalent to tossing a coin until we get tails. The proportion of sample points with $$ t + 1 $$ leading ones is half that of $$ t $$ ones. These $$ t $$ trees requires $$ t $$ merges when we insert (`merge()`) a new tree of height 1 into the forest. So the expected number of merges $$ E(M) = \sum_{t = 0}^{\infty} t/2^t = 2 $$, or $$ \Theta(1) $$ on average. Exquisite.

`dequeue()`: Find the minimum key at the root of a tree with $$ 2 ^ h $$ nodes and remove it, forming $$ h $$ trees with $$ 2^0, 2^1, \dots, 2^{h - 1} $$ nodes. Merge the trees. $$ O(\log n) $$. To create a binomial tree of height $$ h $$, you must merge two binomial trees of height $$ h - 1 $$. The first $$ h - 1 $$ children of the binomial tree of height $$ h $$ do not change, and the last is a binomial tree of height $$ h - 1 $$.

`change_key()`: `shrink_key()` is easy, because we can just swim the key up using $$ O(\log n) $$ comparisons. `grow_key()` requires `shrink_key(∞)` followed by `deque()` followed by `enqueue()`, just like leftist heaps.

`merge()`: Any two trees in the forest can be merged by using the smaller root as the root of the merged tree and absorbing the larger root as one of its children. This takes exactly 1 comparison. But what about merging two forests? If we have two forests with $$ t $$ and $$ t + 1 $$ trees each, each as densely packed as possible (bitstrings with $$ t $$ and $$ t + 1 $$ ones), we require $$ t + 1 $$ merges. Because $$ n \ge 2^t - 2 $$, this operation is $$ O(\log n) $$.

`find_minimum()`: If the implementation keeps a reference to the minimum key, this is $$ \Theta(1) $$.
