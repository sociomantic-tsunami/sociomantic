Relevant Concepts Explained
===========================

assert
------

`assert` is D programming language built-in intended to help catching programmer
errors and verify the application's internal sanity. It is not intended for
error handling and any assert violation is normally considered to be a critical
application failure that can only be handled by immediate termination of the
process.

In D1, failed assertions throw `AssertException`, which can in fact be caught
(and some Sociomantic projects do that), thus the "fatal" behaviour is only a
convention.

In D2, failed assertions throw `AssertError`, which (being an `Error`) is
ignored by `catch (Exception)`. It can still be caught by `catch(Error)` or
`catch(Throwable)` but that may leave the program in a broken state, by
[specification](http://dlang.org/expression.html#AssertExpression). In practice,
that means that destructors and `scope(exit)` blocks may be ignored in some
situations, depending on compiler implementation, making it suitable only to
peform slightly more verbose message printing or data dump before termination.
Recovery from `Error` is considered impossible.

When compiled with `-release`, assertions are completely disabled and
execution will continue normally even if condition would be violated in
non-release builds, possibly corrupting the program if it does not have any
other protection checks.

`assert(0)` (with `0` being known at compile-time) is a special case in both D1
and D2.  It remains in the program even when compiled with `-release` and gets
replaced with an `HLT` instruction which will usually cause program to crash
because, on modern linux systems, this instruction is only allowed in kernel
privileged mode. Any custom assertion message will be ignored in that case.
`assert(0)` means "unreachable code" and is recognized as such by the compiler
during code flow analysis (i.e. checking for unreachable statements with `-w`).

Contracts
---------

The official spec has an extensive [article](http://dlang.org/contracts.html)
describing the "Design by Contract" ideology. To put it shortly, contracts are
extended versions of assertions that put additional semantics on how exactly
program sanity is checked.

Functions/methods have `in` and `out` contracts, checking pre-conditions and
post-conditions respectively. `out` contracts have access to the function's
return value; both have access to the function's argument list (at given states
of the function).

Aggregate types also have `invariant` contracts which get called whenever a
constructor, destructor, or any public method of the aggregate is called (check
official docs for a more precise explanation of the invariant rules).

The same as assertions, invariants are completely removed from the program when
compiled in `-release` mode and have the same intended meaning -- verification
of internal program sanity.

enforce
-------

`enforce` is a helper function provided by the `ocean.core.Exception` module and
originally inspired by Phobos' `std.exception.enforce`. It mimicks the syntax of
`assert`, but throws a user-defined exception object (`new Exception` by
default).

Enforcements are designed to be a tool for error handling, to conveniently
protect against possible error conditions that are unlikely in the main code
flow but can still happen in deployed applications.

Issues
======

The main problem with the proposed "design by contract" model is the assumption
that once the program is sufficiently tested in debug mode it is safe to remove
assertions and contracts without compromising correctness (for better
performance).

However, in the case of a server application working primarily with external
data, it is very hard to get to the point of such confidence, especially if any
error can have a potentially very costly impact on business. Even full test
coverage can be misleading. This has resulted in some application never using
the `-release` flag out of fear of possible uncaught programmer errors.

After some discussion, it was agreed that we need a more practical approach to
using contracts, even if it will violate the intended DbC ideology. Most known
issues seem to be associated with `in` contracts, because those do not notify
the programmer when violation of some internal assumption occurs but rather at
some later point when the violation is about to cause application corruption and
must be protected against at all costs.

At the same time, the actual performance penalty of calling such safety checks
is often very small, being as simple as checking for null pointers or integer
boundaries. Any possible performance gain is simply not worth the added risk of
removing the check (one case where it can make a difference, though, is in very
small functions where extra operations can make the difference between inlining
or not).

Goals
=====

All suggested guidelines come from a simple goal - eventually it should be
possible to compile our applications with `-release` flag without introducing
serious business risks.

"serious business risks" here mean anything that will cause money loss to
the company - logical violations that can result in broken bid values,
memory corruption that can damage stored user profiles, simultaneous
denial of service for majority of deployed services. Crashing or
small non-corrupting data loss is not considered a serious risk because
at current system scale it doesn't make a big impact for the system as
a whole.

Considering the before-mentioned issues this goal limits contracts/asserts to
relative narrow specialization - introducing additional costly sanity checks to
help in debugging and testing. With such an approach application compiled in
release mode won't compromise safety - it will simply provide less convenient
error messages.

Guidelines
==========

What Kind of Error Check to Use?
--------------------------------

1. Checks for programmer error / basic sanity or safety (e.g. pointer checks,
integer expressions) should be performed using ocean's `verify`
(`ocean.core.Verify`), which throws an exception of type `SanityException` upon
failure. These checks are still present in release builds.

2. Checks on external data should use ocean's `enforce` (`ocean.core.Enforce`)
(or another exception-throwing mechanism), but _not_ `verify`. This is in order
to distinguish types of error. These checks are still present in release builds.

3. The built-in `assert` should _only_ be used as a means to spot issues earlier
in the debugging process. It should never be relied upon for error checking in
a live application, as it is not present in release builds. In practice, this
means that anything checked by `assert`s must also be covered at some point in
the code by `verify`, `enforce`, etc. In this way, the error conditions will
still be checked in release builds, but non-release builds can benefit from
additional error checks to aid debugging.

4. Contracts (`in`, `out`, `invariant`) should only use `assert`, not `verify`,
`enforce`, etc. This is because contracts are not compiled in release builds, so
placing release-build checks such as `verify` inside contracts is useless.

General Error Checking Principles
---------------------------------

1. Try to verify any external data (e.g. coming from the DHT or database or SSP
requests) as early as possible with `enforce`s or similar exception-throwing
checks. Using tagged type wrappers is one approach of ensuring that no part of
the application works with non-validated data by accident. One example of such a
type is the `Contiguous` struct from `ocean.util.serialize.contiguous`.

2. When thinking about application memory safety, assume that no contracts are
present. They are to make debugging the issue more convenient, not to actually
prevent it happening.

3. `in` contracts (containing `assert`s) should only be used on private methods/
functions which are called via a public API including `verify`/`enforce` checks.
`in` contracts on non-private methods/functions should be avoided.

4. `out` and `invariant` contracts are good, but should never be used to
_protect_ against errors. They are well-suited to perform computationally costly
checks that verify program sanity and help find the issue as early as possible.

5. As an extra safety measure for multi-instance services, do at least one test
deployment of a live service compiled with debug mode before updating all
services in all regions / servers. One service which responds more slowly (due
to being built in debug mode) shouldn't have a critical impact on the overall
performance of the system, but it can help to find any additional issues
triggered only by real-world traffic load.

Adapting Older Code
-------------------

A lot of code was written before the advent of `enforce` and `verify`. The
following rules of thumb may be helpful when adapting such code to comply to the
previous guidelines:

* Generally, whenever an `assert` is present, you must ensure that the condition
is being checked by a `verify` or `enforce` at some point in the code. If this
is not the case, convert the `assert` to a `verify` statement.

* All checks in `in` contracts should be converted to `verify` statements in the
method/function body.

* `out` contracts and `invariant`s can remain as they are, using `assert`s. They
will not be present in release builds.

