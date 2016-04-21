---
layout: post
title:  "Consistent Hashing"
date:   2016-04-07 08:35:00 +0000
tags:
- scala
- akka
- distributed systems
---

Consistent hashing is useful when creating distribute key value stores.

Assume a very simple (and useless) hash function that outputs `4 bit` hashes. More concretely, for any given key the hash function will return an integer between `0` and `15` (inclusive). This makes the hash space easy to reason about because `2^4=16`. This does, however, limit us to at most `16` members in our Akka cluster.

In order to determine where a given key value pair will be stored we can do `hash(k) % n` where k is the key and n is the number of servers. For example:

```
0 -> 0, 4, 8, 12
1 -> 1, 5, 9, 13
2 -> 2, 6, 10, 14
3 -> 3, 7, 11, 15
```

Whilst that works it's not very easy to reason about larger hash spaces. Further it's not very efficient if we assuming we want to move data around when failures occur:

```
0 -> 0, 3, 6, 9, 12, 15
1 -> 1, 4, 7, 10, 13
2 -> 2, 5, 8, 11, 14
```

Assuming a full store with no collisions then that would be at a minimum 12 moves. If instead we slice the hash space into sections:

```
0 -> 0, 1, 2, 3
1 -> 4, 5, 6, 7
2 -> 8, 9, 10, 11
3 -> 12, 13, 14, 15
```

Then we reduce the minimum moves to 7 and make the distribution of key value pairs easier to reason about:

```
0 -> 0, 1, 2, 3, 4
1 -> 5, 6, 7, 8, 9
2 -> 10, 11, 12, 13, 14, 15
```

The function becomes `min(n - 1, hash(k) / (16 / n))` where k is the key and n is the number of members. Note that `16` is the size of the hash space i.e. `2^4=16`. So more generally: `min(n - 1, hash(k) / (s / n))` where s is the size of the hash space.

Does `hash(k) / ()(s / n) + 1)` just work? Is it just that it's slightly different when s / n is odd vs. even?
