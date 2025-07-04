## Working with Environment Variables

We’ll improve the `minigrep` binary by adding an extra feature: an option for
case-insensitive searching that the user can turn on via an environment
variable. We could make this feature a command line option and require that
users enter it each time they want it to apply, but by instead making it an
environment variable, we allow our users to set the environment variable once
and have all their searches be case insensitive in that terminal session.

### Writing a Failing Test for the Case-Insensitive `search` Function

We first add a new `search_case_insensitive` function to the `minigrep` library
that will be called when the environment variable has a value. We’ll continue
to follow the TDD process, so the first step is again to write a failing test.
We’ll add a new test for the new `search_case_insensitive` function and rename
our old test from `one_result` to `case_sensitive` to clarify the differences
between the two tests, as shown in Listing 12-20.

<Listing number="12-20" file-name="src/lib.rs" caption="Adding a new failing test for the case-insensitive function we’re about to add">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-20/src/lib.rs:here}}
```

</Listing>

Note that we’ve edited the old test’s `contents` too. We’ve added a new line
with the text `"Duct tape."` using a capital _D_ that shouldn’t match the query
`"duct"` when we’re searching in a case-sensitive manner. Changing the old test
in this way helps ensure that we don’t accidentally break the case-sensitive
search functionality that we’ve already implemented. This test should pass now
and should continue to pass as we work on the case-insensitive search.

The new test for the case-_insensitive_ search uses `"rUsT"` as its query. In
the `search_case_insensitive` function we’re about to add, the query `"rUsT"`
should match the line containing `"Rust:"` with a capital _R_ and match the
line `"Trust me."` even though both have different casing from the query. This
is our failing test, and it will fail to compile because we haven’t yet defined
the `search_case_insensitive` function. Feel free to add a skeleton
implementation that always returns an empty vector, similar to the way we did
for the `search` function in Listing 12-16 to see the test compile and fail.

### Implementing the `search_case_insensitive` Function

The `search_case_insensitive` function, shown in Listing 12-21, will be almost
the same as the `search` function. The only difference is that we’ll lowercase
the `query` and each `line` so that whatever the case of the input arguments,
they’ll be the same case when we check whether the line contains the query.

<Listing number="12-21" file-name="src/lib.rs" caption="Defining the `search_case_insensitive` function to lowercase the query and the line before comparing them">

```rust,noplayground
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-21/src/lib.rs:here}}
```

</Listing>

First we lowercase the `query` string and store it in a new variable with the
same name, shadowing the original `query`. Calling `to_lowercase` on the query
is necessary so that no matter whether the user’s query is `"rust"`, `"RUST"`,
`"Rust"`, or `"``rUsT``"`, we’ll treat the query as if it were `"rust"` and be
insensitive to the case. While `to_lowercase` will handle basic Unicode, it
won’t be 100 percent accurate. If we were writing a real application, we’d want
to do a bit more work here, but this section is about environment variables,
not Unicode, so we’ll leave it at that here.

Note that `query` is now a `String` rather than a string slice because calling
`to_lowercase` creates new data rather than referencing existing data. Say the
query is `"rUsT"`, as an example: that string slice doesn’t contain a lowercase
`u` or `t` for us to use, so we have to allocate a new `String` containing
`"rust"`. When we pass `query` as an argument to the `contains` method now, we
need to add an ampersand because the signature of `contains` is defined to take
a string slice.

Next, we add a call to `to_lowercase` on each `line` to lowercase all
characters. Now that we’ve converted `line` and `query` to lowercase, we’ll
find matches no matter what the case of the query is.

Let’s see if this implementation passes the tests:

```console
{{#include ../listings/ch12-an-io-project/listing-12-21/output.txt}}
```

Great! They passed. Now, let’s call the new `search_case_insensitive` function
from the `run` function. First we’ll add a configuration option to the `Config`
struct to switch between case-sensitive and case-insensitive search. Adding
this field will cause compiler errors because we aren’t initializing this field
anywhere yet:

<span class="filename">Filename: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-22/src/main.rs:here}}
```

We added the `ignore_case` field that holds a Boolean. Next, we need the `run`
function to check the `ignore_case` field’s value and use that to decide
whether to call the `search` function or the `search_case_insensitive`
function, as shown in Listing 12-22. This still won’t compile yet.

<Listing number="12-22" file-name="src/main.rs" caption="Calling either `search` or `search_case_insensitive` based on the value in `config.ignore_case`">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-22/src/main.rs:there}}
```

</Listing>

Finally, we need to check for the environment variable. The functions for
working with environment variables are in the `env` module in the standard
library, which is already in scope at the top of _src/main.rs_. We’ll use the
`var` function from the `env` module to check to see if any value has been set
for an environment variable named `IGNORE_CASE`, as shown in Listing 12-23.

<Listing number="12-23" file-name="src/main.rs" caption="Checking for any value in an environment variable named `IGNORE_CASE`">

```rust,ignore,noplayground
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-23/src/main.rs:here}}
```

</Listing>

Here, we create a new variable, `ignore_case`. To set its value, we call the
`env::var` function and pass it the name of the `IGNORE_CASE` environment
variable. The `env::var` function returns a `Result` that will be the
successful `Ok` variant that contains the value of the environment variable if
the environment variable is set to any value. It will return the `Err` variant
if the environment variable is not set.

We’re using the `is_ok` method on the `Result` to check whether the environment
variable is set, which means the program should do a case-insensitive search.
If the `IGNORE_CASE` environment variable isn’t set to anything, `is_ok` will
return `false` and the program will perform a case-sensitive search. We don’t
care about the _value_ of the environment variable, just whether it’s set or
unset, so we’re checking `is_ok` rather than using `unwrap`, `expect`, or any
of the other methods we’ve seen on `Result`.

We pass the value in the `ignore_case` variable to the `Config` instance so the
`run` function can read that value and decide whether to call
`search_case_insensitive` or `search`, as we implemented in Listing 12-22.

Let’s give it a try! First we’ll run our program without the environment
variable set and with the query `to`, which should match any line that contains
the word _to_ in all lowercase:

```console
{{#include ../listings/ch12-an-io-project/listing-12-23/output.txt}}
```

Looks like that still works! Now let’s run the program with `IGNORE_CASE` set
to `1` but with the same query _to_:

```console
$ IGNORE_CASE=1 cargo run -- to poem.txt
```

If you’re using PowerShell, you will need to set the environment variable and
run the program as separate commands:

```console
PS> $Env:IGNORE_CASE=1; cargo run -- to poem.txt
```

This will make `IGNORE_CASE` persist for the remainder of your shell session.
It can be unset with the `Remove-Item` cmdlet:

```console
PS> Remove-Item Env:IGNORE_CASE
```

We should get lines that contain _to_ that might have uppercase letters:

<!-- manual-regeneration
cd listings/ch12-an-io-project/listing-12-23
IGNORE_CASE=1 cargo run -- to poem.txt
can't extract because of the environment variable
-->

```console
Are you nobody, too?
How dreary to be somebody!
To tell your name the livelong day
To an admiring bog!
```

Excellent, we also got lines containing _To_! Our `minigrep` program can now do
case-insensitive searching controlled by an environment variable. Now you know
how to manage options set using either command line arguments or environment
variables.

Some programs allow arguments _and_ environment variables for the same
configuration. In those cases, the programs decide that one or the other takes
precedence. For another exercise on your own, try controlling case sensitivity
through either a command line argument or an environment variable. Decide
whether the command line argument or the environment variable should take
precedence if the program is run with one set to case sensitive and one set to
ignore case.

The `std::env` module contains many more useful features for dealing with
environment variables: check out its documentation to see what is available.
