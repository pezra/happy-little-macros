
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

@[3, 5-7](the boilerplate from the previous example)
@[4](execute it do block passed in)

Note:
In this context all that time and logging code is the important part, *not* boilerplate.

---

how does that work?

---

First a little theory

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

the macro contract: the compiler passes in the original AST; the macro returns the AST to use in it's place

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

Domain Specific Languages

---

- `use`
- `Mix.Config`
- ExUnit

---

```elixir
use MixConfig

config :my_app, :go_slow, false
```
@[1](use macro)

---

```elixir
defmacro use(module, opts \\ []) do
  calls = Enum.map(expand_aliases(module, __CALLER__), fn
    expanded when is_atom(expanded) ->
      quote do
        require unquote(expanded)
        unquote(expanded).__using__(unquote(opts))
      end
    _otherwise ->
      raise ArgumentError, "invalid arguments for use, expected a compile time atom or alias, got: #{Macro.to_string(module)}"
  end)
  quote(do: (unquote_splicing calls))
end
```
@[2](expand aliases manually since we are at compile time)
@[4-7](build a AST that calls the __using__ macro of each module)
@[11](smoosh all one of those ASTs together)

---

```elixir
use MixConfig

config :my_app, :go_slow, false
```
@[3](add a config item)

---

```elixir
defmacro config(app, key, opts) do
  quote do
    Mix.Config.Agent.merge var!(config_agent, Mix.Config),
      [{unquote(app), [{unquote(key), unquote(opts)}]}]
  end
end
```
@[1]
@[2]
@[3](var!/2)

---

Hygiene

Note:
Variables in marcos and the surrounding code are in different namespaces.

---

```elixir
defmodule Hygiene do
  defmacro naive_set_x(val) do
    quote do
      x = unquote(val)
    end
  end
end
```

```elixir
iex> x = 1
iex> Hygiene.naive_set_x(42)
iex> x
1
```
@[4](x is unchanged)

---

```elixir
defmodule Hygiene do
  defmacro set_x(val) do
    quote do
      var!(x) = unquote(val)
    end
  end
end
```

```elixir
iex> x = 1
iex> Hygiene.set_x(42)
iex> x
42
```

---

Back to our config example

---

```elixir
config :my_app, :go_slow, false
```

```elixir
defmacro config(app, key, opts) do
  quote do
    Mix.Config.Agent.merge var!(config_agent, Mix.Config),
      [{unquote(app), [{unquote(key), unquote(opts)}]}]
  end
end
```
@[3](get the config_agent defined earlier)
@[4]([my_app: [go_slow: false]])
@[3](merge the new config item with the existing config)

---

Where did `config_agent` come from?

---

```elixir
use Mix.Config
```

```elixir
defmacro __using__(_) do
  quote do
    import Mix.Config, only: [config: 2, config: 3, import_config: 1]
    {:ok, agent} = Mix.Config.Agent.start_link
    var!(config_agent, Mix.Config) = agent
  end
end
```
@[4](start an agent in which to collect)
@[5](assign it to config_agent variable in the Mix.Config namespace)

---

How does `Application.get_env` get access to this?

---

```elixir
def persist(config) do
  for {app, kw} <- config do
    for {k, v} <- kw do
      Application.put_env(app, k, v, persistent: true)
    end
    app
  end
end
```

Note:
In `Mix.Config`

---