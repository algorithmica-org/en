# Implicit Static B-trees

This is a follow up on a [previous article](https://algorithmica.org/en/eytzinger) about using Eytzinger memory layout to speed up binary search. Here we use implicit (pointerless) B-trees accelerated with SIMD operations to perform search efficiently while using less memory bandwidth.

It performs slightly worse on array sizes that fit lower layers of cache, but in low-bandwidth environments it can be up to 3x faster (or 7x faster than `std::lower_bound`).

## B-tree layout

B-trees generalize the concept of binary search trees by allowing nodes to have more than two children.

Instead of single key, a B-tree node contains up to $B$ sorted keys may have up to $(B+1)$ children, thus reducing the tree heigh in $\frac{\log_2 n}{\log_B n} = \frac{\log B}{\log 2} = \log_2 B$ times.

They were primarily developed for the purpose of managing on-disk databases, as their random access times are almost the same as reading 1MB of data sequentially, which makes the trade-off between number of comparisons and tree height beneficial. In our implementation, we will make each the size of each block equal to the cache line size, which in case of `int` is 16 elements.

Normally, a B-tree node also stores $(B+1)$ pointers to its children, but we will only store keys and rely on pointer arithmetic, similar to the one used in Eytzinger array:

* The root node is numbered $0$.

* Node $k$ has $(B+1)$ child nodes numbered $\{k \cdot (B+1) + i\}$ for $i \in [1, B]$.

Keys are stored in a 2d array in non-decreasing order. If the length of the initial array is not a multiple of $B$, the last block is padded with the largest value if its data type. 

```cpp
const int nblocks = (n + B - 1) / B;
alignas(64) int btree[nblocks][B];

int go(int k, int i) {
    return k * (B + 1) + i + 1;
}
```

In the code, we use zero-indexation for child nodes.

## Construction

We can construct B-tree similarly by traversing the search tree.

```cpp
void build(int k = 0) {
    static int t = 0;
    if (k < nblocks) {
        for (int i = 0; i < B; i++) {
            build(go(k, i));
            btree[k][i] = (t < n ? a[t++] : INF);
        }
        build(go(k, B));
    }
}
```

It is correct, because each value of initial array will be copied to a unique position in the resulting array, and the tree height is $\Theta(\log_{B+1} n)$, because $k$ is multiplied by $(B + 1)$ each time a child node is created.

Note that this approach causes a slight imbalance: "lefter" children may have larger respective ranges.

## Basic Search

Here is a short but rather inefficient implementation that we will improve later.

```cpp
int search(int x) {
    int k = 0, res = INF;
    start: // the only justified usage of goto statement
           // as doing otherwise would add extra inefficiency and more code
    while (k < nblocks) {
        for (int i = 0; i < B; i++) {
            if (btree[k][i] >= x) {
                res = btree[k][i];
                k = go(k, i);
                goto start;
            }
        }
        k = go(k, B);
    }
    return res;
}
```

The issue here is that it runs a linear search on the whole array, and also that it has lots of conditionals that costs much more than just comparing integers.

Here are some ideas to counter this:

* We could unroll the loop so that it performs $B$ comparisons unconditionally and computes index of the right child node.

* We could run a tiny binary search to get the right index, but there is considerable overhead to this.

* We could code all the binary search comparisons by hand, or force compiler to do it so that there is no overhead.

But we'll pick another path. We will honestly do all the comparisons, but in a very efficient way.

## SIMD

Back in the 90s, computer engineers discovered that you can get more bang for a buck by adding circuits that do more useful work per cycle than just trying to increase CPU clock rate which [can't continue forever](https://en.wikipedia.org/wiki/Speed_of_light).

This worked [particularly well](https://finance.yahoo.com/quote/NVDA/) for parallelizable workloads like video game graphics where just you need to perform the same operation over some array of data. This this is how the concept of *SIMD* became a thing, which stands for *single instruction, multiple data*.

Modern hardware can do [lots of stuff](https://software.intel.com/sites/landingpage/IntrinsicsGuide) under this paradigm, leveraging *data-level parallelism*. For example, the simplest thing you can do on modern Intel CPUs is to:

1. load 256-bit block of ints (which is $\frac{256}{32} = 8$ ints),

2. load another 256-bit block of ints,

3. add them together,

4. write the result somewhere else

…and this whole transaction costs the same as loading and adding just two ints—which means we can do 8 times more work. Magic!

So, as we promised before, we will perform all $16$ comparisons to compute the index of the right child node, but we leverage SIMD instructions to do it efficiently. Just to clarify—we want to do something like this:

```cpp
int mask = 0;
for (int i = 0; i < B; i++)
    mask |= (btree[k][i] >= x) << i;
int i = __builtin_ffs(mask) - 1;
// now i is the number of the correct child node
```

…but ~8 times faster.

The algorithm:

1. Somewhere before the main loop, convert $x$ to a vector of $8$ copies of $x$.

2. Load the keys stored in node into another 256-bit vector.

3. Compare these two vectors. This returns a 256-bit mask in which pairs that compared "greater than" are marked with ones.

4. Create a 8-bit mask out of that and return it. Then you can feed it to `__builtin_ffs`.

This is how it looks using C++ intrinsics, which are basically built-in wrappers for raw assembly instructions:

```cpp
// SIMD vector type names are weird and tedious to type, so we define an alias
typedef __m256i reg;

// somewhere in the beginning of search loop:
reg x_vec = _mm256_set1_epi32(x);

int cmp(reg x_vec, int* y_ptr) {
    reg y_vec = _mm256_load_si256((reg*) y_ptr);
    reg mask = _mm256_cmpgt_epi32(x_vec, y_vec);
    return _mm256_movemask_ps((__m256) mask);
}
```

After that, we call this function two times (because our node size / cache line happens to be 512 bits, which is twice as big) and blend these masks together with bitwise operations.

## Complete implementaiton

```cpp
#pragma GCC optimize("O3")
#pragma GCC target("avx2")

#include <x86intrin.h>
#include <bits/stdc++.h>

using namespace std;

typedef __m256i reg;

const int n = (1<<20), m = (1<<22), B = 16;
const int nblocks = (n + B - 1) / B;
const int INF = numeric_limits<int>::max();

int a[n], q[m], results[m];
alignas(64) int b[n+1], btree[nblocks][B];

int go(int k, int i) { return k * (B + 1) + i + 1; }

void build(int k = 0) {
    static int t = 0;
    if (k < nblocks) {
        for (int i = 0; i < B; i++) {
            build(go(k, i));
            btree[k][i] = (t < n ? a[t++] : INF);
        }
        build(go(k, B));
    }
}

int cmp(reg x_vec, int* y_ptr) {
    reg y_vec = _mm256_load_si256((reg*) y_ptr);
    reg mask = _mm256_cmpgt_epi32(x_vec, y_vec);
    return _mm256_movemask_ps((__m256) mask);
}

int search(int x) {
    int k = 0, res = INF;
    reg x_vec = _mm256_set1_epi32(x);
    while (k < nblocks) {
        int mask = ~(
            cmp(x_vec, &btree[k][0]) +
            (cmp(x_vec, &btree[k][8]) << 8)
        );
        int i = __builtin_ffs(mask) - 1;
        if (i < B)
            res = btree[k][i];
        k = go(k, i);
    }
    return res;
}
```

That's it. This implementation should outperform even the [state-of-the-art indexes](http://kaldewey.com/pubs/FAST__SIGMOD10.pdf) used in high-performance databases, though it's mostly due to the fact that data structures used in real databases have to support fast updates while we don't.

Note that this implementation is very specific to the architecture. Older CPUs and CPUs on mobile devices don't have 256-bit wide registers and will crash (but they likely have 128-bit SIMD so the loop can still be split in 4 parts instead of 2), non-Intel CPUs have their own instruction sets for SIMD, and some computers even have different cache line size.

Also, it may take some work to get it working with other data types or different comparators (you'll have to implement custom comparison using SIMD instructions), but in most cases this should be totally doable.




