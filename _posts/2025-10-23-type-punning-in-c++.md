---
layout: post
title: "Type Punning in C++"
---

A common need in systems-level and embedded programming is the ability to reinterpret the bytes of a variable of one type as another type. Unfortunately, this is much more of a challenge to do in C++ than you may believe.

A practical example includes the need to reinterpret a struct as an array of bytes, or an integer type encompassing all struct members to pass into a function that deals with generic binary data. 

A simple example looks like this:
```cpp
struct Color {
    uint8_t r;
    uint8_t g;
    uint8_t b;
    uint8_t a;
};
```

For serialization purposes, we may want to reinterpret this struct as a whole integer, i.e `uint32_t`, or an array of bytes, i.e `uint8_t[4]`.


This is incredibly common when it comes to low-level systems and networking protocols. A simple example involves having some convention for a standard message ID to always be the first byte of every serialized struct, so a receiving event loop can switch over it, know what struct to reinterpret it back to, and handle it accordingly.  


Despite how common this is in low-level programming, C++ has subpar support for this. Take a look at any low-level commercial or open-source code, and there's a good chance you will find yourself staring at a screenful of undefined behavior.


# Explicit (C-style) casts, or reinterpret_cast

A common way to deal with this is to use type cast, and very often this is a C-style cast. This is very common in embedded code written by programmers more accustomed to C-style programming.

Any article discussing type punning is obligated to bring up the Quake III fast inverse square root algorithm. The code is as follows:

```c
float Q_rsqrt( float number )
{
	long i;
	float x2, y;
	const float threehalfs = 1.5F;

	x2 = number * 0.5F;
	y  = number;
	i  = * ( long * ) &y;						// evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

	return y;
}
```

I won't cover how the algorithm works, but just the point relevant to this article. In this line:

```c
i  = * ( long * ) &y;						// evil floating point bit level hacking
```
The address of y, a `float *`, is cast to `long *`. Then, it is dereferenced, effectively reinterpreting the bits of the `float` y as a `long`. This algorithm specifically does this to perform bitwise operations on the float, which are not allowed otherwise.

In C++, it's important to note that because of the [explicit cast rules](https://en.cppreference.com/w/cpp/language/explicit_cast), this line is semantically equivalent to
```c++
i = *reinterpret_cast<long*>(&y);
```

Hopefully, the mere sight of `reinterpret_cast` should give you an idea of what I'm about to say: this is undefined behavior. Not only in C++, but C too.

