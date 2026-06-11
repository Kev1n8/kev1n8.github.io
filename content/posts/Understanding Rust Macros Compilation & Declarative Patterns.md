+++
title = "Understanding Rust Macros: Compilation & Declarative Patterns"
date = 2025-04-01

[taxonomies]
tags = ["rust"]
+++

# Understanding Rust Macros: Compilation & Declarative Patterns

Before deciding which type of macro to write, ask yourself the questions below:
- Is this a basic find/replace (e.g. implementing the same trait for different integer types or tuple lengths) or do I need some sort of conditional logic?
- Do I need to track more "state" than a can be achieved with a simple pushdown accumulator?
- Is this purely a syntactic operation or do I need to inspect a token's value and do logic with it?

## Compiling Trip

Rust compilation goes through stages below:

1. Tokenization -- keywords
2. Parsing -- where AST constructed
3. **Macro Expansion**
4. Name Resolution and Type Checking
5. Intermediate Representation
6. Optimization
7. Code Generation -- to LLVM IR
8. LLVM Optimization
9. Machine Code Generation
10. Linking

**Tokens tree is distinct from AST**

### Macro in AST

1. `# [ $arg ];` e.g. #[derive(Clone)], #[no_mangle], …
2. `# ! [ $arg ];` e.g. #![allow(dead_code)], #![crate_name="blang"], …
3. `$name ! $arg;` e.g. println!("Hi!"), concat!("a", "b"), …
4. `$name ! $arg0 $arg1;` e.g. macro_rules! dummy { () => {}; }.

### Expansion

Intuitively, this is done by "expanding" the AST on its *syntax extensions* invocations **recursively**, and level-up no more than 128(default).

### Hygiene

Simply saying, in Rust, macro can't `use` the identifier ouside or `create` an identifier then `use` outside of it.

## Declarative Macro

`macro_rules!` checks the rules one by one, perform expansion once the `matcher` matches.
```rust
macro_rules! $name {
    $rule0 ;
    $rule1 ;
    // …
    $ruleN ;
}
$rule: ($matcher) => {$expansion}
```

`macro_rules!`'s invocation does not expand in AST, the macro is registered internally in the compiler.

Repetition example
```rust
$ ( ... ) sep rep

macro_rules! sample {
    ($($e:ident),*) => {
        $(
            // operation
        )*
    }
}
```
`sep` options: `,`, `;` or space by default
`rep` options: `*`, `+`, or `?` which can't be used with `sep` since at most 1


### Minutiae

The following labels are that we can capture from the input:
- block -- capture a code block like `{...}`
- expr -- a lot
- ident -- identifiers and keywords
- item -- definitions like `struct Foo` or `pub use crate::mod`
- lifetime -- similar to ident but start with `'`
- literal -- immediate values
- meta -- matches attributes
- pat -- any kind of pattern
- pat_param -- any pattern but no `|`
- path -- a path like `crate::mod`
- stmt -- a statement
- tt -- a token tree
- ty -- a type expression
- vis -- visibility like `pub(crate)`

### Scoping

`#[macro_use]`: `use` all the macro inside the first following mod, applicable for the whole "level".
`#[macro_export]`: export the macro to outside (of `crate`), other users may `use mod::some_macro;`(only applicable for external crate).

### TODO

There are more sections to learn. Maybe learn on practice.

## Procedural Macro

### Types

- *function-like* proc-macros -- `$name ! $arg`
- *atribute* proc-macros -- `#[$arg]`
- *derive* proc-macros -- `#[derive(...)]`


```rust
#[proc_macro]
pub fn my_proc_macro(input: TokenStream) -> TokenStream {
    TokenStream::new()
}
```


```rust
#[proc_macro_attribute]
pub fn my_attribute(input: TokenStream, annotated_item: TokenStream) -> TokenStream {
    TokenStream::new()
}
```


```rust
#[proc_macro_derive]
pub fn my_derive(annotatated_item: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

### Useful Third-party tools

- proc-macro2
- quote
- syn

