# Angstrom

Angstrom is a parser-combinator library that makes it easy to write efficient,
expressive, and reusable parsers suitable for high-performance applications. It
exposes monadic and applicative interfaces for composition, and supports
incremental input through buffered and unbuffered interfaces. Both interfaces
give the user total control over the blocking behavior of their application,
with the unbuffered interface enabling zero-copy IO. Parsers are backtracking
by default and support unbounded lookahead.

[![Build Status](https://travis-ci.org/inhabitedtype/angstrom.svg?branch=master)](https://travis-ci.org/inhabitedtype/angstrom)


## Installation

Install the library and its dependencies via [OPAM][opam]:

[opam]: http://opam.ocaml.org/

```bash
opam install angstrom
```

## Usage

Angstrom is written with network protocols and serialization formats in mind.
As such, its source distribution includes implementations of various RFCs that
are illustrative of real-world applications of the library. These include an
[HTTP parser][http] and a [JSON parser][json].

[http]: https://github.com/inhabitedtype/angstrom/blob/master/rfcs/rFC2616.ml
[json]: https://github.com/inhabitedtype/angstrom/blob/master/rfcs/rFC7159.ml

In addition, it is an informal tradition for OCaml parser-combinator libraries
to include in their READMEs a parser for a simple arithmetic expression
language. The code below implements a parser for such a language and computes
the numerical result of the expression as it is being parsed. Because Angstrom
is written with network protocols and serialization libraries in mind, it does
not include combinators for creating infix expression parsers. Such
combinators, e.g., `chainl1`, are nevertheless simple to define.

```ocaml
open Angstrom

let parens p = char '(' *> p <* char ')'
let add = char '+' *> return (+)
let sub = char '-' *> return (-)
let mul = char '*' *> return ( * )
let div = char '/' *> return (/)
let integer =
  take_while1 (function '0' .. '9' -> true | _ -> false) >>| int_of_string

let chainl1 e op =
  fix (fun r ->
    e >>= fun x -> (op <*> (return x) <*> r) <|> return x)

let expr : int t =
  fix (fun expr ->
    let factor = parens expr <|> integer in
    let term   = chainl1 factor (mul <|> div) in
    chainl1 term (add <|> sub))

let eval (str:string) : int =
  match parse_only expr (`String str) with
  | Result.Ok v      -> v
  | Result.Error msg -> failwith msg
```


## Comparison to Other Libraries

There are several other parser-combinator libraries available for OCaml that
may suit your needs, and are worth considering. Most of them are derivatives of
or inspired by [Parsec][]. As such, they require the use of a `try` combinator
to achieve backtracking, rather than providing it by default. They also all use
something akin to a lazy character stream as the underlying input abstraction.
While this suits Haskell quite nicely, it requires blocking read calls when the
entire input is not immediately available&mdash;an approach that is inherently
incompatible with monadic concurrency libraries such as [Async] and [Lwt], and
writing high-performance, concurrent applications in general. Another
consequence of this approach to modeling and retrieving input is that the
parsers cannot iterate over sections of input in a tight loop, which adversely
affects performance.

Below is a table that compares the features of Angstrom agains the those of
other parser-combinator libraries.

[parsec]: https://hackage.haskell.org/package/parsec
[async]: https://github.com/janestreet/async
[lwt]: https://ocsigen.org/lwt/


Feature \ Library                   | Angstrom | [mparser] | [planck] | [opal] |
------------------------------------|:--------:|:---------:|:--------:|:------:|
Monadic interface                   | ✅        | ✅         | ✅        | ✅      |
Backtracking by default             | ✅        | ❌         | ❌        | ❌      |
Unbounded lookahead                 | ✅        | ✅         | ✅        | ❌      |
Reports line numbers in errors      | ❌        | ✅         | ❌        | ❌      |
Efficient `take_while`/`skip_while` | ✅        | ❌         | ❌        | ❌      |
Unbuffered (zero-copy) interface    | ✅        | ❌         | ❌        | ❌      |
Non-blocking incremental interface  | ✅        | ❌         | ❌        | ❌      |
Async Support                       | ✅        | ❌         | ❌        | ❌      |
Lwt Support                         | ✅        | ❌         | ❌        | ❌      |

[mparser]: https://github.com/cakeplus/mparser
[opal]: https://github.com/pyrocat101/opal
[planck]: https://bitbucket.org/camlspotter/planck


## Development

To install development dependencies, pin the package from the root of the
repository:

```bash
opam pin add -n angstrom .
opam install --deps-only angstrom
```

After this, you may install a development version of the library using the
install command as usual.

For building and running the tests during development, you will need to install
the `alcotest` package and reconfigure the build process to enable tests:

```bash
opam install alcotest
./configure --enable-tests
make && make test
```

## Acknowledgements

This library started off as a direct port of the inimitable [attoparsec][]
library. While the original approach of continuation-passing still survives in
the source code, several modifications have been made in order to adapt the
ideas to OCaml, and in the process allow for more efficient memory usage and
integration with monadic concurrency libraries. This library will undoubtedly
diverge further as time goes on, but it's name will stand as an homage to its
origin.

[attoparsec]: https://github.com/bos/attoparsec


## License

BSD3, see LICENSE file for its text.