### Strict aliasing rules
To cover why this is undefined behavior, we will have to turn to the language standard. From the [C++23 standard draft](https://timsong-cpp.github.io/cppwp/n4950/expr.prop#basic.lval-11):

> If a program attempts to access the stored value of an object through a glvalue whose type is not similar to one of the following types the behavior is undefined \
    (11.1) \
    the dynamic type of the object, \
    (11.2) \
    a type that is the signed or unsigned type corresponding to the dynamic type of the object, or \
    (11.3) \
    a char, unsigned char, or std​::​byte type. 

To summarize, accessing an object through a pointer of a different type that is not a type that is similar to the object's type is undefined behavior. 

Why? Because the compiler is allowed to assume that the object is only accessed through pointers close to its type, and optimize accordingly. Specifically, the compiler can assume that modifying some arbitrary `float *` shouldn't mean that the `int` you defined needs to be loaded back from memory into a register before you access it again.

As a side note, if you've ever heard about Fortran being more performant than C/C++, part of this is because Fortran makes this assumption for all pointers. C/C++ does not for pointers of similar types and thus the compiler will generate code that does a load after each write for them (unless you use [`restrict`](https://en.cppreference.com/w/c/language/restrict.html) in C).

Take this contrived example:
```cpp
#include <cstdint>

extern void send_msg(const uint32_t *data);

int main() {
    auto *data = new uint32_t;
    *data = 0;
    auto *word_ptr = (uint16_t *) data;

    word_ptr[0] = 2;
    word_ptr[1] = 3;

    if(*data != 0)
        send_msg(data);
}
```

The following code creates a `uint32_t` object on the heap, then creates a `uint16_t` pointer to it. It then writes to the `uint16_t` pointer (writing to the underlying `uint32_t` object), then checks if the `uint32_t` object is non-zero. If it is, it sends the data to a function.

At a glance, you might assume that the compiler should easily be able to tell that data is non zero, and thus the `send_msg` function should always be called, with the `uint32_t` object modified appropriately. 

The actual codegen generated by GCC 14.2 with `-O3 --std=c++23` is as follows (see on [Compiler Explorer](https://compiler-explorer.com/z/WPa657aGb)):
```assembly
main:
	xorl    %eax, %eax
	ret
```

The compiler assumes that `data` is always 0 despite the modification using the `uint16_t` pointer. This is not a compiler bug: with the standard rules defined before, the compiler is allowed to assume that the `uint32_t` object is only accessed through `uint32_t` pointers, and thus can optimize the code to always return 0.

Thus, there is no way to do this correctly in C++. Note that some developers even give up on this optimization entirely, and use flags like `-fno-strict-aliasing`, which then ensures that the above behavior is okay (for trivially constructible types - this will be elaborated on later). The [Compiler Explorer](https://compiler-explorer.com/z/WPa657aGb) link demonstrates the output code with and without this flag, where disabling strict aliasing has the expected behavior.

#### Alignment rules
Another factor when it comes to using this method is memory alignment. While this *is* possible to work around using `alignas`, it's often forgotten that the resulting pointer you're dealing with would need to obey the alignment requirements of your source type. Violating this results in undefined behavior.

```c++
#include <cstdint>
void unaligned_access_example() noexcept {
    unsigned char data[5];
    // Accessing a uint32_t from a misaligned address
    auto* misaligned_ptr = reinterpret_cast<uint32_t*>(data + 1);
    // Undefined behavior: misaligned access, not aligned to 4 bytes (uint32_t)
    *misaligned_ptr = 0;
}
```

The way each platform deals with unaligned memory accesses varies by architecture. x86-64 will silently allow an unaligned memory access with a performance penalty, but ARM will crash with a `SIGBUS` error. I have personal experience with the pain of running code that relies on this behavior on ARM for the first time.

#### Casting to std::byte and unsigned char

An important exception is made in the standard here:

> If a program attempts to access the stored value of an object through a glvalue whose type is not similar to one of the following types the behavior is undefined
>   ...
>   a char, unsigned char, or std​::​byte type.

This exception is clearly made to allow accessing the byte representation of an object. So, a reinterpret cast to `std::byte,` `char`, or `unsigned char` is allowed.

It's important to note that while the *cast* is allowed, a strict reading of the standard makes *accessing* the bytes technically undefined behavior. Specifically, there is no guarantee that the dereferenced pointer will point to the first byte of a given object. This is covered much more in depth by the proposed wording fix by [P1839](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p1839r7.html), which has already been [approved as a defect report](https://github.com/cplusplus/papers/issues/592), and hopefully should land in the language soon. As a defect report, it will apply retroactively to all versions of C++. In practice, all compilers treat this as well-defined behavior, and this is safe to rely on.


# Unions
Another idea is to use a union to type pun, such as in the following snippet.
```c++
union {
    struct {
	int8_t r;
	int8_t g;
	int8_t b;
	int8_t a;
    } components;
    int32_t num;
    std::byte bytes[sizeof(components)];
} rgb;

rgb.components = {.r = 1, .g = 1, .b = 1, .a = 0};
const auto rgb_code = rgb.num;
```

A union is supposed to share the same bytes between all of its members, so if we just access one member of the same size, we should effectively reinterpret the same memory as another type by accessing the other member of the union, as long as they are the same size.

This, too, is undefined behavior. Writing to a member of a union makes it the active member, starting the lifetime of that object. Reading from an inactive member means you will be accessing that member before its lifetime as an object has started, which incurs undefined behavior (see [\[class.union\]](https://timsong-cpp.github.io/cppwp/n4950/class.union#anon)).

Despite this, GCC (and many other compilers) do support this behavior as a [compiler extension](https://gcc.gnu.org/onlinedocs/gcc-9.3.0/gcc/Structures-unions-enumerations-and-bit-fields-implementation.html#Structures-unions-enumerations-and-bit-fields-implementation), so this is a very popular approach in a lot of code. Relying on compiler extensions, however, leads to non-portable code which is best avoided.


Something interesting is that C99 actually *does* define this behavior (see [section 6.5.2.3](https://open-std.org/JTC1/SC22/WG14/www/docs/n1124.pdf#page=85)), so this method of type punning in C is well defined. This is feasible because C doesn't have to worry about objects and lifetimes.

### Object lifetimes
The previous mention about object lifetimes introduce another complexity into type punning. The standard formally defines the start of an object's lifetime as when storage suitable for the object is obtained, and initialization is complete, meaning its constructor has finished - so either defined with automatic storage duration, as a temporary, changing the active member of a union, or a new expression. (see [\[basic.life\]](https://timsong-cpp.github.io/cppwp/n4950/basic.life)).

This is relevant, as objects that have not had their lifetime started yet are only allowed to do a very limited set of operations, and using it in basically any useful way (i.e accessing the object) invokes undefined behavior. So, we can't simply use a union (or reinterpret_cast) as a way to interpret an object as another object without ensuring the new lifetime has started.

#### Implicit Object Creation
One mitigation for a long-standing issue in the language added in C++20, which you may have picked up on from that previous statement, is implicit object creation (original proposal [here](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p0593r6.html)). This adds sane behavior to functions that necessarily need to be able to implicitly create objects - i.e, ones that deal with memory for objects that have constructors and destructors that do not execute code, or implicit-lifetime types. This applies to functions like `std::malloc`, `std::memcpy`, `operator new`, and other memory functions. This makes code trivial C code like this, which was technically undefined, defined once again.

```c++
#include <cstdlib>

void alloc_int() noexcept {
    auto *number = (int *) std::malloc(sizeof(int));
    *number = 0;
    std::free(number);
}
```

The proposal specifically nixes out including union accesses as implicit object creation, stating that preserving undefined behavior in that situation results in more explicit code, given that we have safer, more explicit operations to do so.


# Correct ways to type pun
## std::memcpy
The (only) correct way to deal with this prior to C++20 is to use std::memcpy to copy the bytes of the object to an existing object of the desired type, i.e:

```c++
#include <cstdint>
#include <cstdlib>
#include <cstddef>
#include <cstring>
#include <cassert>

struct NetworkHeader {
    uint16_t id;
    uint16_t flags;
};

NetworkHeader memcpy_example(const std::byte *raw_data) noexcept {
    NetworkHeader header;
    std::memcpy(&header, raw_data, sizeof(NetworkHeader));

    return header;
}

int main() {
    constexpr std::byte raw_data[4] = {
	std::byte{0x05}, std::byte{0x00},
	std::byte{0x11}, std::byte{0x00}
    };

    static_assert(sizeof(raw_data) == sizeof(NetworkHeader), "Buffer must be same size as header");
    const auto header = memcpy_example(raw_data);

    // NOTE: This assumes little-endian architecture.
    assert((header.id == 0x0005) && (header.flags == 0x0011));
}
```

Of course, you may want to avoid an actual copy. However, purely writing memcpy doesn't mean an actual memcpy call will occur - compiler authors are well aware of this pattern and most compilers will optimize it out. In fact, the following code compiled with `-O0` (no optimizations enabled) compiles to this [(see compiler explorer)](https://compiler-explorer.com/z/qzKj7x8qe):


```asm
memcpy_example(std::byte const*):
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-24], rdi
        mov     rax, QWORD PTR [rbp-24]
        mov     eax, DWORD PTR [rax]
        mov     DWORD PTR [rbp-4], eax
        mov     eax, DWORD PTR [rbp-4]
        pop     rbp
        ret
```
No calls to memcpy are invoked, and memory is simply just moved between the buffer and the integer. `-O3` will further optimize the function body to a single move instruction.

This accounts for strict aliasing and memory alignment, since a distinct memory location is used. Note that as mentioned before, `std::memcpy` is one of the functions that implicitly created objects, making this well-defined. 

Note that per the comment, `std::memcpy` correctly copies these bytes, but the interpretation of those bytes into a `uint16_t` is dependent on the [endianness](https://en.wikipedia.org/wiki/Endianness) of the platform.

## std::bit_cast
Using `std::memcpy` is safe, but leaves room for error, since the onus is on the user to check that the types have the same size. 

`std::bit_cast` is the more explicit way to reinterpret a set of bits as a value, handling size checking and enforcing that the types are trivially copyable, and that the destination type is trivially constructable:

```c++
#include <bit>
#include <cstdint>
uint32_t bitcast_example() noexcept {
    constexpr unsigned char buf[4] = {0, 0, 0, 1};
    const auto num = std::bit_cast<uint32_t>(buf);
    return num;
}

```
`std::bit_cast` is a `constexpr` function, so you are also able to use it in compile-time contexts, unlike `std::memcpy`.
## std::start_lifetime_as, or explicit lifetime management
While both `std::memcpy` and `std::bit_cast` work well, they are not ideal for every situation. Both require new destination objects to copy the source object representation into (even if this is typically optimized away), and std::bit_cast in particular operates on values, not just raw storage (i.e a function accepting an `unsigned char*`). `std::bit_cast` additionally enforces that both types are trivially copyable. 

Sometimes we may want to reuse existing storage and explicitly tell the compiler to interpret it as an object we desire, similar to the expected behavior of using a type cast. The same proposal that introduced implicit object creation spun off [another](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2590r0.pdf), introducing the functions `std::start_lifetime_as` and `std::start_lifetime_as_array`, adding true explicit lifetime management to C++. This has been added into the language in C++23.


```c++
#include <memory>
#include <cstdint>
uint32_t explicit_lifetime_example() noexcept {
    // Alignment needs to be set here, since we are using buf as the underlying storage
    alignas(uint32_t) const unsigned char buf[4] = {0, 0, 0, 1};
    
    // using pointer here to show
    // we could use start_lifetime_as with a pointer to raw storage
    // (i.e without size information of a stack array)
    const auto* buf_ptr = buf;

    // Start the lifetime of a uint32_t using buf as underlying storage
    const auto* num_ptr = std::start_lifetime_as<uint32_t>(buf_ptr);
    return *num_ptr;
}
```

`std::start_lifetime_as` gives the power of lifetime management to the user when necessary, not just held implicitly by special library functions like `std::memcpy`. This is the best replacement for the use of `reinterpret_cast` in the context of type punning that fully ensures defined behavior. Unfortunately, as of today, no compiler has implemented this feature yet.

# Conclusion
Type punning is a common practice in a lot of systems and embedded programming scenarios. Until semi-recently, C++ has made this surprisingly difficult to do safely, and the kinks of dealing with the bytes of an object are still being worked out today. Practices commonly adopted in industry codebases to do this, like `reinterpret_cast` or union usage, invoke undefined behavior and can often silently break programs when compiler optimizations are enabled, or different platforms are targeted.

Objects can't be arbitrarily cast back and forth between different types in C++. The strict aliasing rules prevent the compiler from assuming that pointers of different types  can point to the same memory location. Additionally, C++'s object model requires lifetime management, and pointer casts must respect alignment requirements of the target type. Modern C++ has greatly improved type punning solutions with the introduction of `std::bit_cast` for value-to-value conversion, and `std::start_lifetime_as` for in-place reinterpretation.


## Resources
* [Type punning in modern C++ - Timur Doumler - CppCon 2019](https://www.youtube.com/watch?v=_qzMpk-22cc) - This is by far the quintessential resource on type punning in C++ and was strongly relied on for this article. Timur Doumler covers each challenge involved with type punning, techniques for type punning without invoking undefined behavior, along with discussing holes in the language around type punning (specifically addressing accessing the bytes of an object.)
* [Taking a Byte Out of C++ - Avoiding Punning by Starting Lifetimes - Robert Leahy - CppCon 2022](https://www.youtube.com/watch?v=pbkQG09grFw) - A great talk on lifetimes, with implicit object creation and explicit lifetime management using `start_lifetime_as`. I really like his analogy of trying to use the type system "as a lens", which is a trap a lot of people fall into. He covers practical usage of `start_lifetime_as` in real-world low-level code, along with additional concepts I didn't explore here, like the use of `std::launder` and `std::start_lifetime_as_array`.
