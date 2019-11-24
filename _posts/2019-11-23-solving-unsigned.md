---
layout: post
published: true
title: Solving Overflow
subtitle: Solving underflow and overflow
date: '2019-11-23'
---
> Note: I didn't really want people to read a refresher on X (my theory language) to understand any one of these posts so I'll just go over concepts each time (but keep it short and simple) :).

## What is overflow?

> Note: I'm purely going to be talking about integer overflow/underflow in the context of signed and unsigned integers for arithmetic but I will mention other forms of integers such as size_t (an integer capable of holding a pointer address) and how we may want the overflow to work there.  I'll also just talk about `overflow` because the exact same principles occur to underflow.

Overflow (and underflow) is where you perform an arithmetic operation and the resultant value is too large (or too small) to be stored.

For example if we have an 8 bit unsigned integer 254 we can store it like;

```
| 1111 1110 |
```

> I will store all bytes in nibble groups (i.e. groups of 4 bytes).  I will also only care about unsigned integers for the first little bit.

Now if we tried to add 1 to this then we get the following (just add each bit one at a time)

```
| 1111 1110  |
|          + |
| 0000 0001  |
|          = |
| 1111 1111  |
```

Let's then go further and try to add one more

```
|  1111 1111  |
|           + |
|  0000 0001  |
|           = |
| 10000 0000  |
```

And we have an extra little bit just dangling at the front!  This is problematic since well we only have 8 bits we can't just magically extend it.  So there are a few ways computers can handle this kind of overflow.  What will happen is that this extra bit is stored inside the 'carry' flag in most architectures (we will use x86 in this post but it should really apply to anything)

### Wrapping Overflow

Wrapping overflow is effectively just going to completely ignore the carry flag and wrap around the results for example `255 + 1 = 0` as we saw.  However `255 + 2 = 1` since;

```
|  1111 1111  |
|           + |
|  0000 0010  |
|           = |
| 10000 0001  |
```

See how the first bit isn't touched so it remains the same value i.e. ignoring carry we get just 1.

This is the most common implementation and is effectively represented by a modulus relationship (sometimes it is called modulo wrapping) for 8 bit unsigned integers it is effectively saying `result = (a + b) % 256` and in more general terms `result = (a + b) % (2 ** n)` where `n` is the number of bits in your unsigned integer.

For signed integers we can instead use the overflow flag for detecting overflow since it'll be toggled if the CPU detects that the result was impossible given the circumstances i.e. if you add a positive with a positive number and get a negative then clearly we have overflowed.

> Modern architecures use 2's complement for signed numbers and wrap it as mentioned above.  However relying on this behaviour is always a bit interesting for architectures that use a different form of signed integers.  For the sake of simplicity I'm going to presume 2's complement (and you can always follow in rust's footprints and implement ways such that you can get this behaviour even on systems that use different methods).

### Saturating Overflow

Effectively if our value becomes too large then it'll remain at the largest possible value and if it becomes too small then it'll remain at the smallest possible value (and so on).  For example in our case of `255 + 2` it'll become just `255` instead of `1` (i.e. .

Same for signed integers the only difference is the smallest is going to be the smallest negative number and that for 8 bit integers our largest is going to be `127` so `127 + 1 = -128` and `127 + 2 = -127`

## What is the problem?

Signed overflow is almost always a bug.  C understood this position all the way back when it was created and it stated that thus signed overflow was undefined behaviour or 'UB'.

UB basically means the compiler can do whatever it wants (within reason) for example in gcc if you have `for (int i = 0; i >= 0; i++) {}` the gcc compiler will most likely just transform that to just;

```c
int i = 0;
while (1) {
    i++;
}
```

> And it'll probably remove the `i` as you'll see since that is not actually used

i.e. since signed overflow is undefined it will think that `i` must always be greater than or equal to 0 if it begins at 0 thus it will loop forever!

To confirm you can check [here](https://godbolt.org/z/Q7n_t_ "Overflow Loop")

Specifically just look at the right hand side it is just;

```asm
test1:
.L2:
        jmp     .L2
```

Which should be pretty simple to understand even if you don't know assembly all it does is just begins at `test1` then performs a `jmp` back up to the top of `test1` (the `.L2` is called a 'label' and is basically just a byteoffset in the file) and if we write it out in hexdump we get

```asm
400420: eb fe 400420
```

The `eb fe` is just our `jmp` instruction and the `400420` on the right side is the location to jump to and the one on the left side is just what instruction we are at.  So it'll jump to itself basically which creates an endless cycle!

Now this does create a lot of places that the compiler can optimise but it does cause other issues.  The biggest issue is the programmer has no clue that this is occurring meaning it's hard to catch the bug.  So let's look at some potential fixes for signed integers.

> Note once more that unsigned integers almost always are well defined for overflow in all languages (for example both C and Rust allow it) this is because there are a lot of cases that it is useful and unsigned integers aren't used as a general purpose number so bugs with them are rarer.

## Fix 1) Always wrap/saturate

Well clearly the first issue with this fix is what do we choose?  Do we wrap or do we saturate?  Maybe we let the user decide?  Regardless what one we choose we still have the problem that if the user chose to wrap they probably have a bug they didn't intend for.

So this isn't great but maybe it would be nice to allow them to opt in for this!

## Fix 2) UB

Not a fix I would take.  But C took this.  Just say it can't happen.

## Fix 3) Error

If overflow / underflow occurs be angry at the programmer and error out.  This is quite a nice solution but it is expensive since you encur a branch/jump (typically a `jno` or `jo` flag for signed integers where `jno` means jump if no overflow and `jo` means jump if overflow) this isn't terrible in a lot of code especially on debug builds (and especially since we can branch predict that the jump towards the case of overflow won't occur) but it's not great.

