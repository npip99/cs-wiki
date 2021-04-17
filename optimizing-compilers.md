## Optimizing Compilers

Let's consider the following program:

```c++
#include <iostream>
#include <vector>

int main() {
  std::vector<unsigned int> arr(100'000'000);

  arr[0] = 0;
  arr[1] = 1;

  for(int i = 2; i < arr.size(); i++) {
    arr[i] = arr[i-1] + arr[i-2];
  }

  std::cout << arr.back();
  return 0;
}
```

And now, let's compile it and time how long it takes to compute:

```
$ g++ fib.cpp 
$ time ./a.out
4126703330
real	0m1.129s <- The actual time it took in seconds
user	0m1.029s
sys	0m0.100s
```

But, what we might notice, is that passing a simple `-O3` for optimization, can cause the program to go much faster!

```
$ g++ -O3 fib.cpp 
$ time ./a.out
4126703330
real	0m0.309s <- The actual time it took in seconds
user	0m0.125s
sys	0m0.185s
```

How does this happen?

### Function inlining

First of all, note that `arr.size()` calls a function. Calling a function is quite expensive, it means

- Saving the current instruction pointer
- Creating a stack
- `jmp`'ing to the function's code
- Actually executing the logic
- Popping the stack away
- `jmp`'ing back to the original instruction pointer.

Of these, only the 4'th step actually involves useful work, the rest is useless work. Now, if 99% of the time is spent in step 4, then we're fundamentally
not losing very much time. However, in the case of `arr.size()`, "Executing the logic" is a single assembly instruction, meaning that step 4 is in-fact only 1/6th = 16% of the total work being done, meaning we spend 84% of our time in `arr.size()` doing absolutely nothing useful at all!

This is fixed very simply: By using function inlining!

```c++
template<typename T>
int vector<T>::size() {
  return this->internal_size;
}
```

Given this implementation of `arr.size()`, we can simply replace the for loop with

```c++
for(int i = 2; i < arr.internal_size; i++) {
```

Yes, `internal_size` is a private variable, but that only makes it illegal for _you_ to use it, the compiler can in-fact do whatever it wants and no one can tell it otherwise!

Of course, this is only helpful when the body of the function is tiny, but as it stands, a lot of functions have tiny function bodies! When the function is very large, inlining can massively increase the size of the executable, since rather than have a 4byte call to a 100byte function, every function call involves copying 100bytes of assembly into the point of function invocation. Sure people have big SSDs nowadays, but the CPU only has so much cache. And by inlining too many large functions, you end up constantly flushing the instruction cache of the CPU because there are too many instructions to keep track of. For this reason, generally optimizing compilers have a cutoff of around 15-25 lines of code / assembly instructions when deciding whether or not to inline a function call.

And, as you might have noticed, Function inlining is one of the biggest reasons why debugging in release mode is so difficult, your stacktrace has quite literally been optimized away! If you would like to use all of the other optimizations, but still have working stacktraces, feel free to pass in "-fno-inline-functions" into your compiler, and all of your functions will remain as-is. Remember that this will make your programs slower though.

### Loop unrolling

Now, function inlining is great and all, but in our example we mostly wrote all of our code in `main()`, and most deep computations that take a long time are often single functions that are taking a long time. So how do we help those?

Well, fundamentally, loops are the crux of the problem. Without a loop, it'd be impossible to write a program that doesn't terminate virtually instantly. However, we often can't really optimize the body of the for loop very much all on its own. So one of the most important strategies is _loop unrolling_, which greatly utilizes the fact that consecutive iterations of a for loop generally do approximately the same thing.

To unroll a loop, it's awfully simple, simply transform

```c++
for(int i = A; i < A + 50; i++) {
  do_stuff(i)
}
```

into,

```c++
for(int i = A; i < A + 50; i += 4) {
  do_stuff(i)
  do_stuff(i+1)
  do_stuff(i+2)
  do_stuff(i+3)
}
```

First of all, what's great, is that we can totally nuke the `i < A + 50` condition that used to occur between each for-loop body. We can do this because 50 is a multiple of 4, so we know that we can batch in groups of 4 and we'll still be okay. That's already a great optimization! But what if we iterated 53 times, where 53 wasn't a multiple of 4? Well, that's not too hard either,

```c++
for(int i = A; i < A + 53; i++) {
  do_stuff(i)
}
```

->

