---
layout: post
title: "Exploring inband lifetimes by converting librustc_mir"
author: Mark Rousskov
---

The diff from the conversion can be found
[here](https://github.com/rust-lang/rust/compare/12ed235adc...Mark-Simulacrum:rustc-mir-inband).

## General Feeling

Inband lifetimes is a limited change, and does not feel like it greatly enhances code. However, it
also doesn't hurt much and feels slightly better in many cases.

However, there are numerous edge cases and slight pain points, many having to do with a lack of
known standard ways to do things. As such, many of the edge cases are likely to fall away as we
develop after stabilization and come up with standard methods to work with the new feature.

The primary work to migrate is essentially just deleting ~all lifetime headers (`<'a, 'b, 'c>`)
across impls and functions. More intensive migration would involve replacing untied/single-use
lifetimes with `'_` in all cases. This is quite hard to do from a person perspective (though
compiler can likely do so fairly easily).

## Concern points

### Determining what a lifetime's "scope" is

The problem here lies in `'tcx`-like lifetimes which are named repeatedly, e.g. the `'tcx` lifetime
below is distinct from the `'tcx` lifetime in the parent.

Inband lifetimes remove the requirement to declare lifetimes at each scope. This is certainly a win
for horizontal codespace and for ease of adding a lifetime (two places, input and output, instead of
three) but is a loss for determining whether the lifetime being referenced comes from the function,
that is, is freshly declared, related to the lifetime of each call or comes from a parent impl
block.

```rust
# 2015
impl<'a, 'tcx> Foo for Bar<'a, 'tcx> {
    fn foo(&self) {
        // ... code ...
        fn baz<'a, 'tcx>(baz: &'a mut Bar<'a, 'tcx>) -> &'a mut Vec<u32> {
            unimplemented!()
        }
    }
}
```

```rust
# 2018
impl Foo for Bar<'a, 'tcx> {
    fn foo(&self) {
        // ... code ...
        // where does this 'a refer to? In order to know, I need to look at indent levels essentially --
        // harder than before, to an extent.
        fn baz(baz: &'a mut Bar<'a, 'tcx>) -> &'a mut Vec<u32> {
            unimplemented!()
        }
    }
}
```

Another case that introduces friction (especially, I think, for new users):

```rust
# 2018
impl MirBorrowckCtxt<'cx, 'gcx, 'tcx> {
    fn report_use_of_moved_or_uninitialized(
        &mut self,
        curr_move_out: &FlowAtLocation<MovingOutStatements<'_, 'gcx, 'tcx>>,
    ) {
        unimplemented!()
    }
}
```

The `&FlowAtLocation<MovingOutStatements<'_, 'gcx, 'tcx>>` type for `curr_move_out`: what does the
`'_` refer to here? Some new lifetime akin to what in 2015 would have `'a` on the `fn`,
not the `impl`? A new user might become confused, while "experienced" Rust programmers would likely
relatively quickly determine that this is equivalent to `<'a>` on the fn in Rust 2015. However, it
would take a slight amount of thought

This specific pain point is grouped in with determining scope because previously such a lifetime
could always be 'looked up' whereas now `'_` is never declared.

### Need for `'_` in already borrowed data

Consider `&Mir<'_>`: what benefit does having the `'_` there give us? The user is given no
additional information, because the `&` already indicates that the type is borrowed data for some
lifetime. The reason we omit requiring `'_` on `&` and `&mut` is because it already
obvious to the reader that there is a non-'static lifetime there.

Can we do the same for any type that is explicitly behind `&`/`&mut` as well? This seems a natural
extension of the rules, though perhaps may not communicate quite as much information when
copying/moving code around.

### The ever-prevalent `'a` lifetime

```rust
fn mir_borrowck<'a, 'tcx>(tcx: TyCtxt<'a, 'tcx, 'tcx>, def_id: DefId) -> BorrowCheckResult<'tcx>
```

This definition uses the typical `'a` lifetime to indicate the TyCtxt's internal borrows, because
`TyCtxt` is actually just a pair of references. The straightforward way of rewriting this is to
simply omit the lifetime declarations:

```rust
fn mir_borrowck(tcx: TyCtxt<'a, 'tcx, 'tcx>, def_id: DefId) -> BorrowCheckResult<'tcx>
```

I would argue though that this should be unidiomatic. Instead, a better way of rewriting this
function would be:

```rust
fn mir_borrowck(tcx: TyCtxt<'_, 'tcx, 'tcx>, def_id: DefId) -> BorrowCheckResult<'tcx>
```

The initial intuition here is that any lifetime that is unnamed in all code should be replaced with `'_`.

However, I'm not sure that `'_` is quite the best way to do this. It might be that I'm just not used
to this yet, but today, at least, whenever you see a `'a` lifetime that generally just means that
either an explicit lifetime is necessary to tie a few lifetimes together, or you're in a struct/impl
declaration with one lifetime.

Some part of me really likes `'a` over `'_` because to me, seeing `a` there immediately communicates
the unimportance of the lifetime, and lack of meaning attached to the lifetime. Unlike `a`, though,
`_` stands out quite a bit, especially in (most) fonts where the `'` and `_` symbols are quite far
apart vertically, and is harder to ignore. It also looks more sigil-y when compared to `'a`; `'_`
looks far more special than `'a`. In some ways it is, but it's entire point is to de-emphasize
itself.

All this is somewhat nebulous; it's unclear whether this is an initial feeling and we'll all get
used to seeing `'a` replaced with `'_`. However, it's also not completely obvious that such a
replacement is always possible: for example, if the above function returned something more
complicated:

```rust
# 2015
fn mir_borrowck<...>(tcx: TyCtxt<'a, 'tcx, 'tcx>, def_id: &DefId) -> BorrowCheckResult<'a, 'tcx>`
# 2018 (but doesn't compile)
fn mir_borrowck(tcx: TyCtxt<'_, 'tcx, 'tcx>, def_id: &DefId) -> BorrowCheckResult<'_, 'tcx>`
```

This function can't be written with `'_` in the output, because that would be ambiguous. But, we
have no better name to give it. We want `'a` but that is (per the suggested rules) generally not the
best idea, might lead to needing to allow lints, that sort of thing.

I believe that in the past people have simply gone with `'a` here and if necessary cycled to `'b'`,
`'c`, etc. as they need more lifetimes, but the new approach tends away from that, tending towards
trying to name all lifetimes explicitly.

The overall conclusion here is that `'_` is primarily only usable for eliding any name in the input
parameters. Elision through it in the output parameters is likely to fail with time, as the above
code shows. This makes me believe that a valid approach may be to limit the use of `'_` to input
arguments, and disallow it in the output parameters.

Does this mean we shouldn't support `'_` at all? I would argue that the answer is yes. In all cases
I've come up with, replacing `'_` with an explicit tie to the input parameters isn't that much more
work, and communicates intent far better. This is *especially* true to a beginner, who may believe
that `'_` is special (it, afterall, looks special) whereas in most cases seeing the explicit
lifetime tie will be much clearer and less magic-feeling. Lifetimes are an area of the language
where beginners are often already confused, so eliminating magic seems like a good general sense. We
can also always permit elision through `'_` in the future.

### Closure Confusion

In the case where closure parameters are declared with lifetime names, it is unclear that these are
not 'fresh' names. See, for example, `|mir: &mut MirBorrowckCtxt<'cx, 'gcx, 'tcx>|`: is the closure
generic over these lifetimes, or is it taking them from the parent scope? That is, are we treating
the function somewhat like an `impl` block and inheriting lifetimes from it, or is the closure more
like `fn foo<'cx, 'gcx, 'tcx>`?

Inband lifetimes make the distinction in a closure less clear, as especially new users may see these
'cx, 'gcx, and 'tcx as re-declaring new lifetimes. This is especially true where `'a` or similar
generic names. Previously we'd be able to say that you can look for `<...>` to find the
"declaration" and no other place could declare lifetimes or types. This is no longer the case.

This also means that rewriting a closure without captures to a `fn` was never lossless, but
previously the compiler would say "please declare these lifetimes" when making such an attempt. Now,
no such warning/error is issued, but behavior has changed.

### Lifetimes cannot use keyword names

For example, `fn outgoing_edges(&'self self) -> Edges<'self>` is not permitted; which is not all
that nice. It seems like inband lifetimes are going to drive people towards wanting to use keywords
in lifetime names and it's rather ugly to do `'self_` or the compiler's semi-convention of `this`.
I suppose the primary use case for reserving `'self` as a lifetime is to later be able to use it for
safe self-referential structs and the like, but perhaps we can allow it inside functions? There may
be a good middle ground here.

### Repeated use of `'_`

For example, `TyCtxt<'_, '_, '_>`. This looks somewhat like it's tying the lifetimes of the TyCtxt
together, though of course after we explain that `'_` is special then perhaps you won't think that.
But it does seem like another case where `'_` feels possibly harmful.

But it is actually quite common to see cases where nothing *actually* cares that the lifetimes are
tied together. For example, often you see code take a TyCtxt, where unless you need it to be
local (via `'tcx` being passed into both the `'gcx` and `'tcx` slots) you likely don't actually care
what the lifetimes are; you just need three distinct lifetimes. This use case seems like a
surface-level perfect fit for `'_`.

### When named lifetimes don't matter

Consider `&Place<'tcx>` inside a function that also takes a `TyCtxt` with that lifetime or impl
block of similar nature. There's nothing you can actually do with the `Place` that would use the
fact that it's lifetime parameter is `'tcx` in most cases (if it's variant over that lifetime, of
course), which means that this feels like it should instead be `&Place<'_>`. But is that misleading?
All Places likely want to be `'tcx` in practice, so perhaps there's no reason for us to elide the
lifetime there.

`impl<'tcx> ToRegionVid for &'tcx RegionKind` uses `'tcx`, but this code arguably should be `'_` or
`'a`: the lifetime of the reference is absolutely unimportant.

Also `Ty<'tcx>` as a param where the output is `bool` or some such.

This is actually quite common in rustc code, especially where a type could contain tcx data but in
this case contains static data (e.g., enum with a reference in one variant, u32 in the other).

### Replacing multiple 'a, 'b with '_ is really nice

```
impl<'a, 'b, 'gcx, 'tcx> TypeOutlivesDelegate<'tcx> for &'a mut ConstraintConversion<'b, 'gcx, 'tcx>
impl TypeOutlivesDelegate<'tcx> for &'_ mut ConstraintConversion<'_, 'gcx, 'tcx>
````

This is really nice, and a perfect use case for `'_`: the lifetimes don't matter, but we need to
specify them. More so, using `'_` also gives the added benefit that *using* the lifetime becomes
impossible: you can't refer to `'_` from inside the impl block.

### Error: lifetimes used in `fn` or `Fn` syntax must be explicitly declared using `<...>` binders

Quite unexpected, and especially awkward when the lifetime must be declared at impl scope, causing
all other lifetimes declared to no longer be able to use inband.

See also [this issue](https://github.com/rust-lang/rust/issues/52532), which indicates that this is
expected behavior.

### When you have lots of functions taking 'a

Harder to tell if all of these are bound to some root struct/scope. Previously you'd declare them
via `<'a>` at each function.

## Thoughts for standard choices

### Tie lifetime naming

Anytime a need arises to tie lifetimes together, use the argument name instead of `'a` or similar if
no better name exists (e.g., `'tcx` can be a better name). This makes it easier to speak about the
lifetimes.

### Unused lifetimes in impl blocks

Consider `impl Iterator for Edges<'_>`: is that preferable to `impl Iterator for Edges<'graph>`?

I posit that the answer is no: explicitly stating the lifetime provides better context and more
information to the user. However, it is more verbose.

I do think that in general we should eventually lint for "unused" lifetimes in impls as well,
suggesting `'_` if that's the pattern we recommend. This is made more difficult by the fact that
today we do not lint at all for unused lifetimes.

However, this distinction becomes more murky with structs that are essentially bags of references,
where the lifetime is actually uninteresting and has no good name, e.g., for `impl Iterator for
TyCtxt<'a, 'gcx, 'tcx>`: the `'a` there is uninteresting and would like be better replaced with
`'_`.

### Single-use lifetimes are actually quite common

`TyCtxt<'a, 'gcx, 'tcx>` is often used in old code and can often be replaced with
`TyCtxt<'_, '_, 'tcx>`. It is unclear at this point whether doing so in all cases is beneficial.
