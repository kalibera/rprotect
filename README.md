
This is a report from running PROTECT bug finding tools `bcheck` and
`maacheck` from [rchk](http://www.github.com/kalibera/rchk) on R packages. 
The tools are based on static analysis, they analyze compiled code of the
packages without actually running them.  Consequently, the tools can find
errors that are not covered by any tests and the tools report the lines of
code with the error, so errors are easy to locate and fix.  The downside is
that the tools are sometimes report false alarms (tools based on static
analysis in principle do) --- some reported errors are not really errors. 
Often, however, the code is worth making more readable anyway in these
cases.  The tools are [available](http://www.github.com/kalibera/rchk) but
using them is somewhat difficult because one needs to build native code into
LLVM bitcode (using Link-Time-Optimization, Clang and Clang++), which is a
rather painful process.

## The danger of PROTECT errors

Failure to PROTECT an R object can lead to memory corruption, which usually
causes crash (segfault) of R.  It is usually very hard to track down these
errors for a number of reasons (the crash is far from where the error
happened, there is no useful error message, attempts to simplify the error
case into a small reproducible example make the error disappear, PROTECT
errors usually stay undetected for long in the code and only cause a crash
much later due to innocuous, unrelated changes).  Sometimes a PROTECT error
causes an R error at runtime that has no logical explanation.  A PROTECT
error could also lead to incorrect results for programs in R.  Given these
difficulties, there is a need for tools that specifically aim at detecting
PROTECT errors, such as `rchk`.  Another tool is
[gctorture](https://cran.r-project.org/doc/manuals/r-release/R-exts.html#Using-gctorture)
which increases the chances that PROTECT errors will trigger (cause a crash
or error that can be detected with strict barrier checking) while running
tests.

## How to PROTECT

One has to make sure that during any allocation from the R heap, all R
objects (values) that may still be used by the program are reachable by the
garbage collector.  The garbage collector will find objects that are on the
*precious list* (`Rf_PreserveObject` and `Rf_ReleaseObject`), on the
*protect stack* (`PROTECT`, `PROTECT_WITH_INDEX`, `REPROTECT`, `UNPROTECT`)
or in some known locations, such as the symbol table.  The garbage collector
will also find objects that are pointed to (via SEXP) by objects it already
knows about. In the following, the word "pointer" refers to SEXPs (not to
raw C pointers pointing outside R heap).

These general rules are recommended to keep the source code readable and to
reduce the risk of new errors being introduced as the code evolves.  They
are based on analyzing and fixing a number of PROTECT bugs in the R runtime:

1. **pointer protection balance** inside every C function: balance
PROTECT/UNPROTECT so that the usage of the pointer protection stack is the
same at the beginning and at the end of the function. A non-local return in
case of R error (`Rf_error`) is an exception: the balance there is handled
by R runtime. In particular, return values from functions are unprotected -
they have to be protected by the caller. From this it follows that one
should never wrap `PROTECT` or `UNPROTECT` functions.

2. **caller protection**: all arguments passed to a function must be protected
by the caller before the call.

3.  **all functions allocate**: it is highly recommended to conservatively
assume that all functions allocate, and hence to have all local variables
protected before calling a function.  There have been a large number of
PROTECT bugs because of incorrect assumptions that a function does not
allocate or does not allocate in some circumstances; it often is infeasible
to check all callers (recursively!) whenever introducing allocation into a
function. The R API definitely does not guarantee that certain functions
won't allocate and allocation gets often introduced (note that e.g. a
typical thing such as introducing a warning means introducing also
allocation).

4.  **any return value may need protection**: it is highly recommended to
conservatively assume that a pointer returned from any function needs
protection.  The only sane exception to this rule is `install`: it is safe
to assume that the returned pointer is protected implicitly by the symbol
table (but note, `install` and its variants still are function calls and
they allocate).

5.  Callee-protect functions should be an exception.  Still, if used, they
should protect all their arguments (not just some) and protect them for the
whole duration of the function, even if some of the values may not be needed
anymore in the execution of the function.

6. It is ok to depend on that an object pointer from a protected object is
also implicitly protected, but one should only use this when fewer
explicit protections actually make code more readable (comments may help).


## maacheck

This tool is quite simple and only reports instances of *multiple
allocating arguments* pattern, such as (from `Rcpp` code):

```
Rf_lang2(::Rf_install("simpleError"), Rf_mkString(str.c_str()))

```

This is a real error: `Rf_mkString` returns an unprotected newly allocated
object. `Rf_install` may allocate and may be called after `Rf_mkString`, and
hence may trigger garbage collection that will corrupt the object allocated
by `Rf_mkString`. The full text of this report is e.g.

```
WARNING Suspicious call (two or more unprotected arguments) to Rf_lang2 at string_to_try_error(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&) Rcpp/include/Rcpp/exceptions.h:236
```

## bcheck

This tool is far more sophisticated and looks for a wide range of protection
errors.  The tool is much more permissive than the recommended rules above:
it tries to find out which functions actually allocate, which are
callee-protect and for what arguments, which return a newly allocated
object, etc. It is intentionally made more permissive so that it can be used
in practice also on the code of R itself, where sometimes these rules are
violated in the interest of performance but at the cost at increasing code
complexity.

Still, the tool reports some false alarms as any bug-finding tool.  In the
interest of prevention of further PROTECT errors, it might yet still be
advisable to "fix" some "false alarms" by making the code conform to the
rules suggested above, like in the following example.  The reports usually
have multiple lines per C function where issues are found, such as (from
`curl` code)

```
Function R_handle_setopt
  [UP] unprotected variable optnames while calling allocating function Rf_asInteger curl/src/handle.c:183
  [UP] unprotected variable optnames while calling allocating function Rf_asReal curl/src/handle.c:201
```

The messages are self-explanatory. Functions `Rf_asInteger` and `Rf_asReal`
really may allocate, because they may issue a coercion warning and issuing a
warning involves allocation. The variable `optnames` holds a pointer
obtained from `getAttrib`:

```
SEXP optnames = getAttrib(values, R_NamesSymbol);
```

This pointer is not protected.  In practice `getAttrib` sometimes returns
newly allocated objects and sometimes existing objects (pointed from the
object they are attributes of).  In particular, the `names` attribute will
be newly allocated if `values` are represented as a pairlist (e.g.  it is a
language object).  Further analysis of `R_handle_setopt` reveals that
`values` will in fact be represented as a vector:

```
  if(!isVector(values))
    error("`values` must be a list");
```

So, in fact `optnames` are not newly allocated. Moreover, `optnames` are
pointed to from `values` and since `values` is an argument of the function,
it will be reachable by the GC at calls to `Rf_asInteger` and
`Rf_asReal`.

```
SEXP R_handle_setopt(SEXP ptr, SEXP keys, SEXP values){
  SEXP optnames = getAttrib(values, R_NamesSymbol);

```

Hence, technically speaking, with the present code of R and packages this
reported problem should not lead to an error and hence this report is a
false alarm.  However, note how complicated reasoning is needed to figure
this out.  One has to read and understand particularly the source code of
`getAttrib` implementation inside R.  And yet worse, this implementation may
change at any time e.g.  to return a newly allocated vector with the names. 
Also, this code in `curl` is just setting options, apparently it is not
performance critical and the price of a PROTECT would be irrelevant given the
rest of the code of `R_handle_setopt`.  So it would be worth fixing this
e.g.  to `PROTECT(optnames = getAttrib(values, R_NamesSymbol);`, which is
also in line with the rules mentioned above.

## Filtering the tool outputs

In the provided outputs from the tools, some reports were filtered out in
the interest of focusing on reports that are most likely true errors. 
Sometimes the tool knows it is confused by the code (it reports "results
will be incomplete" for a function which it does not sufficiently
understand).  While this may be also because of a true error in the code,
these reports were still excluded.  Also, reports mentioning `Rcpp` were
excluded, because they otherwise would appear with every package that uses
`Rcpp`, even recursively, thus polluting the reports.  Some of these reports
are because of the `Rcpp` design -- functions that wrap PROTECT indeed have
pointer protection imbalance, some of these are true errors like the first
example above. The tool has been primarily developed for checking C code and
it was debugged based on experience with finding bugs in the R runtime. It
works less well with C++ code, but it still has reported true PROTECT errors
in C++ code.

