SeqBind
=======

Problem
-------

Does it bother you that some of your code looks like this?

```erlang
 L1 = lists:map(fun (X) -> ... end, L),
 L2 = lists:filter(fun (X) -> ... end, L1)
 %% or
 {Q,Req1} = cowboy_http_req:qs_val(<<"q">>,Req),
 {Id,Req2} = cowboy_http_req:qs_val(<<"id">>,Req1)
```

Solution
--------

While there are some solutions available to address this, such as giving variables more descriptive names or extracting each step into a separate function, this might not be particularly  suitable for every case. Providing descriptive names still doesn't assist with reordering and having too many function calls may add complexity and unnecessary overhead.

SeqBind offers a different, yet simple, solution to this problem.

It introduces the concept of sequential bindings. What is that? Sequential bindings are bindings that carry the suffix `@` (like `L@` or `Req@`) for example.

SeqBind is a parse transformation that auto-numbers all occurrences of these bindings following the suffix @ (creating `L@0`, `L@1`, `Req@0`, `Req@1`) and so on.

In order to use SeqBind, one should enable the `seqbind` parse transformation, this can be done either through compiler options or by adding the line below to your module:

```erlang
-compile({parse_transform,seqbind}).
```

One of the important properties of SeqBind is that it does not introduce any overhead (unlike some other, relatively similar solutions). Namely, it doesn't wrap anything into `fun`s but simply auto-numbers bindings. Effectively, your compiled code is no different from the original code structurally.

Returning to the problem definition, this is how your code will look like with SeqBind:

```erlang
 L@ = lists:map(fun (X) -> ... end, L@),
 L@ = lists:filter(fun (X) -> ... end, L@)
 %% or
 {Q,Req@} = cowboy_http_req:qs_val(<<"q">>,Req@),
 {Id,Req@} = cowboy_http_req:qs_val(<<"id">>,Req@)
```

Neat, eh?

__Please__, don't use SeqBind everywhere. It is intended to be only used in those situations that really warrant its use. _Overuse of this technique will make your code look too noisy_ (`@` does stand out) and in general should be avoided as it does obscure the functional nature of Erlang.

General Rules
---

There are few rules to be followed.

### Left and Right


In the matches (such as `A@ = 1`) the side of the match is significant to SeqBind. Even though in Erlang itself it is not, this agreement allows SeqBind to do its job.

The basic idea is that if you have a sequential binding on the left, its counter will be incremented. If it's on the right, the current counter value will be used.

As a consequence of this, one should be aware, that with the commonly used syntax:

```erlang
#state{} = State@
```

Notice that because the bound variable is on the right; it will, as a result, not increment the counter for `State@` (with the noted exception of when this syntax is used in a function clause). In order to achieve the intended result, one should do:

```erlang
State@ = #state{}
```

### Matching

If instead of incrementing a sequential binding's counter you actually just want to match its value while putting it on the left side, you simply drop the suffix like so:


```erlang
State@ = get_state(),
State = get_state_again()
```

The same goes with `case`,`if` and `receive` clauses, also note that you cannot `seqbind` a variable after it has been declared, it must come first.

Extra Goodies
-------------

### Debug helpers

SeqBind has a debug helper `seqbind:i/3` that will print out the source code for a given M:F/A (provided abstract code was not removed). 

This will help SeqBind users to match sequential bindings names to those they have in their debugger.

Example:


```erlang
1> seqbind:i(seqbind_tests, multiple_assignments_test, 0).
Line 5:
multiple_assignments_test() ->
    A@0 = 1,
    A@1 = 2,
    fun(__X) ->
           case A@1 of
               __X ->
                   ok;
               __V ->
                   .erlang:error({assertEqual_failed,
                                  [{module,seqbind_tests},
                                   {line,8},
                                   {expression,"A@"},
                                   {expected,__X},
                                   {value,__V}]})
           end
    end(2).

ok
```

### Let syntax

SeqBind also adds experimental `let` syntax in a form of a function call:

```erlang
A@ = 1,
let@(A@ = 2,
     %% here A@ is 2
    ),
%% and here A@ is 1
```