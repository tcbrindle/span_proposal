
# Usability Enhancements for `std::span` #

Document Number: DXXXXR0

Reply-to: Tristan Brindle <tcbrindle@gmail.com>

Date: 2018-04-16

Audience: Library Evolution Working Group

## 1. Introduction ##

The class template `span<ElementType, Extent>` was recently added to the working draft of the C++ International Standard [N4741]. A `span` is a lightweight object providing a "view" of an underlying contiguous array, which does not own the elements it points to. It is intended as a new "vocabulary type" for contiguous ranges, replacing the use of `(pointer, length)` pairs and, in some cases, `vector<T, A>&`  function parameters.

This paper identifies several opportunities to enhance the usability of `span` by improving consistency with existing container interfaces and removing potential points of confusion for users.

A reference implementation of `span` including the changes detailed in this paper is available at XXXXX.

### 1.1 Terminology ###

For the purposes of this paper, a *fixed-size span* is a `span` whose `Extent` is greater than or equal to zero. A *dynamically-sized span* is a `span` whose `Extent` is equal to `std::dynamic_extent`.

## 2. Proposals ##

### 2.1 Add `front()` and `back()` member functions ###

To improve consistency with standard library containers, we propose adding `front()` and `back()` member functions with their usual meanings (that is, returning references to the first and last elements respectively). The effect of calling these functions on an empty `span` is undefined.

### 2.2 Add `at()` member function ###

The `span` proposal paper[P0122R7] makes it clear that implementations are encouraged to add bounds-checking to `span`'s interfaces. However, this is not required by the wording, and it seems highly likely that for effiency reasons such checks (if present) will be disabled in optimised builds.

In common with other containers then, we propose to add an `at()` member function with guaranteed bounds checking on all implementations even in optimised builds, throwing `std::out_of_range` if the supplied argument is invalid.

### 2.3 Non-member subspan operations ###

The current wording for `span` defines the subspan operations (namely `first()`, `last()` and `subspan()`) as member functions. We note that these are specified only to call public `span` constructors, and could therefore be made non-members instead, following the well-known principle of "preferring non-member, non-friends".

This in itself would be a minor quibble, but unfortunately making these functions members introduces a major usability issue. As member templates of a class template, the fixed-size overloads of these functions require the use of the highly unusual "dot template" syntax in cases where the `span` itself is a dependent type. For example:

```cpp
template <typename T>
std::span<T, 3> first_three(std::span<T> s)
{
    return s.template front<3>(); // Urgh!
}
```

Without the keyword `template` here, compilers are required to parse the return expression as an attempted use of less-than, leading to extremely unhelpful error messages in current implementations.

As a vocabulary type, `span` will be used by programmers with varying levels of C++ knowledge. It is our experience that the "dot template" syntax is unfamiliar to all but experts. We hesitate to think of the questions that would be generated on Stack Overflow and elsewhere if this syntax were required. For this reason, we propose making the `first()`, `last()` and `subspan()` operations non-members, which avoids the need for the `template` keyword.

(We note that `std::get<N>()` for `tuple` and `variant`, which conceptually behaves like a member of those classes, is defined as a non-member, likely for exactly this reason.)

### 2.4 Remove `operator()` ###

The current wording for `span` includes an overload of the function call operator, duplicating the behaviour of `operator[]`. We assume that this is a holdover from `span`'s genesis as a multidimentional `array_view`.

Providing this operator for member access is inconsistent with other container types and with built-in language arrays. Furthermore, it provides the mistaken impression that it is possible to "invoke" a `span`. We therefore propose its removal.

### 2.5 Structured bindings support for fixed-size spans ###

Built-in arrays and `std::array`s can be used with structured bindings, via core language and library support respectively. To allow function arguments of type `T (&)[N]` to be replaced with the more appealing `span<T, N>` with equal functionality, we propose adding support for structured bindings for fixed-size spans. Specifically, we propose a new overload of `std::get<N>()`, and specialisations of `tuple_element` and `tuple_size` for `span`.

Dynamically-sized spans cannot be decomposed. To prevent obtuse error messages should this be attempted, this proposal adds an empty partial specialisation

```cpp
template <class ElementType>
struct tuple_size<span<ElementType, dynamic_extent>> {};
```

which does not provide a `value` member.

## 3 Proposed wording ##

TBD

## References ##

[N4741] Richard Smith, *"Working Draft, Standard for Programming
Language C++"*
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/n4741.pdf

[P0122R7] Neil Macintosh and Stephan T. Lavavej, *"span: bounds-safe views for sequences of objects"*
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0122r7.pdf