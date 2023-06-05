# Rerast

Rerast is a search/replace tool for Rust code using rules. A rule consists of a
search pattern, a replacement and possibly some placeholders that can appear in
both the search pattern and the replacement. Matching is done on syntax, not on
text, so formatting doesn't matter. Placeholders are typed and must match the
type found in the code for the rule to apply.

Rerast is deprecated. We suggest using the [Structured Search
Replace](https://rust-analyzer.github.io/manual.html#structural-search-and-replace) feature available
in rust-analyzer. It is available either in vscode or from the command line (and possibly also vim).
If you are missing any particular feature that Rerast supported (or didn't), please comment on [this
issue](https://github.com/rust-analyzer/rust-analyzer/issues/3186).

## Installation

```sh
rustup toolchain add nightly-2020-02-27
rustup component add --toolchain nightly-2020-02-27 rustc-dev
cargo +nightly-2020-02-27 install --version 0.1.88 rerast
```

## Usage

Basic operations can be performed entirely from the command line
```sh
cargo +nightly-2020-02-27 rerast --placeholders 'a: i32' --search 'a + 1' --replace_with 'a - 1' --diff
```

Alternatively you can put your rule in a Rust file
```rust
fn rule1(a: i32) {
  replace!(a + 1 => a - 1);
}
```
then use

```sh
cargo +nightly-2020-02-27 rerast --rules_file=my_rules.rs
```
Putting your rules in a file is required if you want to apply multiple rules at once.

If you'd like to actually update your files, that can be done as follows:

```sh
cargo +nightly-2020-02-27 rerast --placeholders 'a: i32' --search 'a + 1' --replace_with 'a - 1' --force --backup
```

You can control which compilation roots rerast will inject the rule into using the `--file` argument, e.g.:

```sh
cargo +nightly-2020-02-27 rerast --rules_file=my_rules.rs --targets tests --file tests/testsuite/main.rs --diff
```

Here's a more complex example

```rust
use std::rc::Rc;
fn rule1<T>(r: Rc<T>) {
  replace!(r.clone() => Rc::clone(&r))
}
```

Here we're replacing calls to the clone() method on an Rc<T> with the more explicit way of cloning
an Rc - via Rc::clone.

"r" is a placeholder which will match any expression of the type specified. The name of the function
"rule1" is not currently used for anything. In future it may be possible to selectively
enable/disable rules by specifying their name, so it's probably a good idea to put a slightly
descriptive name here. Similarly, comments placed before the function may in the future be displayed
to users when the rule matches. This is not yet implemented.

A function can contain multiple invocations of the replace! macro, with earlier rules taking precedence.
This is useful if you want to do several replacements that make use of the same placeholders or if you want
to handle certain special patterns first, ahead of a more general match.

Besides replace! there are several other replacement macros that can be used:

* replace\_pattern! - this replaces patterns. e.g. &Some(a). Such a pattern might appear in a match
  arm or if let. Irrefutable patterns (those that are guaranteed to always match) can also be
  matched within let statements and function arguments.
* replace\_type! - this replaces types. It's currently a bit limited in that it doesn't support
  placeholders. Also note, if your type is just a trait you should consider using
  replace\_trait\_ref! instead, since trait references can appear in contexts where types cannot -
  specifically generic bounds and where clauses.
* replace\_trait\_ref! - this replaces references to the named trait

Replacing statements is currently disabled pending a good use-case.

## Matching macro invocations

Macro invocations can be matched so long as they expand to code that can be matched. Note however
that a macro invocation will not match against the equivalent code, nor the invocation of a
different, but identical macro. This is intentional. When verifying a match, we check that the same
sequence of expansions was followed. Also note, that if a macro expands to something different every
time it is invoked, it will never match. println! is an example of such a macro, since it generates
a constant that is referenced from the expanded code and every invocation references a different
constant.

## Order of operations

Suppose you're replacing foo(a, b) with a && !b. Depending on what the placeholders end up matching
and what context the entire expression is in, there may be need for extra parenthesis. For example
if the matched code was !foo(x == 1, y == 2), if we didn't add any parenthesis, we'd end up with !x
== 1 && !y == 2 which clearly isn't correct. Rerast detects this and adds parenthesis as needed in
order to preserve the order or precedence found in the replacement. This would give !(x == 1 && !(y
== 2)).

## Formatting of code

No reformatting of code is currently done. Unmatched code will not be affected. Replacement code is
produced by copying the replacement code from the rule and splicing in any matched patterns. In
future, we may adjust identation for multi-line replacements. Running rustfmt afterwards is probably
a good idea since some identation and line lengths may not be ideal.

## Recursive and overlapping matches

The first matched rule wins. When some code is matched, no later rules will be applied to that
code. However, code matched to placeholders will be searched for further matches to all rules.

## Automatically determining a rule from a source code change

If you're about to make a change multiple times throughout your source code and you're using git,
you can commit (or stage) your changes, make one edit then run:

```sh
cargo +nightly-2020-02-27 rerast --replay_git --diff
```

This will locate the changed expression in your project (of which there should be only one) then try
to determine a rule that would have produced this change. It will print the rule, then apply it to
your project. If you are happy with the changes, you can run again with --force to apply them, or
you could copy the printed rule into a .rs file and apply it with --rules_file.

* The rule produced will use placeholders to the maximum extent possible. i.e. wherever a
  subexpression is found in both the old and the new code, it will be replaced with a placeholder.
* This only works for changed expressions at the moment, not for statements, types, patterns etc.
* Your code must be able to compile both with and without the change.

## Limitations

* Use statements are not yet updated, so depending on your rule, may need to be updated after the
  rule is applied. This should eventually be fixed, there just wasn't time before release and it's
  kind of tricky.
* Your code must be able to compile for this to work.
* The replacement code must also compile.  This means rerast is better at replacing a deprecated API
  usage with its non-deprecated equivalent than dealing with breaking changes.  Often the best
  workaround is to create a new API temporarily.
* Code within rustdoc is not yet processed and matched.
* Conditional code that disabled with a cfg attribute isn't matched. It's suggested to enable all
  features if possible when running so that as much code can be checked as possible.
* replace_type! doesn't yet support placeholders.
* Probably many bugs and missing features. Please feel free to file bugs / feature requests.

## Known issues

* If you have integration tests (a "tests" directory) in your project, you might
  be no matches. Not sure why. This started from nightly-2020-04-10. You might
  be able to work around this issue by passing `--targets ''` to cargo rerast.
  Unfortunately then you won't get matches in non-integration tests (i.e.
  cfg(test)). Alternatively you could install an older version of rust and the
  corresponding rerast version.
* Using `?` in the replacement is currently broken. This broke I think in
  February 2020. Something changed with spans.

## More examples
See the [Rerast Cookbook](COOKBOOK.md) for more examples.

## Groups
* [Users group](https://groups.google.com/forum/#!forum/rerast-users)
* [Developers group](https://groups.google.com/forum/#!forum/rerast-dev)

## Questions?
Feel free to just file an issue on github.

## Authors

See Cargo.toml

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## Code of conduct

This project defers to the [Rust code of conduct](https://www.rust-lang.org/en-US/conduct.html). If
you feel someone is not adhering to the code of conduct in relation to this project, please contact
David Lattimore. My email address is in Cargo.toml.

## Disclaimer

This is not an official Google product. It's released by Google only because the (original) author
happens to work there.