```c++
for(int i = A; i < A + 50; i += 4) {
  do_stuff(i)
  do_stuff(i+1)
  do_stuff(i+2)
  do_stuff(i+3)
}
// Last 3 calls at the end because 53 is not divisible by 4.
do_stuff(50)
do_stuff(51)
do_stuff(52)
```

But sometimes `n` isn't known at compile time though, what do we do then? Well, just take it mod 4!

```c++
for(int i = A; i < A + n; i += 4) {
  do_stuff(i)
}
```

->

```c++
int new_n = n - (n % 4); // Which, the compiler can easily transform into n & ~0b11, if you're worried about speed there
                         // It has all arithmetic with powers-of-2 hardcoded that way
// Now, we loop-unroll with our new-found multiple of 4!
for(int i = A; i < A + new_n; i += 4) {
  do_stuff(i)
  do_stuff(i+1)
  do_stuff(i+2)
  do_stuff(i+3)
}
// And, now we do the rest
for(int i = new_n; i < n; i++) {
  do_stuff(i)
}
```

If `n` is really large, this saves a lot of time since we're not comparing against the for loop condition every time! If `n` is small, well then we didn't spend very much time in the for loop anyway so it wasn't a big deal and we'll be taking about the same amount of time regardless.

If you're really into squeezing out every last drip of speed, check out [Duff's Device](https://en.wikipedia.org/wiki/Duff%27s_device) for an efficient way to combine the modular arithmetic and the single for-loop into a single instruction. This only matters when `n` is small, which can help increase speed if you call a function with small `n` millions of times. This is of course is done by the optimizing compiler whenever it unrolls a loop, but if you're writing your own optimizing compiler for a personal project, don't worry about it! It's really not very important!

And yes, powers-of-two are often used for loop-unrolling, this is to ensure that the modular arithmetic can be done with bitwise operations, rather than the much more expensive "remainder" instruction.

### Static Single Assignment Form

In order for the compiler to analyze your code (rather than simply perform abstract optimizing manipulations on it like function inlining and loop unrolling), it needs to turn it into something that's very simple and logical, so that programmed optimizing transformations can act on it. The way most optimizing compilers do this is by rewriting your code in _Static Single Assignment Form_, or SSA. Let's see an example of SSA:

```c++
X = 5
Y = 2 + X
Z = X * Y
W = Z + X
```

What are the attributes of SSA? Well, to be in SSA, every single variable must only be set _exactly_ once - after that, it must remain static. Additionally, it must be represented solely as binary operators (Well, simply because SSA acts on the resultant Assembly, not the original C). Thus, we notice a crucial property:

- Every variable will have between 0 and 2 dependencies. X = 5 has 0 dependencies, X = 2 * Y has 1 dependency, and X = Z + Y has 2 dependencies.

This means a few things:

1) We can freely move the creation of variable to X up to anywhere after its dependencies have been initialized, and we can also move X down to anywhere before the first variable that has X as a dependency, which might help us so we should keep it in mind.
2) If the variable is not an ancestor of the return value of the function, we can also safely remove the variable entirely.
3) If a variable X is identically equal to another variable Y (via X = Y), then we can remove 'X' and replace every usage of it with 'Y'.

Now, let's see how this simple concept can drastically reduce total instruction count:

- Fundamentally, fib(n) "reads" from arr[n-1], "reads" from arr[n-2], "computes" arr[n-1]+arr[n-2], "writes" to arr[n]
- Now, let's loop unroll:

```
"reads" from arr[n-2], "reads" from arr[n-1], "computes" arr[n-2]+arr[n-1], "writes" to arr[n]
"reads" from arr[n-1], "reads" from arr[n], "computes" arr[n-1]+arr[n], "writes" to arr[n+1]
```

And, now we translate to SSA Assembly:

```c++
A = arr[n-2]
B = arr[n-1]
C = A+B
arr[n] = C
D = arr[n-1]
E = arr[n]
F = D+E
arr[n+1] = F
```

Now, we can already start optimizing our code! The first way to optimize our code would be by replacing all uses of RAM with uses of registers, via application of property (3):

```c++
A = arr[n-2]
B = arr[n-1]
C = A+B
arr[n] = C
D = B
E = C
F = D+E
arr[n+1] = F
```

Now, via property (3) again, we can start wiping out variable duplication!

```c++
A = arr[n-2]
B = arr[n-1]
C = A+B
arr[n] = C
F = B+C
arr[n+1] = F
```

