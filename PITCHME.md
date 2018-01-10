
# Happy little macros

Peter Williams  
pezra@barelyenough.org

---

```elixir
before_ms = :erlang.monotonic_time(:millisecond)
:timer.sleep(:rand.uniform(100))
after_ms = :erlang.monotonic_time(:millisecond)
elapsed = after_ms - before_ms
Logger.info("Doing important stuff (#{elapsed}ms)")
```
@[2]
@[1, 3-5](Boilerplate sucks!)

14:25:06.869 [info]  Doing important stuff (46ms)

---

Let's have a happy little macro that makes our intent clear

---

```elixir
benchmark("Doing important stuff") do
  :timer.sleep(:rand.uniform(100))
end
```

14:25:06.869 [info]  Doing important stuff (46ms)

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

@[3, 5-7]
@[4]
@[2, 8]

---

How does that work?

---

```elixir
iex> a = 3
...> quote do: unquote(a) + 1
{:+, [context: Elixir, import: Kernel], [3, 2]}
```

@[1-3](`quote` parses the code and returns its AST)
@[1-3](`unquote` returns the AST for a value outside the `quote` block)

---

Back to our benchmarking example

---

```elixir
benchmark("Doing important stuff") do
  :timer.sleep(:rand.uniform(100))
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


