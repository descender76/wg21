---
breaks: false
---

<style type="text/css">
ins { background-color: #CCFFCC }
s { background-color: #FFCACA }
blockquote { color: inherit !important }
</style>

<table><tbody>
<tr><th>Doc. no.:</th>    <td>D0792R10</td></tr>
<tr><th>Date:</th>        <td>2022-06-13</td></tr>
<tr><th>Audience:</th>    <td>LWG</td></tr>
<tr><th>Reply-to:</th>    <td>Vittorio Romeo &lt;vittorio.romeo@outlook.com&gt;<br>
Zhihao Yuan &lt;zy@miator.net&gt;<br>
Jarrad Waterloo &lt;descender76@gmail.com&gt;</tr>
</tbody></table>


# `function_ref`: a type-erased callable reference

## Table of contents

[TOC]

## Changelog

#### R10

- Integrate the [`nontype_t`](#Additional-information) constructors.

#### R9

- Declare the main template as variadic for future extension;
- Allow declaring a variable of `function_ref` `constinit`.

#### R8

- Stop supporting pointer-to-members;
- Prevent assigning from callable objects other than function pointers while retaining copy assignment;
- Consolidate the wording with better terminologies.

#### R7

- Clarify the proposal to handle function and function pointers in the same way.

#### R6

- Avoid double-wrapping existing references to callables;
- Reworked the wording to follow the latest standardese;
- Applied changes requested by LWG (2020-07);
- Removed a deduction guide that is incompatible with explicit object parameters.

#### R5

- Removed "qualifiers" from `operator()` specification (typo);

#### R4

- Removed `constexpr` due to implementation concerns;
- Explicitly say that the type is trivially copyable;
- Added brief before synopsis;
- Reworded specification following P1369.

#### R3

- Constructing or assigning from `std::function` no longer has a precondition;
- `function_ref::operator()` is now unconditionally `const`-qualified.

#### R2

- Made copy constructor and assignment operator `= default`;
- Added _exposition only_ data members.

#### R1

- Removed empty state, comparisons with `nullptr`, and default constructor;
- Added support for `noexcept` and `const`-qualified function signatures;
- Added deduction guides similar to `std::function`;
- Added example implementation;
- Added feature test macro;
- Removed `noexcept` from constructor and assignment operator.


## Abstract

This paper proposes the addition of `function_ref<R(Args...)>`, a _vocabulary type_ with reference semantics for passing entities to call, to the standard library.


## Motivating examples

Here's one example use case that benefits from *higher-order functions*: a `retry(n, f)` function that attempts to call `f` up to `n` times synchronously until success. This example might model the real-world scenario of repeatedly querying a flaky web service.

```cpp
using payload = std::optional< /* ... */ >;

// Repeatedly invokes `action` up to `times` repetitions.
// Immediately returns if `action` returns a valid `payload`.
// Returns `std::nullopt` otherwise.
payload retry(size_t times, /* ????? */ action);
```

The passed-in `action` should be a callable entity that takes no arguments and returns a `payload`. Let's see how to implemented `retry` with various techniques.


### Using function pointers

```cpp
payload retry(size_t times, payload(*action)())
{
    /* ... */
}
```

- **Advantages**:

  - Easy to implement: no template, nor constraint. The function pointer type has a signature that nails down which functions to pass.

  - Minimal overhead: no allocations, no exceptions, and major calling conventions can pass a function pointer in a register.

- **Drawbacks**:

  - A function is usually not stateful, nor does captureless closure objects. One cannot pass other function objects in C++ this way.

### Using a template

```cpp
template<class F>
auto retry(size_t times, F&& action)
requires std::is_invocable_r_v<payload, F>
{
    /* ... */
}
```

- **Advantages**:

  - Support arbitrary function objects, such as closures with captures.

  - Zero-overhead: no allocations, no exceptions, no indirections.

- **Drawbacks**:

  - Harder to implement: users must constrain `action`'s signature.

  - Fail to support separable compilation: the implementation of `retry` must appear in a header file. A slight change at the call site will cause recompilation of the function body.

### Using `std::function` or `std::move_only_function`

```cpp
payload retry(size_t times, std::move_only_function<payload()> action)
{
    /* ... */
}
```

- **Advantages**:

  - Take more _callable objects_, from closures to pointer-to-members.

  - Easy to implement: no need to use a template or any explicit constraint. `std::function` and `std::move_only_function` constructor is constrained.

- **Drawbacks**:

  - `std::function` and `std::move_only_function`'s converting constructor[^p0288r9] require their target objects to be copy-constructible or move-constructible, respectively;

  - Comes with potentially significant overhead:

    - The call wrappers start to allocate the target objects when they do not fit in a small buffer, introducing more indirection when calling the objects.

    - No calling conventions can pass these call wrappers in registers.

    - Modern compilers cannot inline these call wrappers, often resulting in inferior codegen then previously mentioned techniques.

  One rarely known technique is to pass callable objects to call wrappers via a `std::reference_wrapper`:

  ```cpp
  auto result = retry(3, std::ref(downloader));
  ```

  But users cannot replace the `downloader` in the example with a lambda expression as such an expression is an rvalue. Meanwhile, all the machinery that implements type-erased copying or moving must still be present in the codegen.



### Using the proposed `function_ref`

```cpp
payload retry(size_t times, function_ref<payload()> action)
{
    /* ... */
}
```

- **Advantages**:

  - Takes any _callable objects_ regardless of whether they are constructible.

  - Easy to implement: no need to use a template or any constraint. `function_ref` is constrained.

  - Clean ownership semantics: `function_ref` has reference semantics as its name suggests.

  - Minimal overhead: no allocations, no exceptions, certain calling conventions can pass `function_ref` in registers.

    - Modern compilers can perform tail-call optimization when generating thunks. If the function body is visible, they can deliver optimal codegen, identical to the template solution.


## Design considerations

This paper went through LEWG at R5, with a number of consensuses reached and applied to the wording:

1. Do not provide `target()` or `target_type`;
2. Do not provide `operator bool`, default constructor, or comparison with `nullptr`;
3. Provide `R(Args...) noexcept` specializations;
4. Provide `R(Args...) const` specializations;
5. Require the target entity to be _Lvalue-Callable_;
6. Make `operator()` unconditionally `const`;
7. Choose `function_ref` as the right name.

One design question remains not fully understood by many: how should a function pointer initialize `function_ref`?

In a typical scenario, there is no lifetime issue no matter whether the `download` entity below is a function, a function pointer, or a closure:

```cpp
auto result = retry(3, download); // always safe
```

However, even if the users use `function_ref` only as parameters initially, it's not uncommon to evolve the API by grouping parameters into structures,

```cpp
struct retry_options
{
    size_t times;
    function_ref<payload()> action;
    seconds step_back;
};

payload retry(retry_options);

/* ... */

auto result = retry({.times = 3,
                     .action = download,
                     .step_back = 1.5s});

```

and structures start to need constructors or factories to simplify initialization:

```cpp
auto opt = default_strategy();
opt.action = download;
auto result = retry(opt);
```

According to the P0792R5[^p0792r5] wording, the code has well-defined behavior if `download` is a function. However, one **cannot** write the code as

```cpp
auto opt = default_strategy();
opt.action = &download;
auto result = retry(opt);
```

since this will create a `function_ref` with a dangling object pointer that points to a temporary object -- the function pointer.

In other words, the following code also has undefined behavior:

```cpp
auto opt = default_strategy();
opt.action = ssh.get_download_callback(); // a function pointer
auto result = retry(opt);
```

The users have to write the following to get well-defined behavior.

```cpp
auto opt = default_strategy();
opt.action = *ssh.get_download_callback();
auto result = retry(opt);
```

### Survey

We collected the following `function_ref` implementations available today:

<span style="color: rgb(9, 99, 125)">■</span>[`llvm::function_ref`](https://github.com/llvm/llvm-project/blob/main/llvm/include/llvm/ADT/STLExtras.h) -- from LLVM[^llvmfuncref]

<span style="color: rgb(132, 190, 106);">■</span>[`tl::function_ref`](https://github.com/TartanLlama/function_ref/blob/master/include/tl/function_ref.hpp) -- by Sy Brand

<span style="color: rgb(43, 57, 144);">■</span>[`folly::FunctionRef`](https://github.com/facebook/folly/blob/main/folly/Function.h) -- from Meta

<span style="color: rgb(255, 207, 171);">■</span>[`gdb::function_view`](https://github.com/bminor/binutils-gdb/blob/master/gdbsupport/function-view.h) -- from GNU

<span style="color: rgb(158, 16, 32);">■</span>[`type_safe::function_ref`](https://github.com/foonathan/type_safe/blob/main/include/type_safe/reference.hpp) -- by Jonathan Müller[^jmfunctionview]

<span style="color: rgb(227, 178, 0);">■</span>[`absl::function_ref`](https://github.com/abseil/abseil-cpp/blob/master/absl/functional/function_ref.h) -- from Google

They have diverging behaviors when initialized from function pointers:

<table>
<caption>Behavior A.1: Stores a function pointer if initialized from a function, stores a pointer to function pointer if initialized from a function pointer</caption>
<thead>
<tr><th>Outcome</th><th>Library</th></tr>
</thead>
<tbody>

<tr>
<td>

Undefined:

```cpp
opt.action = ssh.get_download_callback();
```

Well-defined:
```cpp
opt.action = download;
```

</td>
<td>

<span style="color: rgb(9, 99, 125)">■</span>`llvm::function_ref`

<span style="color: rgb(132, 190, 106);">■</span>`tl::function_ref`

</td></tr>
</tbody></table>

<table>
<caption>Behavior A.2: Stores a function pointer if initialized from a function or a function pointer</caption>
<thead>
<tr><th>Outcome</th><th>Library</th></tr>
</thead>
<tbody>

<tr>
<td>

Well-defined:

```cpp
opt.action = ssh.get_download_callback();
```

Well-defined:
```cpp
opt.action = download;
```

</td>
<td>

<span style="color: rgb(43, 57, 144);">■</span>`folly::FunctionRef`

<span style="color: rgb(255, 207, 171);">■</span>`gdb::function_view`

<span style="color: rgb(158, 16, 32);">■</span>`type_safe::function_ref`

<span style="color: rgb(227, 178, 0);">■</span>`absl::function_ref`

</td></tr>
</tbody></table>

P0792R5 wording gives **Behavior A.1**.

A related question is what happens when initialized from pointer-to-members. In the following tables, assume `&Ssh::connect` is a pointer to member function:

<table>
<caption>Behavior B.1: Stores a pointer to pointer-to-member if initialized from a pointer-to-member</caption>
<thead>
<tr><th>Outcome</th><th>Library</th></tr>
</thead>
<tbody>

<tr>
<td>

Well-defined:

```cpp
lib.send_cmd(&Ssh::connect);
```

Undefined:
```cpp
function_ref<void(Ssh&)> cmd = &Ssh::connect;
```

</td>
<td>

<span style="color: rgb(132, 190, 106);">■</span>`tl::function_ref`

<span style="color: rgb(43, 57, 144);">■</span>`folly::FunctionRef`

<span style="color: rgb(227, 178, 0);">■</span>`absl::function_ref`

</td></tr>
</tbody></table>

<table>
<caption>Behavior B.2: Only supports callable entities with function call expression</caption>
<thead>
<tr><th>Outcome</th><th>Library</th></tr>
</thead>
<tbody>

<tr>
<td>

Ill-formed:

```cpp
lib.send_cmd(&Ssh::connect);
```

Ill-formed:
```cpp
function_ref<void(Ssh&)> cmd = &Ssh::connect;
```

Well-defined:

```cpp
lib.send_cmd(std::mem_fn(&Ssh::connect));
```

</td>
<td>

<span style="color: rgb(9, 99, 125)">■</span>`llvm::function_ref`

<span style="color: rgb(255, 207, 171);">■</span>`gdb::function_view`

<span style="color: rgb(158, 16, 32);">■</span>`type_safe::function_ref`

</td></tr>
</tbody></table>

P0792R5-R7 wording gives **Behavior B.1**.


### Additional information

P2472 "make `function_ref` more functional" [^p2472r3] suggests a way to initialize `function_ref` from pointer-to-members without dangling in all contexts:

```cpp
function_ref<void(Ssh&)> cmd = nontype<&Ssh::connect>;
```

Not convertible from pointer-to-members means that `function_ref` does not need to use `invoke_r` in the implementation, improving debug codegen in specific toolchains with little effort.

Making `function_ref` large enough to fit a thunk pointer plus any pointer-to-member-function may render `std::function_ref` irrelevant in the real world. Some platform ABIs can pass a trivially copyable type of a 2-word size in registers and cannot do the same to a bigger type. Here is some LLVM IR to show the difference:
https://godbolt.org/z/Ke3475vz8.

For good or bad, the expression <code>&amp;_qualified-id_</code> that retrieves pointer-to-member shares grammar with the expression that gets a pointer to explicit object member functions. [[expr.unary.op/3]](https://eel.is/c++draft/expr.unary.op#3)


## Design decisions

- **Behavior A.2** is incorporated to eliminate the difference between initializing `function_ref` from a function and initializing `function_ref` from a function pointer.

- **Behavior B.2** is incorporated to reduce potential damage at a small cost. This change will reflect on the constructor's constraints.

- LEWG further requested making converting assignment from anything other than functions and function pointers ill-formed. Note that `function_ref` will still be copy-assignable.

- LEWG achieved consensus on [P2472R3](#Additional-information), and the wording is merged here. This change brings back the pointer-to-member support in a different form.


## Wording

The wording is relative to [N4910](https://wg21.link/N4910).

Add new templates to [[utility.syn]](https://eel.is/c++draft/utility.syn), header `<utility>` synopsis, after `in_place_index_t` and `in_place_index`:

<pre>
namespace std {
  [...]

  <i>// nontype argument tag</i>
  template&lt;auto V&gt;
    struct nontype_t {
      explicit nontype_t() = default;
    };

  template&lt;auto V&gt; inline constexpr nontype_t&lt;V&gt; nontype{};
}
</pre>

Add the template to [[functional.syn]](https://eel.is/c++draft/functional.syn), header `<functional>` synopsis:

> [...]
<pre>
  <i>// [func.wrap.move], move only wrapper</i>
  template&lt;class... S&gt; class move_only_function;        <i>// not defined</i>
  template&lt;class R, class... ArgTypes&gt;
    class move_only_function&lt;R(ArgTypes...) <i>cv ref</i> noexcept(<i>noex</i>)&gt;; <i>// see below</i>

  <ins><i>// [func.wrap.ref], non-owning wrapper</i>
  template&lt;class... S&gt; class function_ref;              <i>// not defined</i>
  template&lt;class R, class... ArgTypes&gt;
    class function_ref&lt;R(ArgTypes...) <i>cv</i> noexcept(<i>noex</i>)&gt;;           <i>// see below</i></ins>
</pre>
> [...]


Create a new section "Non-owning wrapper", `[func.wrap.ref]` with the following:

> ## General
> [func.wrap.ref.general]
>
> The header provides partial specializations of `function_ref` for each combination of the possible replacements of the placeholders *cv* and *noex* where:
>
> - *cv* is either `const` or empty.
> - *noex* is either `true` or `false`.
>

<br>

> ### Class template `function_ref`
> [func.wrap.ref.class]

<pre>
namespace std
{
  template&lt;class... S&gt; class function_ref;    <i>// not defined</i>

  template&lt;class R, class... ArgTypes&gt;
  class function_ref&lt;R(ArgTypes...) <i>cv</i> noexcept(<i>noex</i>)&gt;
  {
  public:
    <i>// [func.wrap.ref.ctor], constructors and assignment operators</i>
    template&lt;class F&gt; function_ref(F*) noexcept;
    template&lt;class F&gt; constexpr function_ref(F&amp;&amp;) noexcept;
    template&lt;auto f&gt; constexpr function_ref(nontype_t&lt;f&gt;) noexcept;
    template&lt;auto f, class T&gt;
      constexpr functIon_ref(nontype_t&lt;f&gt;, T&) noexcept;
    template&lt;auto f, class T&gt;
      constexpr function_ref(nontype_t&lt;f&gt;, <i>cv</i> T*) noexcept;

    constexpr function_ref(const function_ref&amp;) noexcept = default;
    constexpr function_ref&amp; operator=(const function_ref&amp;) noexcept = default;
    template&lt;class T&gt; function_ref&amp; operator=(T) = delete;

    <i>// [func.wrap.ref.inv], invocation</i>
    R operator()(ArgTypes...) const noexcept(<i>noex</i>);
  private:
    template&lt;class... T&gt;
      static constexpr bool <i>is-invocable-using</i> = <i>see below</i>;   <i>// exposition only</i>
  };

  <i>// [func.wrap.ref.deduct], deduction guides</i>
  template&lt;class F&gt;
    function_ref(F*) -> function_ref&lt;F&gt;;
  template&lt;auto f&gt;
    function_ref(nontype_t&lt;f&gt;) -> function_ref&lt;<i>see below</i>&gt;;
  template&lt;auto f&gt;
    function_ref(nontype_t&lt;f&gt;, auto) -> function_ref&lt;<i>see below</i>&gt;;
}
</pre>

<br>

> An object of class <code>function_ref&lt;R(Args\.\.\.) _cv_ noexcept(_noex_)&gt;</code> stores a pointer to thunk _`thunk-ptr`_ and a bound entity _`bound-entity`_.
> The bound entity has an implementation-defined type `BoundEntityType`.
> `BoundEntityType` is trivially copyable and models `copyable`.
> `BoundEntityType` is capable of storing a pointer to object value, a pointer to function value, an unused value, or a null pointer value.
> A thunk is a function of signature <code>R(BoundEntityType, Args&amp;&amp;\.\.\.) noexcept(_noex_)</code>.
>
> Each specialization of `function_ref` is a trivially copyable type [[basic.types]](https://eel.is/c++draft/basic.types).
>
> Within this subclause, _`call-args`_ is an argument pack with elements that have types `Args&&...` respectively.


<br>
<br>

> ### Constructors and assignment operators
> [func.wrap.ref.ctor]

<pre>
template&lt;class... T&gt;
  static constexpr bool <i>is-invocable-using</i> = <i>see below</i>;
</pre>
> If *noex* is true, <code><i>is-invocable-using</i>&lt;T\.\.\.&gt;</code> is equal to:
>
> &nbsp;&nbsp;`is_nothrow_invocable_r_v<R, T..., ArgTypes...>`
>
> Otherwise, <code><i>is-invocable-using</i>&lt;T\.\.\.&gt;</code> is equal to:
>
> &nbsp;&nbsp;`is_invocable_r_v<R, T..., ArgTypes...>`
>

<br>

```
template<class F> function_ref(F* f);
```
> *Constraints:*
> - `is_function_v<F>` is `true`, and
> - <code><i>is-invocable-using</i>&lt;F&gt;</code> is `true`.
>
> *Effects:* Initializes _`bound-entity`_ with `f`, and _`thunk-ptr`_ to the address of a function such that <code>_thunk-ptr_(_bound-entity_, _call-args_\.\.\.)</code> is expression equivalent to <code>invoke_r&lt;R&gt;(f, _call-args_\.\.\.)</code>.
>

<br>

```
template<class F> constexpr function_ref(F&& f);
```
> Let `T` be `remove_reference_t<F>`.
>
> *Constraints:*
> - `remove_cvref_t<F>` is not the same type as `function_ref`,
> - `is_member_pointer_v<T>` is `false`, and
> - <code><i>is-invocable-using</i>&lt;<i>cv</i> T&amp;&gt;</code> is `true`.
>
> *Effects:* Initializes _`bound-entity`_ with `addressof(f)`, and _`thunk-ptr`_ to address of a function such that <code>_thunk-ptr_(_bound-entity_, _call-args_\.\.\.)</code> is expression equivalent to <code>invoke_r&lt;R&gt;(static_cast&lt;_cv_ T&amp;&gt;(f), _call-args_\.\.\.)</code>.
>

<br>

```
template<auto f> constexpr function_ref(nontype_t<f>) noexcept;
```

> *Constraints:* <code><i>is-invocable-using</i>&lt;decltype(f)&gt;</code> is `true`.
> 
> *Effects:* Initializes _`bound-entity`_ with an unused value, and _`thunk-ptr`_ to address of a function such that <code>_thunk-ptr_(_bound-entity_, _call-args_\.\.\.)</code> is expression equivalent to <code>invoke_r&lt;R&gt;(f, _call-args_\.\.\.)</code>.

<br>

```
template<auto f, class T>
  constexpr function_ref(nontype_t<f>, T& obj) noexcept;
```

> 
> *Constraints:* <code><i>is-invocable-using</i>&lt;decltype(f), <i>cv</i> T&amp;&gt;</code> is `true`.
> 
> *Effects:* Initializes _`bound-entity`_ with `addressof(obj)`, and _`thunk-ptr`_ to address of a function such that <code>_thunk-ptr_(_bound-entity_, _call-args_\.\.\.)</code> is expression equivalent to <code>invoke_r&lt;R&gt;(f, static_cast&lt;_cv_ T&amp;&gt;(obj), _call-args_\.\.\.)</code>.

<br>

<pre>
template&lt;auto f, class T&gt;
  constexpr function_ref(nontype_t&lt;f&gt;, <i>cv</i> T* obj) noexcept;
</pre>

> 
> *Constraints:* <code><i>is-invocable-using</i>&lt;decltype(f), <i>cv</i> T*&gt;</code> is `true`.
> 
> *Effects:* Initializes _`bound-entity`_ with `obj`, and _`thunk-ptr`_ to address of a function such that <code>_thunk-ptr_(_bound-entity_, _call-args_\.\.\.)</code> is expression equivalent to <code>invoke_r&lt;R&gt;(f, obj, _call-args_\.\.\.)</code>.

<br>

```
template<class T> function_ref& operator=(T) = delete;
```
> *Constraints:*
> - `T` is not the same type as `function_ref`, and
> - `is_pointer_v<T>` is `false`.
>

<br>
<br>

> ### Invocation
> [func.wrap.ref.inv]
>
<pre>
R operator()(ArgTypes... args) const noexcept(<i>noex</i>);
</pre>
> *Preconditions*: _`bound-entity`_ does not store a null pointer value.
>
> *Effects:* Equivalent to
> <code>return _thunk-ptr_(_bound-entity_, std::forward&lt;ArgTypes&gt;(args)\.\.\.);</code>

<br>
<br>

> ### Deduction guides
> [func.wrap.ref.deduct]
>
```
template<class F>
  function_ref(F*) -> function_ref<F>;
```
> *Constraints:* `is_function_v<F>` is `true`.

<br>

<pre>
template&lt;auto f&gt;
  function_ref(nontype_t&lt;f&gt;) -> function_ref&lt;<i>see below</i>&gt;;
</pre>
>
> Let `F` be `remove_pointer_t<decltype(f)>`.
>
> *Constraints:* `is_function_v<F>` is `true`.
>
> *Remarks:* The deduced type is `function_ref<F>`.

<br>

<pre>
template&lt;auto f&gt;
  function_ref(nontype_t&lt;f&gt;, auto) -> function_ref&lt;<i>see below</i>&gt;;
</pre>
> *Constraints:*
> - `decltype(f)` is of the form <code>R(G::*)(A\.\.\.) <i>cv</i> &amp;<sub><i>opt</i></sub> noexcept(<i>E</i>)</code> for a class type `G`, or
> - `decltype(f)` is of the form <code>R G::*</code> for a class type `G`, in which case let `A...` be an empty pack, or
> - `decltype(f)` is of the form <code>R(U, A\.\.\.) noexcept(<i>E</i>)</code>.
>
> *Remarks:* The deduced type is <code>function_ref&lt;R(A\.\.\.) noexcept(<i>E</i>)&gt;</code>.


<br>


## Feature test macro

> Insert the following to [[version.syn]](https://eel.is/c++draft/version.syn), header `<version>` synopsis, after `__cpp_lib_move_only_function`:
<pre>
#define __cpp_lib_function_ref 20XXXXL <i>// also in &lt;functional&gt</i>
</pre>


## Implementation Experience

A complete implementation is available from
<span style="color: rgb(185, 208, 240);">■</span>&nbsp;[zhihaoy/nontype_functional@v0.8.1](https://github.com/zhihaoy/nontype_functional/tree/v0.8.1).

Many facilities similar to `function_ref` exist and are widely used in large codebases. See [Survey](#Survey) for details.


## Acknowledgments

Thanks to **Agustín Bergé**, **Dietmar Kühl**, **Eric Niebler**, **Tim van Deurzen**, and **Alisdair Meredith** for providing very valuable feedback on earlier drafts of this proposal.

Thanks to **Jens Maurer** for encouraging participation and **Tomasz Kamiński** for the thorough wording review.


## References

[^jmfunctionview]: _Implementing function_view is harder than you might think_
http://foonathan.net/blog/2017/01/20/function-ref-implementation.html
[^boundmemberfunctions]: Extracting the Function Pointer from a Bound Pointer to Member Function.
_Using the GNU Compiler Collection (GCC)_
https://gcc.gnu.org/onlinedocs/gcc/Bound-member-functions.html
[^p0792r5]: _function_ref: a non-owning reference to a Callable_
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0792r5.html
[^p0288r9]: _move_only_function_
http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0288r9.html
[^functionrefstringview]: _On function_ref and string_view_
https://quuxplusone.github.io/blog/2019/05/10/function-ref-vs-string-view/
[^llvmfuncref]: The function_ref class template.
_LLVM Programmer's Manual_
https://llvm.org/docs/ProgrammersManual.html#the-function-ref-class-template
[^stringviewtemp]: _std::string_view accepting temporaries: good idea or horrible pitfall?_
https://www.foonathan.net/2017/03/string_view-temporary/
[^p2511r1]: _Beyond operator(): NTTP callables in type-erased call wrappers_
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2511r1.html
[^p2472r3]: _make function_ref more functional_
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2472r3.html