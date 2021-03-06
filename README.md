# OTPCL

## What is it?

Open Telecom Platform Command Language, a.k.a. Tcl-flavored Erlang.
Or maybe it's Erlang-flavored Tcl?

## How do I use it?

For now, clone this repo, and make sure you have rebar3 installed.
Then, from the repo's root:

```
$ rebar3 compile
[ ... bunch of rebar3 output that hopefully looks successful ... ]

$ bin/otpcl
OTPCL Shell (WIP!)

otpcl> print "Hello, world!~n"
Hello, world!
ok
```

You can also use it from an existing project in some other BEAM-based
language (note that said language will need to be able to see OTPCL's
compiled libs; this happens automatically if you're doing things with
Hex like described below, but otherwise, you'll have to point to it
with the ERL_LIBS variable).

For example, in Erlang (w/ rebar3):

```
$ grep otpcl rebar.config
{deps, [{otpcl, "0.2.0"}]}.
$ rebar3 shell
Eshell V10.0  (abort with ^G)
1> otpcl:eval("import io; format {Hello, world!~n}").
Hello, world!
[ ... bunch of output because we just imported everything from
      Erlang's io module and otpcl:eval returns the full
      interpreter state when it's done executing stuff ... ]
```

And again, in Elixir (w/ Mix):

```
$ grep otpcl mix.exs
      {:otpcl, "~> 0.2.0"}
$ iex -S mix
Interactive Elixir (1.7.3) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> :otpcl.eval("import Elixir.IO; puts {Hello, world!}")
Hello, world!
[ ... bunch of output because we just imported everything from
      Elixir's IO module and :otpcl.eval returns the full
      interpreter state when it's done executing stuff ... ]
```

## What (else) can it do?

Well, as you might've guessed from above, it can parse a Tcl-like
language:

```erlang

2> otpcl:parse("foo {bar $baz {bam [bat $baf]} bal} $bad $bak$bae [bah $bay]").
{ok,{parsed,program,
        [{parsed,command,
             [{parsed,unquoted,
                  [{102,{nofile,0,0}},{111,{nofile,0,1}},{111,{nofile,0,2}}]},
              {parsed,braced,
                  [{98,{nofile,0,5}},
                   {97,{nofile,0,6}},
                   {114,{nofile,0,7}},
                   {32,{nofile,0,8}},
                   {36,{nofile,0,9}},
                   {98,{nofile,0,10}},
                   {97,{nofile,0,11}},
                   {122,{nofile,0,12}},
                   {32,{nofile,0,13}},
                   {123,{nofile,0,14}},
                   {98,{nofile,0,...}},
                   {97,{nofile,...}},
                   {109,{...}},
                   {32,...},
                   {...}|...]},
              {parsed,var_unquoted,
                  [{98,{nofile,0,37}},{97,{nofile,0,38}},{100,{nofile,0,39}}]},
              {parsed,var_unquoted,
                  [{98,{nofile,0,42}},
                   {97,{nofile,0,43}},
                   {107,{nofile,0,44}},
                   {36,{nofile,0,45}},
                   {98,{nofile,0,46}},
                   {97,{nofile,0,47}},
                   {101,{nofile,0,48}}]},
              {parsed,funcall,
                  [{parsed,unquoted,
                       [{98,{nofile,0,51}},
                        {97,{nofile,0,52}},
                        {104,{nofile,0,53}}]},
                   {parsed,var_unquoted,
                       [{98,{nofile,0,56}},
                        {97,{nofile,0,57}},
                        {121,{nofile,0,...}}]}]}]}]},
    []}

```

And it can interpret that language, too:

```erlang

3> otpcl:eval("set foo 1; set bar 2; set baz 3").
{3,
 {#{decr => fun otpcl_stdlib:decr/2,
    'if' => fun otpcl_stdlib:if/2,
    incr => fun otpcl_stdlib:incr/2,
    print => fun otpcl_stdlib:print/2,
    set => fun otpcl_stdlib:set/2,
    unless => fun otpcl_stdlib:unless/2},
  #{'RETVAL' => 3,bar => 2,baz => 3,foo => 1}}}

```

And as demonstrated above, you can do things from the OTPCL shell/REPL
(albeit with very poor error handling at the moment, alas):

```
otpcl> import math
ok
otpcl> exp 4
54.598150033144236
otpcl> exp foo
error: badarg
Stacktrace:
  math:exp/[foo]
  otpcl_stdmeta:'-import/2-fun-1-'/4
    file: "/home/ryno/Projects/otpcl/src/otpcl_stdmeta.erl"
    line: 33
  otpcl_eval:interpret/2
    file: "/home/ryno/Projects/otpcl/src/otpcl_eval.erl"
    line: 87
  otpcl_shell:eval/2
    file: "/home/ryno/Projects/otpcl/src/otpcl_shell.erl"
    line: 46
```