## Rust's Solution

Rust takes the solution that overflow for signed integers is an error on debug builds but on production it will wrap (and isn't checked).  Yeh... while I would argue this is better than C in some ways it isn't perfect.

## X's Solution

My new theoretical language `X` solves this differently.  Effectively it states that all signed integer operations return an option of either `()` (empty tuple effectively 'null') or the actual result.  This means that every operation can effectively fail...  Now this may seem like chaining operators will get really really really tiresome but it alleviates this by stating that if it is 'obvious' that the result won't overflow then there is no need for this special handling.

For example;

```swift
x := 0;
x += 1; // this is fine it is obvious it won't overflow

i := 0;
while (i >= 0) {
    i += 1;
    // this will give an error
    // that i >= 0 may overflow and needs to be handled
}
```

This has no runtime cost since we are just stating you need to tell us how to handle it.  We aren't checking it at runtime we are just checking if it may occur at compile time.

### Case 1) No check

```swift
#no_overflow i := 0;
while (i >= 0) {
    i += 1;
}
```

`#no_overflow` is a 100% opt in 'C' way of handling it.  Effectively you are stating that there is no overflow possibility on `i` and then most likely this will get optimised to the code you saw before!  Now clearly this isn't particularly safe and should only be used sparingly (i.e. to optimise a hot path of code).

> Note: `#no_overflow` also applies to underflow we just call both of them integer overflow (perhaps a bit confusing).

### Case 2) Check

```swift
#check_overflow i := 0;
while (i >= 0) {
    // will error when this gets too big
    i += 1;
}
```

This will encur an error in the case it overflows (or underflows of course).  I'm considering allowing a flag that effectively defaults all signed integers to this (or default it in the language) but I haven't decided yet.

### Case 3) Handle using saturation or wrapping

```swift
#saturating_overflow i := 0;
while (i >= 0) {
    // will technically always be >= 0
    // since when it gets too big it'll just stay
    // at int max.
    i += 1;
}

// and
#wrapping_overflow i := 0;
while (i >= 0) {
    // will eventually be < 0
    // and stop executing
    i += 1;
}
```

> Where `#saturating_overflow` does overflow saturation (i.e. remain at largest/smallest) and `#wrapping_overflow` does overflow wrapping (i.e. `127 + 2 = -127` for 8 bit signed integers)

In this case the user probably never expected i to get large enough to even possibly overflow so the best choice is probably error but maybe they did?  Maybe they wanted to print out all positive numbers in the signed range and so they thought this was a quick way of doing it and so wrapping_overflow is the better choice in that case.

Regardless they are forced to choose the option that best fits their case!

## Unsigned overflow unintentionally

A big case of errors in C is something like;

```c
int *getDistArray(int64_t n) {
    int *array = malloc(sizeof(int) * n);
    // do whatever...
    return array;
}
```

Now the problem is if `n * sizeof(int)` overflows then even though `n` is too large for the array to allocate enough space `n * sizeof(int)` maybe be small enough that the allocation can be done!  This would then lead to a buffer overflow oh no.

> C and C++ definitely has this issue and I believe Rust but it seems much harder for Rust to 'show' it so to speak.  It also has the problem if n < 0!

X fixes this problem by defining it's equivalent of sizeof to a special type called ptr_size (awful name but basically size_t) the special part of ptr_size is that it will never promote to another type (you can manage to do it using a special conversion function but it has error bound checking).  The other special part is it's defined with `#saturating_overflow`!

This means that if you do `size * n` where n is a 64 bit int in X what will happen is that not only will `n` be converted to ptr_size but if the result of `size * n` is going to overflow then it'll just remain at max ptr_size.  This will always fail since MAX_PTR_SIZE is left as a special sentinel value to represent an impossibly large allocation (or address) kind of like how 0 represents a NULL address.

In the case that you do `n - size` and the result would underflow then what will occur is that it'll saturate at `0` which in X is not a valid allocation.  This is a bit different to allocations in C which often allow `0` as a size.  You could still allow 0 by either checking it or grabbing the error it returns and ignoring it (instead of propagating it) and maybe using an optional to store the fact it has a size of 0 (in the case of an array or something).  You typically won't run into this issue though in X since you'll be using the standard library arrays (since we typically don't do raw allocations in X) and those handle the 0 sized arrays as you would expect.

> No exceptions in X just error codes that are done very nicely.

Personally I love this solution since it provides a ridiculous amount of flexibility while also maintaining a huge amount of speed and forcing the user to be explicit.  My only concern is that it'll just lead to people putting `#no_overflow` (no!) or `#check_overflow` (good but eh verbose) all over the place.  So that's why I think having it always default to check (potentially with / without compiler option) is a really good idea!

Anyways that is me signing off for tonight...  I hope you learnt something and I would be interested in any opinions you hold around X's way of handling overflow.
