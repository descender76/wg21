---
breaks: false
---

<style type="text/css">
ins { background-color: #A0FFA0 }
s { background-color: #FFA0A0 }
blockquote { color: inherit !important }
</style>

<table><tbody>
<tr><th>Doc. no.:</th>    <td>D0792R6</td></tr>
<tr><th>Date:</th>        <td>2021-12-31</td></tr>
<tr><th>Audience:</th>    <td>LEWG, LWG</td></tr>
<tr><th>Reply-to:</th>    <td>Vittorio Romeo &lt;vittorio.romeo@outlook.com&gt;</td></tr>
<tr><th></th>    <td>Jarrad Waterloo &lt;descender76@gmail.com&gt;</td></tr>
<tr><th></th>    <td>Zhihao Yuan &lt;zy@miator.net&gt;</td></tr>
</tbody></table>


# `function_ref`: a type-erased call proxy

## Table of contents

[TOC]

## Changelog

### R6

- TODO

### R5

- Removed "qualifiers" from `operator()` specification (typo);

### R4

- Removed `constexpr` due to implementation concerns;
- Explicitly say that the type is trivially copyable;
- Added brief before synopsis;
- Reworded specification following P1369.

### R3

- Constructing or assigning from `std::function` no longer has a precondition;
- `function_ref::operator()` is now unconditionally `const`-qualified.

### R2

- Made copy constructor and assignment operator `= default`;
- Added _exposition only_ data members.

### R1

- Removed empty state, comparisons with `nullptr`, and default constructor;
- Added support for `noexcept` and `const`-qualified function signatures;
- Added deduction guides similar to `std::function`;
- Added example implementation;
- Added feature test macro;
- Removed `noexcept` from constructor and assignment operator.


## Abstract

This paper proposes the addition of `function_ref<R(Args...)>`, a _vocabulary type_ with reference semantics for passing entities to call, to the standard library.


## Design considerations

This paper went through LEWG at R5, with a number of consensus reached and applied to the wording:

1. do not provide `target()` or `target_type`
2. no `operator bool`, default constructor, or comparison with `nullptr`
3. `R(Args...) noexcept` specializations
4. `R(Args...) const` specializations
5. require the target entity to be _Lvalue-Callable_
6. `operator()` is unconditionally `const`
7. `function_ref` is the right name

One design question remains not fully understood by many: how should a function, a function pointer, or a pointer-to-member initialize `function_ref`? The sixth revision of the paper aims to provide full background and dive deep into that question.

```cpp
auto retry(std::size_t times,
           function_ref<std::optional<payload>()> action)
{
    /* ... */
}
```

## Discussion


## Changes to `<functional>` header

Add the following to `[functional.syn]`:

> ```cpp
> namespace std
> {
>     // ...
>
>     template <typename Signature> class function_ref;
>
>     template <typename Signature>
>     void swap(function_ref<Signature>& lhs, function_ref<Signature>& rhs) noexcept;
>
>     // ...
> }
> ```



## Class synopsis

Create a new section "Class template `function_ref`", `[functionref]`" with the following:

> ```cpp
> namespace std
> {
>     template <typename Signature>
>     class function_ref
>     {
>         void* erased_object; // exposition only
>
>         R(*erased_function)(Args...); // exposition only
>         // `R`, and `Args...` are the return type, and the parameter-type-list,
>         // of the function type `Signature`, respectively.
>
>     public:
>         function_ref(const function_ref&) noexcept = default;
>
>         template <typename F>
>         function_ref(F&&);
>
>         function_ref& operator=(const function_ref&) noexcept = default;
>
>         template <typename F>
>         function_ref& operator=(F&&);
>
>         void swap(function_ref&) noexcept;
>
>         R operator()(Args...) const noexcept(see below);
>         // `R` and `Args...` are the return type and the parameter-type-list
>         // of the function type `Signature`, respectively.
>     };
>
>     template <typename Signature>
>     void swap(function_ref<Signature>&, function_ref<Signature>&) noexcept;
>
>     template <typename R, typename... Args>
>     function_ref(R (*)(Args...)) -> function_ref<R(Args...)>;
>
>     template <typename R, typename... Args>
>     function_ref(R (*)(Args...) noexcept) -> function_ref<R(Args...) noexcept>;
>
>     template <typename F>
>     function_ref(F) -> function_ref<see below>;
> }
> ```
>
> 1. `function_ref<Signature>` is a `Cpp17CopyConstructible` and `Cpp17CopyAssignable` reference to an `Invocable` object with signature `Signature`.
>
> 2. `function_ref<Signature>` is a trivially copyable type.
>
> 3. The template argument `Signature` shall be a non-`volatile`-qualified function type.







## Specification

```cpp
template <typename F>
function_ref(F&& f);
```

* *Constraints:* `is_same_v<remove_cvref_t<F>, function_ref>` is `false` and

    * If `Signature` has a `noexcept` specifier: `is_nothrow_invocable_r_v<R, cv-qualifiers F&, Args...>` is `true`;

    * Otherwise: `is_invocable_r_v<R, cv-qualifiers F&, Args...>` is `true`.

    Where `R`, `Args...`, and `cv-qualifiers` are the *return type*, the *parameter-type-list*, and the sequence "*cv-qualifier-seq-opt*" of the function type `Signature`, respectively.

* *Expects:* `f` is neither a null function pointer value nor a null member pointer value.

* *Effects:* Constructs a `function_ref` referring to `f`.

* *Remarks:* `erased_object` will point to `f`. `erased_function` will point to a function whose invocation is equivalent to `return INVOKE<R>(f, std::forward<Args>(xs)...);`, where `f` is qualified with the same *cv-qualifiers* as the function type `Signature`.

<br>



```cpp
template <typename F>
function_ref& operator=(F&&);
```

* *Constraints:* `is_same_v<remove_cvref_t<F>, function_ref>` is `false` and

    * If `Signature` has a `noexcept` specifier: `is_nothrow_invocable_r_v<R, cv-qualifiers F&, Args...>` is `true`;

    * Otherwise: `is_invocable_r_v<R, cv-qualifiers F&, Args...>` is `true`.

    Where `R`, `Args...`, and `cv-qualifiers` are the *return type*, the *parameter-type-list*, and the sequence "*cv-qualifier-seq-opt*" of the function type `Signature`, respectively.

* *Expects:* `f` is neither a null function pointer value nor a null member pointer value.

* *Ensures:* `*this` refers to `f`.

* *Returns:* `*this`.

<br>



```cpp
void swap(function_ref& rhs) noexcept;
```

* *Effects:* Exchanges the values of `*this` and `rhs`.

<br>



```cpp
R operator()(Args... xs) noexcept(see below);
```

* *Effects:* Equivalent to `return INVOKE<R>(f, std::forward<Args>(xs)...);`, where `f` is the callable object referred to by `*this`, qualified with the same *cv-qualifiers* as the function type `Signature`.

* *Remarks:* `R` and `Args...` are the return type and the parameter-type-list of the function type `Signature`, respectively. The expression inside `noexcept` is the sequence "noexcept-specifier-opt" of the function type `Signature`.

<br>



```cpp
template <typename F>
function_ref(F) -> function_ref<see below>;
```

* *Constraints:* `&F::operator()` is well-formed when treated as an unevaluated operand.

* *Remarks:* If `decltype(&F::operator())` is of the form `R(G::*)(A...) qualifiers` for a class type `G`, then the deduced type is `function_ref<R(A...) qualifiers>`.

<br>



```cpp
template <typename Signature>
void swap(function_ref<Signature>& lhs, function_ref<Signature>& rhs) noexcept;
```

* *Effects:* Equivalent to `lhs.swap(rhs)`.

<br>



## Feature test macro

Append to §17.3.1 General `[support.limits.general]`'s Table 36 one additional entry:

| Macro name               | Value     | Headers        |
| -------------------------|-----------|----------------|
| `__cpp_lib_function_ref` | `201811L` | `<functional>` |

## Specification Take 2

> The following is relative to N4901.[5]
> 
> Insert the following in Header synopsis [version.syn], in section 2, below #define __cpp_lib_memory_resource 201603L
> ```cpp
> #define __cpp_lib_function_ref 20XXXXL // also in <functional>
> ```
> Let SECTION is a placeholder for the root of the section numbering for [functional].
> 
> Insert the following section in Header `<functional>` synopsis [functional.syn], at the end of SECTION.20, polymorphic function wrappers
> ```cpp
> template<class... S> class function_ref; // not defined
> 
> // Set of partial specializations of function_ref
> template<class R, class... ArgTypes>
>   class function_ref<R(ArgTypes...) cv noexcept(noex)>; // ... see below
> ```
> Insert the following section at the end of Polymorphic function wrapper [func.wrap] at the end.
> 
> ###	SECTION.20.3 function_ref                                 [func.wrap.ref]
> 
>  1  The header provides partial specializations of function_ref for
> 	each combination of the possible replacements of the placeholders *cv* and
> 	*noex* where:
> 
> (1.1)   — *cv* is either const or empty.
> 
> (1.2)   — *noex* is either true or false.
> 
>  2  For each of the possible combinations of the placeholders mentioned above,
> 	there is a placeholder inv-quals defined as follows:
> 
> (2.1)   — Let *inv-quals* be *cv*&
> 
> ###	SECTION.20.3.1 Class template function_ref            [func.wrap.ref.class]
> ```cpp
> namespace std {
> template<class... S> class function_ref; // not defined
> 
> template<class R, class... ArgTypes>
> class function_ref<R(ArgTypes...) cv noexcept(noex)> {
> public:
>   using result_type = R;
> 
>   // SECTION.20.3.2, construct/move/destroy
>   function_ref() noexcept;
>   function_ref(function_ref&&) noexcept;
>   template<class F> function_ref(F&&);
>
>   function_ref& operator=(function_ref&&);
>   template<class F> function_ref& operator=(F&&);
> 
>   ~function_ref();
> 
>   // SECTION.20.3.3, function_ref invocation
>   R operator()(ArgTypes...) cv noexcept(noex);
> 
>   // SECTION.20.3.4, function_ref utility
>   void swap(function_ref&) noexcept;
> 
>   friend void swap(function_ref&, function_ref&) noexcept;
> 
> private:
>   template<class VT>
> 	static constexpr bool is-callable-from = see below; // exposition-only
> };
> }
> ```
> 1  The function_ref class template provides polymorphic wrappers that generalize
> the notion of a callable object [func.def]. These wrappers can reference and
> call arbitrary callable objects, given a call signature, allowing functions to
> be first-class objects.
> 
> 2  Implementations are encouraged to avoid the use of dynamically allocated memory under any circumstance.
> 
> ### SECTION.20.3.3  Constructors and destructor                              [func.wrap.ref.con]
> ```cpp
> template<class VT>
> 	  static constexpr bool is-callable-from = see below; // exposition-only
> ```
> 1  If noex is `true`, `is-callable-from<VT>` is equal to
>   `is_nothrow_invocable_r_v<R, VT cv ref, Args...> &&`
> 	`is_nothrow_invocable_r_v<R, VT inv-quals, Args...>`.
> Otherwise, `is-callable-from<VT>` is equal to
>   `is_invocable_r_v<R, VT cv ref, Args...> &&`
> 	`is_invocable_r_v<R, VT inv-quals, Args...>`.
> ```cpp
> function_ref(function_ref& f) noexcept;
> ```
> 2        *Postconditions:* This constructor trivially copies the function_ref.
> ```cpp
> template<class F> function_ref(F&& f);
> ```
> 3        Let VT be `decay_t<F>`.
> 
> 4        *Constraints:*
> 
> (4.1)            — `remove_cvref_t<F>` is not the same type as function_ref, and
> 
> (4.2)            — `is-callable-from<VT>` is `true`.
> 
> (4.3)            — `F` is `Callable`.
> 
> 5        *Mandates:* `is_constructible_v<VT, F>` is `true`
> 
> 6        *Preconditions:* VT meets the Cpp17Destructible requirements, and if
> 	  `is_move_constructible_v<VT>` is `true`, VT meets the Cpp17MoveConstructible
> 	  requirements.
> 
> 7        *Postconditions:* `*this` has a target object unless any of the following hold:
> 
> (7.1)            — f is a null function pointer value, or
> 
> (7.2)            — f is a null member pointer value, or
> 
> (7.3)            — `remove_cvref_t<F>` is a specialization of the function_ref class template,
> 			 and f has no target object.
> 
> 8        Otherwise, `*this` targets an object of type VT
> 	  direct-non-list-initialized with `std::forward<F>(f)`.
> 
> 9       *Throws:* Any exception thrown by the initialization of the target object.
> 	  May throw bad_alloc unless VT is a function pointer or a specialization of
> 	  reference_wrapper.
> ```cpp
> function_ref& operator=(function_ref&& f);
> ```
> 10        *Effects:* This assignment operator is trivial.
> 
> 11        *Returns:* `*this`.
> ```cpp
> ~function_ref();
> ```
> 12        *Effects:* The destructor is trivial.
> 
> ### SECTION.20.3.4  Invocation                                              [func.wrap.ref.inv]
> ```cpp
> R operator()(ArgTypes... args) cv noexcept(noex);
> ```
> 1        *Preconditions:* `*this` has a target object.
> 
> 2        *Effects:* Equivalent to:
> 	  `return INVOKE<R>(static_cast<F inv-quals>(f), std::forward<ArgTypes>(args)...);`
> 	  where f is the target object of `*this` and f is lvalue of type F.
> 
> ### SECTION.20.3.5  Utility                                                 [func.wrap.ref.util]
> ```cpp
> void swap(function_ref& other) noexcept;
> ```
> 1        *Effects:* Exchanges the targets of `*this` and other.
> ```cpp
> friend void swap(function_ref& f1, function_ref& f2) noexcept;
> ```
> 2        *Effects:* Equivalent to: `f1.swap(f2)`.

## Example implementation

The most up-to-date implementation, created by Simon Brand, is available on [GitHub/TartanLlama/function_ref](https://github.com/TartanLlama/function_ref).

An older example implementation is available here on [GitHub/SuperV1234/Experiments](https://github.com/SuperV1234/Experiments/blob/master/function_ref.cpp).



## Existing practice

Many facilities similar to `function_ref` exist and are widely used in large codebases. Here are some examples:

* The `llvm::function_ref` [^llvmfunctionref] class template is used throughout LLVM. A quick GitHub search on the LLVM organization reports hundreds of usages both in `llvm` and `clang` [^githubsearch0].

* Facebook's Folly libraries [^folly] provide a `folly::FunctionRef` [^follyfunctionref] class template. A GitHub search shows that it's used in projects `proxygen` and `fbthrift` [^follyusages].

* GNU's popular debugger, `gdb` [^gdb], uses `gdb::function_view` [^gdbfnview] throughout its code base. The documentation in the linked header file [^gdbfnview] is particularly well-written and greatly motivates the need for this facility.

Additionally, combining results from GitHub searches *(excluding "`llvm`" and "`folly`")* for "`function_ref`" [^githubsearch1], "`function_view`" [^githubsearch2], "`FunctionRef`" [^githubsearch3], and "`FunctionView`" [^githubsearch4] roughly shows more than 2800 occurrences.



## Possible issues

Accepting temporaries in `function_ref`'s constructor is extremely useful in the most common use case: using it as a function parameter. E.g.

```cpp
void foo(function_ref<void()>);

int main()
{
    foo([]{ });
}
```

<div class="inline-link">

[*(on wandbox.org)*](https://wandbox.org/permlink/BPtbPeQtErPGj4X7)

</div>

The usage shown above is completely safe: the temporary closure generated by the lambda expression is guarantee to live for the entirety of the call to `foo`. Unfortunately, this also means that the following code snippet will result in *undefined behavior*:

```cpp
int main()
{
    function_ref<void()> f{[]{ }};
    // ...
    f(); // undefined behavior
}
```

<div class="inline-link">

[*(on wandbox.org)*](https://wandbox.org/permlink/cQPEX2sKjCQjgIki)

</div>

The above closure is a temporary whose lifetime ends after the `function_ref` constructor call. The `function_ref` will store an address to a "dead" closure - invoking it will produce undefined behavior [^jmfunctionview]. As an example, `AddressSanitizer` detects an invalid memory access in this gist [^gistub]. Note that this problem is not unique to `function_ref`: the recently standardized `std::string_view` [^stringview] has the same problem [^jmstringview].

I strongly believe that accepting temporaries is a "necessary evil" for both `function_ref` and `std::string_view`, as it enables countless valid use cases. The problem of dangling references has been always present in the language - a more general solution like Herb Sutter and Neil Macintosh's lifetime tracking [^lifetimes] would prevent mistakes without limiting the usefulness of view/reference classes.


## Acknowledgments

Thanks to **Agustín Bergé**, **Dietmar Kühl**, **Eric Niebler**, **Tim van Deurzen**, and **Alisdair Meredith** for providing very valuable feedback on earlier drafts of this proposal.







## Annex: previously open questions

* Why does `operator()` take `Args...` and not `Args&&...`?

    * While taking `Args&&...` would minimize the amount of copies/moves, it would be a pessimization for small value types. Also, taking `Args...` is consistent with how `std::function` works.

* `function_ref<Signature>`'s signature currently only accepts any combination of `const` and `noexcept`. Should this be extended to include *ref-qualifiers*? This would mean that `function_ref::operator()` would first cast the referenced callable to either an *lvalue reference* or *rvalue reference* (depending on `Signature`'s ref qualifiers) before invoking it. See P0045R1 [^p0045r1] and N4159 [^n4159]) for additional context.

    * LEWG agreed that `const` and `noexcept` have useful cases, but we could not find enough motivation to include support for *ref-qualified* signatures. Nevertheless, this could be added as a non-breaking extension to `function_ref` in the future.

* Constructing a `std::function<Signature>` from a `function_ref<Signature>` is completely different from constructing a `std::string` from a `std::string_view`: the latter does actually create a copy while the former remains a reference. It may be reasonable to prevent implicit conversions from `function_ref` to `std::function` in order to avoid surprising dangerous behavior.

    * LEWG decided to not prevent `std::function` construction from `std::function_ref` as it would special-case `std::function` and there are other utilities in the Standard Library (and outside of it) that would need a similar change (e.g. `std::bind`).

* `function_ref::operator()` is not currently marked as `constexpr` due to implementation issues. I could not figure a way to implement a `constexpr`-friendly `operator()`. Is there any possibility it could be marked as `constexpr` to increase the usefulness of `function_ref`?

    * We agreed that there is probably no way of currently having a `constexpr` `function_ref::operator()` and that we do not want to impose that burden on implementations.

* Should the `!f` precondition when constructing `function_ref` from an instance `f` of `std::function` be removed? The behavior in that case is well-defined, as `f` is guarateed to throw on invocation.

    * LEWG decided to remove the precondition as invoking a default-constructed instance of `std::function` is well-defined.

* The `std::is_nothrow_invocable` constraint in `function_ref` construction/assignment for `noexcept` signatures prevents users from providing a non-`noexcept` function, even if they know that it cannot ever throw (e.g. C functions). Should this constraint be removed? Should an `explicit` constructor without the constraint be provided?

    * LEWG agreed that the constraint should be kept and no extra constructors should be added as users can use a `noexcept` lambda to achieve the same result.

* Propagating `const` to `function_ref::operator()` doesn't make sense when looking at `function_ref` as a simple "reference" class. `const` instances of `function_ref` should be able to invoke a `mutable` lambda, as the state of `function_ref` itself doesn't change. E.g.

    ```cpp
    auto l0 = []() mutable { };
    const function_ref<void()> fr{l0};

    fr(); // Currently a compilation error
    ```

    An alternative is to only propagate `noexcept` from the signature to `function_ref::operator()`, and unconditionally `const`-qualify `function_ref::operator()`. Do we want this?

    * LEWG agreed to mark `function_ref::operator()` `const`, unconditionally.

* We want to avoid double indirection when a `function_ref` instance is initialized with a `reference_wrapper`. `function_ref` could just copy the pointer stored inside the `reference_wrapper` instead of pointing to the wrapper itself. This cannot be covered by the *as-if* rule as it changes program semantics. E.g.

    ```cpp
    auto l0 = []{ };
    auto l1 = []{ };
    auto rw = std::ref(l0);

    function_ref<void()> fr{rw};
    fr(); // Invokes `l0`

    rw = l1;
    fr(); // What is invoked?
    ```

    Is adding wording to handle `std::reference_wrapper` as a special case desirable?

    * LEWG decided that special-casing `std::reference_wrapper` is undesirable.

* Is it possible and desirable to remove `function_ref`'s template assignment operator from `F&&` and rely on an implicit conversion to `function_ref` + the default copy assignment operator?

    * LEWG deferred this question to LWG.

* Should `function_ref` only store a `void*` pointer for the callable object, or a `union`? In the first case, seemingly innocent usages will result in undefined behavior:

    ```cpp
    void foo();
    function_ref<void()> f{&foo};
    f(); // Undefined behavior
    ```

    ```cpp
    struct foo { void bar(); }
    function_ref<void(foo)> f{&foo::bar};
    f(foo{}); // Undefined behavior
    ```

    If a `union` is stored instead, the first usage could be well-formed without any extra overhead (assuming `sizeof(void*) == sizeof(void(*)())`). The second usage could also be made well-formed, but with size overhead as `sizeof(void(C::*)()) > sizeof(void*)`.

    Regardless, the exposition-only members should clearly illustrate the outcome of this decision.

    Note that if we want the following to compile and be well-defined, a `void(*)()` would have to be stored inside `function_ref`:

    ```cpp
    void foo();
    function_ref<void()> f{foo};
    f();
    ```

    * LEWG agreed that `function_ref` should fully support the `Callable` concept.

* Should the `function_ref(F&&)` deduction guide take its argument by value instead? This could simplify the wording.

    * LEWG deferred this question to LWG.


## References

[^passingfunctionstofunctions]: <https://vittorioromeo.info/index/blog/passing_functions_to_functions.html#benchmark---generated-assembly>
[^llvmfunctionref]: <http://llvm.org/doxygen/classllvm_1_1function__ref_3_01Ret_07Params_8_8_8_08_4.html>
[^githubsearch0]: <https://github.com/search?q=org%3Allvm-mirror+function_ref&type=Code>
[^folly]: <https://github.com/facebook/folly>
[^follyfunctionref]: <https://github.com/facebook/folly/blob/master/folly/Function.h#L743-L824>
[^follyusages]: <https://github.com/search?q=org%3Afacebook+FunctionRef&type=Code>
[^githubsearch1]: <https://github.com/search?utf8=%E2%9C%93&q=function_ref+AND+NOT+llvm+AND+NOT+folly+language%3AC%2B%2B&type=Code>
[^githubsearch2]: <https://github.com/search?utf8=%E2%9C%93&q=function_view+AND+NOT+llvm+AND+NOT+folly+language%3AC%2B%2B&type=Code>
[^githubsearch3]: <https://github.com/search?utf8=%E2%9C%93&q=functionref+AND+NOT+llvm+AND+NOT+folly+language%3AC%2B%2B&type=Code>
[^githubsearch4]: <https://github.com/search?utf8=%E2%9C%93&q=functionview+AND+NOT+llvm+AND+NOT+folly+language%3AC%2B%2B&type=Code>
[^gistub]: <https://gist.github.com/SuperV1234/a41eb1c825bfbb43f595b13bd4ea99c3>
[^stringview]: <http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3762.html>
[^jmfunctionview]: <http://foonathan.net/blog/2017/01/20/function-ref-implementation.html>
[^jmstringview]: <http://foonathan.net/blog/2017/03/22/string_view-temporary.html>
[^lifetimes]: <https://github.com/isocpp/CppCoreGuidelines/blob/master/docs/Lifetimes%20I%20and%20II%20-%20v0.9.1.pdf>
[^gdb]: <https://www.gnu.org/software/gdb/>
[^p0045r1]: <http://wg21.link/p0045r1>
[^n4159]: <http://wg21.link/N4159>
[^gdbfnview]: <https://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;a=blob;f=gdb/common/function-view.h>