Awesome! 

Using property (1), we can also reorganize into

```c++
A = arr[n-2]
B = arr[n-1]
C = A+B
F = B+C
arr[n] = C
arr[n+1] = F
```

By using property (1) to reorganize the body of the for loop into a (Read Input, Compute, Write Output) format, it allows further more macro-analysis to be made. Such as how (A, B) of one iteration of the for loop, is in-fact equal to the (C, D) of the previous iteration of the for loop! That means that you can do the following transformation:

```c++
arr[0] = 0;
arr[1] = 1;

A = arr[0]
B = arr[1]
for(int i = 2; i < arr.size(); i++) {
  C = A+B
  F = B+C
  arr[n] = C
  arr[n+1] = F
  A = C
  B = F
}
```

Now, this is incredible! Previously, each iteration of the for loop would have involved 2 RAM (`arr[n-2]`, `arr[n-1]`) reads, and 1 RAM (`arr[n]`) write. Now, 2 iterations of the loop are accomplished in 2 RAM writes, and 0 RAM reads! RAM is far more expensive than arithmetic operations, so this is an incredible improvement!

When people say "The compiler turns C into Assembly that's just as good if not better than what I could write by hand", this is what they mean. Is there a faster way to write fibonacci numbers to an array by hand? You'd be hard-pressed to find one. And the transformations and processes behind the optimizations in-fact quite simple! And remember, the actual optimizing compiler would have unrolled 4-8 iterations of the loop, if not more, not just 2. So it would perform even better than the example shown here.

Addendum:

You might have noticed that we didn't include the modular arithmetic during loop unrolling. What if `arr.size()` was odd? The omission of modular arithmetic was simply done for brevity, in real life the optimizing compiler would indeed check parity and progress the computation by a single iteration if `arr.size()` was odd. (And in-fact it would switch-case into all 8 parities for a more realistic loop unroll of 8 iterations).

## SIMD

Another important optimization that optimizing compilers make is that of the _SIMD_ instruction set. This allows parallel computation of arithmetic operations. Now, fibonacci numbers are fundamentally sequential, since each element depends on the previous. However, what if we were simply filling an array with something like the sum squares of the index?

```c++
for(int i = 0; i < n; i++) {
  arr[i] = i * i + n; // Let's say we want arr[i] = i * i + n
}
```

Often, SIMD operations will operate on 4-tuples of 32bit integers. This is to utilize internal CPU 128bit registers used specifically for SIMD. Some processors also have 256bit registers for 8-tuples, but we'll just work with 4-tuples and loop unrolls of 4 for now. So first, as always, we unroll!

```c++
for(int i = 0; i < n; i++) {
  arr[i] = i * i + n; 
  arr[i+1] = (i+1) * (i+1) + n; 
  arr[i+2] = (i+2) * (i+2) + n; 
  arr[i+3] = (i+3) * (i+3) + n; 
}
```

Now, we can reorganize this (Internally, this happens via SSA Assembly, but we'll just manipulate the C code for readability)

```c++
for(int i = 0; i < n; i++) {
  SIMD_A = <i, i, i, i>; // Can be done in only a single set_simd(i) instruction
  SIMD_B = <0, 1, 2, 3>; // Can be done in only a single set_simd(0x00010203) instruction
  SIMD_C = SIMD_A + SIMD_B;
  SIMD_D = SIMD_C * SIMD_C;
  SIMD_E = <n, n, n, n>; // set_simd(n)
  SIMD_F = SIMD_D + SIMD_E;
  arr[i] = SIMD_F; // SIMD acts on 128bits here, so this overflows into setting arr[i+1], arr[i+2], arr[i+3] as well!
}
```

SIMD is just like normal binary operators, except it's done on a super wide register (128/256+ bit), and it also doesn't "carry the 1" across 32bit boundaries (Or whatever 2^n-bit bountries you request in your Assembly instruction).

In the original code, we had 6+4=10 additions, 4 multipliations, and 4 writes to RAM. With the new SIMD code, we only have 2 SIMD additions, 1 SIMD multiplication, 3 SIMD writes to registers, and 1 SIMD write to RAM. This ends up being _much_ faster than the original code.

(And nowadays newer CPUs are going even further, 16-tuples with 512bit registers combined with loop unrolls of 16!, will the need for speed ever end??? Hint: Never!)

