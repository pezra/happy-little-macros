
# Happy little macros

Peter Williams
pezra@barelyenough.org

---

anyone can create beautifully clear code

---

an example: benchmarking

---

```elixir
before_ms = :erlang.monotonic_time(:millisecond)
:timer.sleep(:rand.uniform(100))
after_ms = :erlang.monotonic_time(:millisecond)
elapsed = after_ms - before_ms
Logger.info("important stuff (#{elapsed}ms)")
```
@[2](the actual functionality)
@[1, 3-5](boilerplate)
@[1-5]

14:25:06.869 [info] important stuff (46ms)

Note:
The boilerplate hides the intent of this block of code
---

let's have a happy little macro to clarify our intent

---

```elixir
benchmark("important stuff") do
  :timer.sleep(:rand.uniform(100))
end
```

14:25:06.869 [info] important stuff (46ms)

Note:
The intent of this version is clear.
---

```elixir
defmacro benchmark(msg, blk) do
  quote do
    before_ms = :erlang.monotonic_time(:millisecond)
    unquote(blk)
    after_ms = :erlang.monotonic_time(:millisecond)
    elapsed = after_ms - before_ms
    Logger.info(unquote(msg) <> " (#{elapsed}ms)")
  end
end
```

@[3, 5-7] the boilerplate from the previous example
@[4] execute it do block passed in

Note:
In this context all that time and logging code is the important part, *not* boilerplate.

---

how does that work?

Note:
First a little theory then we'll explore some toy examples.

---

AST = abstract syntax tree

`round(40.3 + 1.7)`

![simple AST](./simple-ast.png)

Note:
This is what parsers build.

---

Compile pipeline

read file |> parse |> convert to byte code

---

macros are functions that alter the AST during compile

---

Compile pipeline

read file |> parse |> expand macros |> convert to byte code

Note:

Basically a hook provided by the compiler.

---

the macro contract: the compiler passes the original AST; the macro returns the AST to use in it's place

Note:
Go wild! All of Elixir is available in macros.

---

tools to ease AST creation

- `quote`
- `unquote`

---

```elixir
iex>  quote do: "hello"
"hello"
```

```elixir
iex>  quote do: 42
42
```

Note:
`quote` returns the AST representation of the code you pass it. For literals it's just the literal.

---

```elixir
iex>  quote do: 3 + 1
{:+, [context: Elixir, import: Kernel], [3, 2]}
```

Note:
`quote` returns the AST representation of the code you pass it. For expression it gets a little more complicated.

---

```elixir
iex> x = 3
...> quote do: unquote(x)
3
```

Note:
`unquote` returns the AST of a value outside the quote.

---


```elixir
iex> a = 3
...> quote do: unquote(a + 2)
5
```

Note: To be more precise. `unquote` evaluates the expressions you pass in the context outside the `quote` block and returns the AST representation of that expression's return value.

----

Back to our benchmarking example

---

```elixir
benchmark("Doing important stuff") do
  :timer.sleep(:rand.uniform(100))
end
```

```elixir
defmacro benchmark(msg, blk) do
  quote do
    before_ms = :erlang.monotonic_time(:millisecond)
    unquote(blk)
    after_ms = :erlang.monotonic_time(:millisecond)
    elapsed = after_ms - before_ms
    Logger.info(unquote(msg) <> " (#{elapsed}ms)")
  end
end
```

---

```elixir
{:__block__, [],
 [{:=, [],
   [{:before_ms, [], Benchmarking},
    {{:., [], [:erlang, :monotonic_time]}, [], [:millisecond]}]},
  [do: {{:., [line: 34], [:timer, :sleep]}, [line: 34],
    [{{:., [line: 34], [:rand, :uniform]}, [line: 34], 'd'}]}],
  {:=, [],
   [{:after_ms, [], Benchmarking},
    {{:., [], [:erlang, :monotonic_time]}, [], [:millisecond]}]},
  {:=, [],
   [{:elapsed, [], Benchmarking},
    {:-, [context: Benchmarking, import: Kernel],
     [{:after_ms, [], Benchmarking},
      {:before_ms, [], Benchmarking}]}]},
  {{:., [],
    [{:__aliases__, [alias: false], [:Logger]}, :info]},
   [],
   [{:<>, [context: Benchmarking, import: Kernel],
     ["Doing important stuff",
      {:<<>>, [],
       [" (",
        {:::, [],
         [{{:., [], [Kernel, :to_string]}, [],
           [{:elapsed, [], Benchmarking}]},
          {:binary, [], Benchmarking}]}, "ms)"]}]}]}]}
```

@[2-4](before_ms = :erlang.monotonic_time(:millisecond))
@[5-6](... do :timer.sleep(:rand.uniform(100)) end)
@[7-9](after_ms = :erlang.monotonic_time(:millisecond))
@[10-14](elapsed = after_ms - before_ms)
@[15-25](Logger.info(unquote(msg) <> " (#{elapsed}ms)"))

---