We can define new Erlang functions and include them as functions for
our interpreter, both from the Erlang side:

```erlang

4> Sum = fun (Nums, State) -> {lists:sum(Nums), State} end.
#Fun<erl_eval.12.127694169>
5> {ok, State} = otpcl_meta:cmd([sum, Sum], otpcl_env:default_state()).
[ ... interpreter state output ... ]
6> {RetVal, NewState} = otpcl:eval("sum 1 2 3 4 5", State).
[ ... interpreter state output ... ]
7> RetVal.
15

```

And of course, no programming language would be complete if we can't
define functions in that language:

```
otpcl> cmd howdy {$pardner} {
  ...> return <howdy $pardner>
  ...> }
ok
otpcl> howdy buckaroo
{howdy,buckaroo}
otpcl> cmd multi-test {a} {
  ...> return "It's an 'a'!"
  ...> } {1} {
  ...> return "It's a 1!"
  ...> } {$else} {
  ...> return "It's something else..."
  ...> }
ok
otpcl> multi-test a
<<"It's an 'a'!">>
otpcl> multi-test 1
<<"It's a 1!">>
otpcl> multi-test asdf
<<"It's something else...">>

```

Or, as demonstrated above, you can even import them, whether as whole
modules:

```
otpcl> import random; uniform 8675309
3848234
```

Or as individual functions:

```
otpcl> import string (split uppercase)
ok
otpcl> split [uppercase "foo,bar,baz"] ","
[<<"FOO">>,<<"BAR,BAZ">>]
```

Alternately, if you want to avoid namespace clashes:

```
otpcl> use string
ok
otpcl> string split [string uppercase "foo,bar,baz"] ","
[<<"FOO">>,<<"BAR,BAZ">>]
```

There's still a lot of work to be done, but it ain't bad for my
first-ever programming language, I'd say (and with a hand-written
parser, to boot!).

## What *should* it do?

* Tokenizer (100%)

* Parser (100%) (there are probably bugs, but it's otherwise complete)

* Interpreter (100%) (there are probably bugs, but it's otherwise complete)

* Standard library / built-in functions (50%)

* Compiler (0%)

* REPL/shell (75%) (mostly functional, and does a decent job of error
  reporting now, but plenty of room for polish)

* Tests (no idea what the test coverage is right now, but hey, at
  least I wrote (some) tests this time!)

* Docs (80%) (making it a point to document new functions as I go)

* Install procedure that's actually sane (or for that matter exists at
  all)

## What's the actual syntax?

Like with Tcl, an OTPCL program is a sequence of
vertical-whitespace-delimited commands (semicolons counting as
"vertical whitespace" in this context), each of which is a sequence of
horizontal-whitespace-delimited words (note: not all forms of
horizontal/vertical whitespace are currently recognized as such by the
parser, whereas a backslash-escaped newline *is* recognized as such).

A word may be any of the following:

* An atom (either `unquoted` or `'Single Quoted'`)
* An integer (`123` or `-123`)
* A float (`123.456` or `-123.456`)
* A binary string (either `"double quoted"` or `{curly braced}`)
* A charlist string (backquoted)
* A list (`(word-elements surrounded by parentheses)`)
* A tuple (`<word-elements surrounded by angle brackets>`)
* A variable substitution (either `$unquoted` or `${braced}`)
* A function call substitution (`[command inside square brackets]`)

There's also the concept of "pipe commands" - that is, if a word starts with a
pipe, OTPCL will treat it like a newline/semicolon and then treat it as a
command name.  The core pipe operator (`|`) behaves similarly to the pipe
operator in Elixir (`|>`); it'll feed the result of the previous command into
the first argument slot for the next command.

### Crash Course

```tcl
this is a command  # this is a comment
this is one command; this is another command
this command (accepts a list)
this command <accepts a tuple>
this command "accepts a binary string"
this command {also accepts a binary string}
this command `accepts an Erlang-style charlist string`
this command 'Accepts an atom that has spaces in it'
this command will use a $variable ${another variable} and a [function call]
this command \
    continues on the next line \
    and takes a (list that also \
    continues onto another line)
C'est | une | pipe  # Take that, Magritte!
```

## What's the license?

OpenBSD-style ISC License:

> Copyright (c) 2018, 2019 Ryan S. Northrup <northrup@yellowapple.us>

> Permission to use, copy, modify, and distribute this software for
> any purpose with or without fee is hereby granted, provided that the
> above copyright notice and this permission notice appear in all
> copies.

> THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL
> WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED
> WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE
> AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL
> DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA
> OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER
> TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR
> PERFORMANCE OF THIS SOFTWARE.
