---
title: Conditional Compilation
---

Sometimes you want to write code that works with different versions of compilers and libraries.

People used to use preprocessors like [C preprocessor](http://tigcc.ticalc.org/doc/cpp.html) for the C family languages. The OCaml community uses several preprocessors: [cppo](https://github.com/mjambon/cppo), [ocp-pp](https://github.com/OCamlPro/typerex-build/tree/master/tools/ocp-pp), camlp4 IFDEF macros, [optcomp](https://github.com/diml/optcomp) and [ppx optcomp](https://github.com/janestreet/ppx_optcomp).

Instead of a preprocessor, BuckleScript adds language-level static `if` compilation. It's less powerful than other preprocessors since it only supports static `if` (no `#define`, `#undefine`, `#include`), but there are several advantages.

- It’s tiny (only ~500 lines) and highly efficient. Everything can be done in a **single pass**. It's easy to rebuild the pre-processor into a standalone file, with no dependencies on compiler libs, to back-port it to old OCaml compilers.
- It’s purely functional and type-safe, and cooperates with editor tooling like Merlin.

**Note**: BuckleScript's conditional compilation doesn't work with Reason yet.

## Usage

`lwt_unix.mli`

```ocaml
type open_flag =
    Unix.open_flag =
  | O_RDONLY
  | O_WRONLY
  | O_RDWR
  | O_NONBLOCK
  | O_APPEND
  | O_CREAT
  | O_TRUNC
  | O_EXCL
  | O_NOCTTY
  | O_DSYNC
  | O_SYNC
  | O_RSYNC
#if OCAML_VERSION =~ ">=3.13" then
  | O_SHARE_DELETE
#end
#if OCAML_VERSION =~ ">=4.01" then
  | O_CLOEXEC
#end
```

You don't have to add anything to the build to have these work. The compiler `bsc` understands these already.

## Built-in & Custom Variables

See the output of `bsc.exe -bs-list-conditionals`:

```sh
> bsc.exe -bs-D CUSTOM_A="ghsigh" -bs-list-conditionals
OCAML_PATCH "BS"
BS_VERSION "1.2.1"
OS_TYPE "Unix"
BS true
CUSTOM_A "ghsigh"
WORD_SIZE 64
OCAML_VERSION "4.02.3+BS"
BIG_ENDIAN false
```

Add your custom variable to the mix with `-bs-D MY_VAR="bla"`:

```sh
> bsc.exe -bs-D MY_VAR="bla" -bs-list-conditionals
OCAML_PATCH "BS"
BS_VERSION "1.2.1"
OS_TYPE "Unix"
BS true
MY_VAR="bla"
...
```

## Concrete Syntax

```
static-if
| HASH-IF-BOL conditional-expression THEN //
   tokens
(HASH-ELIF-BOL conditional-expression THEN) *
(ELSE-BOL tokens)?
HASH-END-BOL

conditional-expression
| conditional-expression && conditional-expression
| conditional-expression || conditional-expression
| atom-predicate

atom-predicate
| atom operator atom
| defined UIDENT
| undefined UIDENT

operator
| (= | < | > | <= | >= | =~ )

atom
| UIDENT | INT | STRING | FLOAT
```

- IF-BOL means `#IF` should be in the beginning of a line.

## Typing Rules

- type of INT is `int`
- type of STRING is `string`
- type of FLOAT is `float`
- value of UIDENT comes from either built-in values (with documented types) or an environment variable, if it is literally `true`, `false` then it is `bool`, else if it is parsable by `int_of_string` then it is of type int, else if it is parsable by `float_of_string` then it is float, otherwise it would be string
- In `lhs operator rhs`, `lhs` and `rhs` are always the same type and return boolean. `=~` is a semantic version operator which requires both sides to be string.

Evaluation rules are obvious. `=~` respect semantic version, for example, the underlying engine

```
semver Location.none "1.2.3" "~1.3.0" = false;;
semver Location.none "1.2.3" "^1.3.0" = true ;;
semver Location.none "1.2.3" ">1.3.0" = false ;;
semver Location.none "1.2.3" ">=1.3.0" = false ;;
semver Location.none "1.2.3" "<1.3.0" = true ;;
semver Location.none "1.2.3" "<=1.3.0" = true ;;
semver Location.none "1.2.3" "1.2.3" = true;;
```

## Tips & Tricks

This is a very small extension to OCaml. It's backward compatible with OCaml except in the following case:

```ocaml
let f x =
  x
#elif //
```

`#elif` at the beginning of a line is interpreted as static `if`. there is no issue with `#if` or `#end`, since they are already keywords.
