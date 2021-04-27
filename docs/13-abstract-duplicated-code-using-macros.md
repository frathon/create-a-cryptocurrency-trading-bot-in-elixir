# Abstract duplicated supervision code

## Objectives
- overview of requirements
- pseudo generalize `Core.ServiceSupervisor` module
- utilize pseudo generalized code inside the `Naive.DynamicSymbolSupervisor`
- implement a truly generic `Core.ServiceSupervisor`
- use the `Core.ServiceSupervisor` module inside the `streamer` application

## Overview of requirements

In the last few chapters, we went through adding and modifying the dynamic supervision tree around the `naive` and `streamer` applications' workers. Initially, we just copied the implementation from the `streamer` application to the `naive` application (with a few minor tweaks like log messages). That wasn't the most sophisticated solution and we will address this copy-paste pattern in this chapter.

We will write an "extension" of the `DynamicSupervisor` that allows to start, stop and autostart workers. Just to keep things simple we will create a new application inside our umbrella project where we will place the logic. This will save us from creating a new repo for time being.

Our new `Core.ServiceSupervisor` module will hold all the logic responsible for starting, stopping, and autostarting worker processes. To limit the boilerplate inside the implementation modules (like `Naive.DynamicSymbolSupervisor` or `Streamer.DynamicStreamerSupervisor`) we will utilize the `use` macro that will dynamically generate low-level wiring for us.

## Pseudo generalize Core.ServiceSupervisor module

Let's start by creating a new *non supervised* application called `core` inside our umbrella project. At this moment our "abstraction code" will sit inside it just to keep things simple as otherwise, we would need to create a new repo and jump between codebases which we will avoid for time being:


```bash
$ cd apps
$ mix new core
* creating README.md
* creating .formatter.exs
* creating .gitignore
* creating mix.exs
* creating lib
* creating lib/core.ex
* creating test
* creating test/test_helper.exs
* creating test/core_test.exs
...
```

We can now create a new directory called `core` inside the `apps/core/lib` directory and a new file called `service_supervisor.ex` inside it where we will put all abstracted starting/stopping/autostarting logic.

Let's start with an empty module:


```elixir
# /apps/core/lib/core/service_supervisor.ex
defmodule Core.ServiceSupervisor do

end
```

The first step in our refactoring process will be to move(cut) all of the functions from the `Naive.DynamicSymbolSupervisor` (excluding the `start_link/1`, `init/1` and `shutdown_trading/1`) and put them inside the `Core.ServiceSupervisor` module which should look as follows:


```elixir
# /apps/core/lib/core/service_supervisor.ex
defmodule Core.ServiceSupervisor do

  def autostart_trading() do
    ...
  end

  def start_trading(symbol) when is_binary(symbol) do
    ...
  end

  def stop_trading(symbol) when is_binary(symbol) do
    ...
  end

  defp get_pid(symbol) do
    ...
  end

  defp update_trading_status(symbol, status)
       when is_binary(symbol) and is_binary(status) do
    ...
  end

  defp start_symbol_supervisor(symbol) do
    ...
  end

  defp fetch_symbols_to_trade() do
    ...
  end
end
```

All of the above code is `trading` related - we need to rename functions/logs to be more generic.

Starting with `autostart_trading/0` we can rename it to `autostart_workers/0`:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  ...
  def autostart_workers() do     # <= updated function name
    fetch_symbols_to_start()     # <= updated function name
    |> Enum.map(&start_worker/1) # <= updated function name
  end
  ...
```

As we updated two functions inside the `autostart_workers/0` we need to update their implementations.

The `start_trading/1` will become `start_worker/1`, internally we will inline the `start_symbol_supervisor/1` function(move it's contents inside the `start_worker/1` function and remove the `start_symbol_supervisor/1` function) as it's used just once inside this module as well as `update_trading_status/2` need to be renamed to `update_status/2`.

The `fetch_symbols_to_trade/0` will get updated to `fetch_symbols_to_start/0`:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def start_worker(symbol) when is_binary(symbol) do # <= updated name
    case get_pid(symbol) do
      nil ->
        Logger.info("Starting trading of #{symbol}")
        {:ok, _settings} = update_status(symbol, "on") # <= updated name

        {:ok, _pid} =
          DynamicSupervisor.start_child(
            Naive.DynamicSymbolSupervisor,
            {Naive.SymbolSupervisor, symbol}
          )  # ^^^^^^ inlined `start_symbol_supervisor/1`

      pid ->
        Logger.warn("Trading on #{symbol} already started")
        {:ok, _settings} = update_status(symbol, "on") # <= updated name
        {:ok, pid}
    end
  end

  ...

  defp fetch_symbols_to_start() do # <= updated name
    ...
  end  
```

Inside the above code we updated the `update_trading_status/2` call to `update_status/2` so we need to update the function header to match:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  defp update_status(symbol, status) # <= updated name
      when is_binary(symbol) and is_binary(status) do
     ...
  end
```

Last function to rename in this module will be the `stop_trading/1` to `stop_worker/1`, we also need to update calls to `update_trading_status/2` to `update_status/2` as it was renamed:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def stop_worker(symbol) when is_binary(symbol) do # <= updated name
    case get_pid(symbol) do
      nil ->
        Logger.warn("Trading on #{symbol} already stopped")
        {:ok, _settings} = update_status(symbol, "off") # <= updated name

      pid ->
        Logger.info("Stopping trading of #{symbol}")

        :ok =
          DynamicSupervisor.terminate_child(
            Naive.DynamicSymbolSupervisor,
            pid
          )

        {:ok, _settings} = update_status(symbol, "off") # <= updated name
    end
  end
```

At this moment we have a pseudo-generic implementation of `start_worker/1` and `stop_worker/1` inside the `Core.ServiceSupervisor` module. Function names are generic but they still refer to `Repo`, `Settings`, and other modules specific to the `naive` app's implementation. We are probably in a worse situation than we have been before starting this refactoring ;) but don't fear this was just the first step on the way to abstract away that starting, stopping and autostarting code.

## Utilize pseudo generalized code inside the `Naive.DynamicSymbolSupervisor`

Before we will jump back to the `naive`'s application modules we need to add the `core` application the dependencies of the `naive` application:


```elixir
  # /apps/naive/mix.exs
  defp deps do
    [
      {:binance, "~> 0.7.1"},
      {:binance_mock, in_umbrella: true},
      {:core, in_umbrella: true}, # <= core dep added 
      ....
```

Let's get back to the `Naive.DynamicSymbolSupervisor` where we expect functions that we just cut out to exist like `start_trading/1` or `stop_trading/1`.

Let's reimplement more generic versions of those functions as just simple calls to the `Core.ServiceSupervisor` module:


```elixir
  # /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
  ...
  def autostart_workers() do
    Core.ServiceSupervisor.autostart_workers()
  end

  def start_worker(symbol) do
    Core.ServiceSupervisor.start_worker(symbol)
  end

  def stop_worker(symbol) do
    Core.ServiceSupervisor.stop_worker(symbol)
  end
```

We also need to update the `shutdown_trading/1` function as we removed all the private functions that it relies on:


```elixir
  # /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
  def shutdown_worker(symbol) when is_binary(symbol) do # <= updated name
    case Core.ServiceSupervisor.get_pid(symbol) do # <= module added
      nil ->
        Logger.warn("Trading on #{symbol} already stopped")
        {:ok, _settings} = Core.ServiceSupervisor.update_status(symbol, "off") # <= updated name + module

      _pid ->
        Logger.info("Shutdown of trading on #{symbol} initialized")
        {:ok, settings} = Core.ServiceSupervisor.update_status(symbol, "shutdown") # <= updated name + module
        Naive.Leader.notify(:settings_updated, settings)
        {:ok, settings}
    end
  end
```

As we were moving all the private helper functions we didn't make them public so the `Naive.DynamicSymbolSupervisor` module can use them - we will fix that now together with temporary aliases/require/imports at the top of the `Core.ServiceSupervisor`:


```elixir
# /apps/core/lib/core/service_supervisor.ex
defmodule Core.ServiceSupervisor do

  require Logger                     # <= added require

  import Ecto.Query, only: [from: 2] # <= added import

  alias Naive.Repo                   # <= added alias
  alias Naive.Schema.Settings        # <= added alias

  ...

  def get_pid(symbol) do # <= updated from private to public
    ...
  end

  def update_status(symbol, status) # <= updated from private to public
      when is_binary(symbol) and is_binary(status) do
    ...
  end
```

As `fetch_symbols_to_start/0` is only used internally by the `Core.ServiceSupervisor` module itself, we don't need to make it public.

We can also remove the `alias`es and `import` from the `Naive.DynamicSymbolSupervisor` as it won't need them anymore.

Next step will be to add `ecto` to the deps of the `core` application as it will make db queries now:


```elixir
  # /apps/core/mix.exs
  defp deps do
    [
      {:ecto_sql, "~> 3.0"}
    ]
  end
```

As we modified the interface of the `Naive.DynamicSymbolSupervisor` (for example renamed `start_trading/1` to `start_worker/1` and others) we need to modify the `Naive.Supervisor`'s children list - more specifically the Task process:


```elixir
# /apps/naive/lib/naive/supervisor.ex
      ...
      {Task,
       fn ->
         Naive.DynamicSymbolSupervisor.autostart_workers() # <= func name updated
       end}
       ...
```

Last step will be to update interface of the `naive` application:


```elixir
# /apps/naive/lib/naive.ex
  alias Naive.DynamicSymbolSupervisor

  def start_trading(symbol) do
    symbol
    |> String.upcase()
    |> DynamicSymbolSupervisor.start_worker()
  end

  def stop_trading(symbol) do
    symbol
    |> String.upcase()
    |> DynamicSymbolSupervisor.stop_worker()
  end

  def shutdown_trading(symbol) do
    symbol
    |> String.upcase()
    |> DynamicSymbolSupervisor.shutdown_worker()
  end
```

Believe it or not, but at this moment(ignoring all of the warnings because we created a circular dependency between the `core` and the `naive` applications - which we will fix in the next steps) our application *runs* just fine! We are able to start and stop trading, autostarting works as well.

## Implement a truly generic `Core.ServiceSupervisor`

Ok. Why did we even do this? What we are aiming for is a separation between the interface of our `Naive.DynamicSymbolSupervisor` module (like `start_worker/1`, `autostart_workers/0` and `stop_worker/1`) and the implementation which is now placed inside the `Core.ServiceSupervisor` module.

That's all nice and to-some-extent understandable but `Core.ServiceSupervisor` module is *not* a generic module. We can't use it inside the `streaming` application to supervise the `Streamer.Binance` processes.

So, what's the point? Well, we can make it even more generic!

### First path starting with the `fetch_symbols_to_start/0` function

Moving on to full generalization of the `Core.ServiceSupervisor` module. We will start with the helper functions first as they are the ones doing the work and they need to be truly generalized first:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def fetch_symbols_to_start() do
    Repo.all(
      from(s in Settings,
        where: s.status == "on",
        select: s.symbol
      )
    )
  end
```

The `fetch_symbols_to_start/0` function uses `Repo` and `Settings` that are aliased at the top of the `Core.ServiceSupervisor` module. This just won't work with any other applications as Streamer will require its own `Repo` and `Settings` modules etc.

To fix that we will *pass* both `repo` and `schema` as arguments to the `fetch_symbols_to_start/0` function which will become `fetch_symbols_to_start/2`:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def fetch_symbols_to_start(repo, schema) do # <= args added
    repo.all( # <= lowercase `repo` is an argument not aliased module
      from(s in schema, # <= settings schema module passed as arg
        where: s.status == "on",
        select: s.symbol
      )
    )
  end
```

This will have a knock-on effect on any functions that are using `fetch_symbols_to_start/0` - now they need to use `fetch_symbols_to_start/2` and pass appropriate `Repo` and `Schema` modules.

So, the `fetch_symbols_to_start/0` is referenced by the `autostart_workers/0` - we will need to modify it to pass the repo and schema to the `fetch_symbols_to_start/2` and as it's inside the `Core.ServiceSupervisor` module it needs to get them passed as arguments:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def autostart_workers(repo, schema) do # <= args added
    fetch_symbols_to_start(repo, schema) # <= args passed
    |> Enum.map(&start_worker/1)
  end
```

Going even further down the line, `autostart_workers/0` is referenced by the `autostart_workers/0` inside the `Naive.DynamicSymbolSupervisor` module. As this module is (`naive`) application-specific, it is a place where `repo` and `schema` are known from the context - for the `naive` application `repo` is the `Naive.Repo` module and `schema` is the `Naive.Schema.Settings` module:


```elixir
  # /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
  ...
  def autostart_workers() do
    Core.ServiceSupervisor.autostart_workers(
      Naive.Repo,           # <= new value passed
      Naive.Schema.Settings # <= new value passed
    )
  end
```

This finishes the first of multiple paths that we need to follow to fully refactor the `Core.ServiceSupervisor` module.

### Second path starting with the `update_status/2`

Let's don't waste time and start from the other helper function inside the `Core.ServiceSupervisor` module. This time we will make the `update_status/2` function fully generic:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def update_status(symbol, status, repo, schema) # <= args added
      when is_binary(symbol) and is_binary(status) do
    repo.get_by(schema, symbol: symbol) # <= using dynamic repo and shcema modules
    |> Ecto.Changeset.change(%{status: status})
    |> repo.update() # <= using dynamic repo module
  end
```

As previously we added `repo` and `schema` as arguments and modified the body of the function to utilize them instead of hardcoded modules (aliased at the top of the `Core.ServiceSupervisor` module).

In the same fashion as previously, we need to check "who" is using the `update_status/2` and update those calls to `update_status/4`. The function is used inside the `start_worker/1` and the `stop_worker/1` inside the `Core.ServiceSupervisor` module so as previously we need to bubble them up(pass via arguments to both `start_worker/1` and `stop_worker/1` functions):


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def start_worker(symbol, repo, schema) when is_binary(symbol) do # <= new args
    ...
        {:ok, _settings} = update_status(symbol, "on", repo, schema) # <= args passed
    ...
        {:ok, _settings} = update_status(symbol, "on", repo, schema) # <= args passed
    ...
  end

  def stop_worker(symbol, repo, schema) when is_binary(symbol) do # <= new args
    ...
        {:ok, _settings} = update_status(symbol, "off", repo, schema) # <= args passed
    ...
        {:ok, _settings} = update_status(symbol, "off", repo, schema) # <= args passed
    ...
  end
```

As we modified both `start_worker/1` and `stop_worker/1` by adding two additional arguments we need to update all references to them and here is where things branch out a bit.

We will start with `start_worker/1` function (which is now `start_worker/3`) - it's used by the `autostart_workers/2` inside `Core.ServiceSupervisor` module. The `autostart_workers/2` function already has `repo` and `schema` so we can just pass them to the `start_worker/3`:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def autostart_workers(repo, schema) do
    fetch_symbols_to_start(repo, schema)
    |> Enum.map(&start_worker(&1, repo, schema)) # <= args passed
  end
```

Both the `start_worker/3` and the `stop_worker/3` function are used by the functions inside the `Naive.DynamicSymbolSupervisor` module. We need to pass the `Repo` and `Schema` in the same fashion as previously with `autostart_workers/2` function:


```elixir
  # /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
  def start_worker(symbol) do
    Core.ServiceSupervisor.start_worker(
      symbol,
      Naive.Repo,            # <= new arg passed
      Naive.Schema.Settings  # <= new arg passed
    )
  end

  def stop_worker(symbol) do
    Core.ServiceSupervisor.stop_worker(
      symbol,
      Naive.Repo,           # <= new arg passed
      Naive.Schema.Settings # <= new arg passed
    )
  end
```

At this moment there's no code inside the `Core.ServiceSupervisor` module referencing the `alias`ed `Repo` nor `Schema` modules so we can safely remove both aliases - definitely, we are moving in the right direction!

Btw. Our project still works at this stage, we can start/stop trading and it autostarts trading.

### Third path starting with the `get_pid/1` function

Starting again from the most nested helper function - this time the `get_pid/1`:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def get_pid(symbol) do
    Process.whereis(:"Elixir.Naive.SymbolSupervisor-#{symbol}")
  end  
```

We can see that it has a hardcoded `Naive.SymbolSupervisor` worker module - we need to make this part dynamic by using the `worker_module` argument:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def get_pid(worker_module, symbol) do  # <= arg added
    Process.whereis(:"#{worker_module}-#{symbol}") # <= arg used
  end
```

Moving up to functions that are referencing the `get_pid/1` function, those will be the `start_worker/3` and the `stop_worker/3` function. 

As those are the two last functions to be updated, we will look into them more closely to finish our refactoring in this 3rd run. At this moment both need to add `worker_module` as both are calling the `get_pid/2` function. Looking at both function we can see two other hardcoded details:
- inside log message there are words "trading" - we can replace them so we will utilize the `worker_module` and `symbol` arguments
- there are two references to the `Naive.DynamicSymbolSupervisor` which we will replace with the `module` argument
- there is one more reference to the `Naive.SymbolSupervisor` module which we will replace with the `worker_module` argument

Let's look at updated functions:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  # module and worker_module args added vvvv
  def start_worker(symbol, repo, schema, module, worker_module)
      when is_binary(symbol) do
    case get_pid(worker_module, symbol) do # <= worker_module passed
      nil ->
        Logger.info("Starting #{worker_module} worker for #{symbol}") # <= dynamic text
        {:ok, _settings} = update_status(symbol, "on", repo, schema)
        {:ok, _pid} = DynamicSupervisor.start_child(module, {worker_module, symbol}) # <= args used

      pid ->
        Logger.warn("#{worker_module} worker for #{symbol} already started") # <= dynamic text
        {:ok, _settings} = update_status(symbol, "on", repo, schema)
        {:ok, pid}
    end
  end

  # module and worker_module added as args vvvv
  def stop_worker(symbol, repo, schema, module, worker_module)
      when is_binary(symbol) do
    case get_pid(worker_module, symbol) do # <= worker_module passed
      nil ->
        Logger.warn("#{worker_module} worker for #{symbol} already stopped") # <= dynamic text
        {:ok, _settings} = update_status(symbol, "off", repo, schema)

      pid ->
        Logger.info("Stopping #{worker_module} worker for #{symbol}") # <= dynamic text
        :ok = DynamicSupervisor.terminate_child(module, pid) # <= arg used
        {:ok, _settings} = update_status(symbol, "off", repo, schema)
    end
  end
```

Inside both the `start_worker/5` and the `stop_worker/5` functions we modified:
- `get_pid/1` to pass the `worker_module`
- `Logger`'s messages to use the `worker_module` and `symbol`
- `DynamicSupervisor`'s functions to use the `module` and the `worker_module`

Again, as we modified `start_worker/5` we need to make the last change inside the `Core.ServiceSupervisor` module - `autostart_workers/2` uses the `start_worker/5` function:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  def autostart_workers(repo, schema, module, worker_module) do # <= args added
    fetch_symbols_to_start(repo, schema)
    |> Enum.map(&start_worker(&1, repo, schema, module, worker_module)) # <= args added
  end
```

Just for referrence - final function headers look as following:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
defmodule Core.ServiceSupervisor do
  def autostart_workers(repo, schema, module, worker_module) do
    ...
  end

  def start_worker(symbol, repo, schema, module, worker_module)
      when is_binary(symbol) do
    ...
  end

  def stop_worker(symbol, repo, schema, module, worker_module)
      when is_binary(symbol) do
    ...
  end

  def get_pid(worker_module, symbol) do
    ...
  end

  def update_status(symbol, status, repo, schema)
      when is_binary(symbol) and is_binary(status) do
    ...
  end

  def fetch_symbols_to_start(repo, schema) do
    ...
  end
end
```

That finishes the 3rd round of updates inside the `Core.ServiceSupervisor` module, now we need to update the `Naive.DynamicSymbolSupervisor` module to use updated functions(and pass required arguments):


```elixir
  # /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
  ...
  def autostart_workers() do
    Core.ServiceSupervisor.autostart_workers(
      Naive.Repo,
      Naive.Schema.Settings,
      __MODULE__,             # <= added arg
      Naive.SymbolSupervisor  # <= added arg
    )
  end

  def start_worker(symbol) do
    Core.ServiceSupervisor.start_worker(
      symbol,
      Naive.Repo,
      Naive.Schema.Settings,
      __MODULE__,             # <= added arg
      Naive.SymbolSupervisor  # <= added arg
    )
  end

  def stop_worker(symbol) do
    Core.ServiceSupervisor.stop_worker(
      symbol,
      Naive.Repo,
      Naive.Schema.Settings,
      __MODULE__,             # <= added arg
      Naive.SymbolSupervisor  # <= added arg
    )
  end

  def shutdown_worker(symbol) when is_binary(symbol) do
    case Core.ServiceSupervisor.get_pid(Naive.SymbolSupervisor, symbol) do # <= arg added
      nil ->
        Logger.warn("#{Naive.SymbolSupervisor} worker for #{symbol} already stopped") # <= updated

        {:ok, _settings} =
          Core.ServiceSupervisor.update_status(symbol, "off", Naive.Repo, Naive.Schema.Settings) # <= args added

      _pid ->
        Logger.info("Initializing shutdown of #{Naive.SymbolSupervisor} worker for #{symbol}") # <= updated

        {:ok, settings} =
          Core.ServiceSupervisor.update_status(
            symbol,
            "shutdown",
            Naive.Repo,
            Naive.Schema.Settings
          ) # ^^^ additional args passed

        Naive.Leader.notify(:settings_updated, settings)
        {:ok, settings}
    end
  end
```

We needed to update referrences inside the `shutdown_trading/1` function as well, as it calls `get_pid/2` and `update_status/4` functions.

We are now done with refactoring the `Core.ServiceSupervisor` module, it's completely generic and can be used inside both `streamer` and the `naive` applications.

At this moment to use the `Core.ServiceSupervisor` module we need to write interface functions in our supervisors and pass multiple arguments in each one - again, would need to use of copies those functions inside `streamer` and `naive` application. In the next section we will look into how could we leverage Elixir [macros](https://elixir-lang.org/getting-started/meta/macros.html) to remove that boilerplate.


## Remove boilerplate using `use` macro

Elixir provides a way to `use` other modules. Idea is that inside the `Naive.DynamicSymbolSupervisor` module we are `use`ing the `DynamicSupervisor` module currently but we could use `Core.ServiceSupervisor`:


```elixir
# /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
defmodule Naive.DynamicSymbolSupervisor do
  use Core.ServiceSupervisor
```

To be able to `use` the `Core.ServiceSupervisor` module it needs to provide the `__using__/1` macro. As the simplest content of that macro, we can use here would be just to use the `DynamicSupervisor` inside:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  defmacro __using__(opts) do
    IO.inspect(opts)
    quote location: :keep do
      use DynamicSupervisor
    end
  end
```

How does this work? As an oversimplification, you can think about it as Elixir will look through the contents of `quote`'s body(everything between `quote ... do` and `end`) in search for the `unquote` function which can inject dynamic content. All of this will become much clearer as we will go through the first example but the important part is that after executing any potential `unquote`s inside the `quote`'s body, Elixir will grab that code as it would be just part of code and place it inside the `Naive.DynamicSymbolSupervisor` module at compile time(we will also see the result of `IO.inspect/1` at compilation).

At this moment(after swapping to `use Core.ServiceSupervisor`) our code still works and it's exactly as we would simply have `use DynamicSupervisor` inside the `Naive.DynamicSymbolSupervisor` module - as at compilation, it will be swapped to it either way(as per contents of the `__using__/1` macro).

As the `autostart_workers/0` function is a part of the boilerplate, we will move it from the `Naive.DynamicSymbolSupervisor` module to the `Core.ServiceSupervisor` module inside the `__using__/1` macro. 

Ok, but it has all of those other `naive` application-specific arguments - where will we get those?

That's what that `opts` argument is for inside the `__using__/1` macro. When we call `use Core.ServiceSupervisor` we can pass an additional keyword list which will contain all `naive` application-specific details:


```elixir
# /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
defmodule Naive.DynamicSymbolSupervisor do
  use Core.ServiceSupervisor,
    repo: Naive.Repo,
    schema: Naive.Schema.Settings,
    module: __MODULE__,
    worker_module: Naive.SymbolSupervisor
```

We can now update the `__using__/1` macro to assign all of those details to variables(instead of using `IO.inspect/1`):


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  defmacro __using__(opts) do
    {:ok, repo} = Keyword.fetch(opts, :repo)
    {:ok, schema} = Keyword.fetch(opts, :schema)
    {:ok, module} = Keyword.fetch(opts, :module)
    {:ok, worker_module} = Keyword.fetch(opts, :worker_module)
    ...
```

At this moment we can use those *dynamic* values to generate code that will be specific to the implementation module for example `autostart_workers/0` that we moved from the `Naive.DynamicSymbolSupervisor` module and will need to have different values passed to it(like `Streamer.Binance` as `worker_module`) for the `streamer` application. We can see that it requires inserting those dynamic values inside the `autstart_workers/0` but how to dynamically inject arguments - `unquote` to the rescue. When we will update the `autostart_workers/0` function from:


```elixir
      # sample moved code from the `Naive.DynamicSymbolSupervisor` module
      def autostart_workers() do
          Core.ServiceSupervisor.autostart_workers(
          Naive.Repo,
          Naive.Schema.Settings,
          __MODULE__,
          Naive.SymbolSupervisor
        )
      end
```

to:


```elixir
      # updated code that will become part of the `__using__/1` macro
      def autostart_workers() do
        Core.ServiceSupervisor.autostart_workers(
          unquote(repo),
          unquote(schema),
          unquote(module),
          unquote(worker_module)
        )
      end
```

in the end generated code that will be "pasted" to the `Naive.DynamicSymbolSupervisor` module at compile time will be:


```elixir
    # compiled code attached to the `Naive.DynamicSymbolSupervisor` module
    def autostart_workers() do
      Core.ServiceSupervisor.autostart_workers(
        Naive.Repo,
        Naive.Schema.Settings,
        Naive.DynamicSymbolSupervisor,
        Naive.SymbolSupervisor
      )
    end
```

This way we can dynamically create functions for any application(for the `streamer` application, it will generate function call with the `Streamer.Repo`, `Streamer.Schema.Settings` args etc).

We can apply that to all of the passed variables inside the `autostart_workers/0` function - just for reference full macro will look as follows:



```elixir
  # /apps/core/lib/core/service_supervisor.ex
  defmacro __using__(opts) do
    {:ok, repo} = Keyword.fetch(opts, :repo)
    {:ok, schema} = Keyword.fetch(opts, :schema)
    {:ok, module} = Keyword.fetch(opts, :module)
    {:ok, worker_module} = Keyword.fetch(opts, :worker_module)

    quote location: :keep do
      use DynamicSupervisor

      def autostart_workers() do
        Core.ServiceSupervisor.autostart_workers(
          unquote(repo),
          unquote(schema),
          unquote(module),
          unquote(worker_module)
        )
      end
    end
  end
```

You can think about the above macro that it will substitute the `unquote(..)` parts with passed values and then it will grab the whole contents between `quote ... do` and `end` and it will paste it to the `Naive.DynamicSymbolSupervisor` module at compile-time - we can visualize generated/"pasted" code as:


```elixir
  # code generated by the `__using__/1` macro that
  # will be "pasted" to the `Naive.DynamicSymbolSupervisor` module
  use DynamicSupervisor

  def autostart_workers() do
    Core.ServiceSupervisor.autostart_workers(
      Naive.Repo,
      Naive.Schema.Settings,
      Naive.DynamicSymbolSupervisor,
      Naive.SymbolSupervisor
    )
  end
```

This is exactly the code that we had before inside the `Naive.DynamicSymbolSupervisor` module but now it's stored away inside the `Core.ServiceSupervisor`'s `__using__/1` macro and it doesn't need to be implemented/copied across twice into two apps anymore.

We can now follow the same principle and move `start_worker/1` and `stop_worker/1` from the `Naive.DynamicSymbolSupervisor` module into `__using__/1` macro inside the `Core.ServiceSupervisor` module:


```elixir
      # /apps/core/lib/core/service_supervisor.ex
      # inside the __using__/1 macro
      ...
      def start_worker(symbol) do
        Core.ServiceSupervisor.start_worker(
          symbol, # <= this needs to stay as variable
          unquote(repo),
          unquote(schema),
          unquote(module), 
          unquote(worker_module)
        )
      end

      def stop_worker(symbol) do
        Core.ServiceSupervisor.stop_worker(
          symbol, # <= this needs to stay as variable
          unquote(repo),
          unquote(schema),
          unquote(module),
          unquote(worker_module)
        )
      end
```

Here we have an example of an execution time variable called `symbol` that we should *not* `unquote` as it will be different per function call (source code should have `symbol` variable there not for example `"NEOUSDT"`).

At this moment the `Naive.DynamicSymbolSupervisor` consists of only `start_link/1`, `init/1` and `shutdown_worker/1`, it's under 50 lines of code and works exactly as before refactoring. All of the boilerplate was moved to the `Core.ServiceSupervisor` module.

We left the `shutdown_worker/1` function as it's specific to the `naive` application, but inside it, we utilize both the `get_pid/2` and the `update_status/4` functions where we are passing the `naive` application-specific variables(like `Naive.Repo`).

To make things even nicer we can create convenience wrappers for those two functions inside the `__using__/1` macro:


```elixir
      # /apps/core/lib/core/service_supervisor.ex
      # add below at the end of `quote` block inside `__using__/1`
      defp get_pid(symbol) do
        Core.ServiceSupervisor.get_pid(
          unquote(worker_module),
          symbol
        )
      end

      defp update_status(symbol, status) do
        Core.ServiceSupervisor.update_status(
          symbol,
          status,
          unquote(repo),
          unquote(schema)
        )
      end
```

As those will get compiled and "pasted" into the `Naive.DynamicSymbolSupervisor` module we can utilize them inside the `shutdown_worker/1` function as they would be much simpler naive application-specific local functions:


```elixir
  # /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
  def shutdown_worker(symbol) when is_binary(symbol) do
    case get_pid(symbol) do # <= macro provided function
      nil ->
        Logger.warn("#{Naive.SymbolSupervisor} worker for #{symbol} already stopped")
        {:ok, _settings} = update_status(symbol, "off") # <= macro provided function

      _pid ->
        Logger.info("Initializing shutdown of #{Naive.SymbolSupervisor} worker for #{symbol}")
        {:ok, settings} = update_status(symbol, "shutdown") # <= macro provided function
        Naive.Leader.notify(:settings_updated, settings)
        {:ok, settings}
    end
  end
```

And now, a very last change - I promise ;) Both the `start_link/1` and the `init/1` functions are still referencing the `DynamicSupervisor` module which could be a little bit confusing - let's swap those calls to use the `Core.ServiceSupervisor` module (both to not confuse people and be consistent with the `use` macro):


```elixir
  # /apps/naive/lib/naive/dynamic_symbol_supervisor.ex
  def start_link(init_arg) do
    Core.ServiceSupervisor.start_link(__MODULE__, init_arg, name: __MODULE__)
  end

  def init(_init_arg) do
    Core.ServiceSupervisor.init(strategy: :one_for_one)
  end
```

As we don't want/need to do anything different inside the `Core.ServiceSupervisor` module than the `DynamicSupervisor` is doing we can just delegate both of those inside the `Core.ServiceSupervisor` module:


```elixir
  # /apps/core/lib/core/service_supervisor.ex
  defdelegate start_link(module, args, opts), to: DynamicSupervisor
  defdelegate init(opts), to: DynamicSupervisor
```

That finishes our refactoring of both the `Naive.DynamicSymbolSupervisor` and the `Core.ServiceSupervisor` modules.

We can test to confirm that everything works as expected:


```bash
$ iex -S mix
iex(1)> Naive.start_trading("NEOUSDT")    
21:42:37.741 [info]  Starting Elixir.Naive.SymbolSupervisor worker for NEOUSDT
21:42:37.768 [info]  Starting new supervision tree to trade on NEOUSDT
{:ok, #PID<0.464.0>}
21:42:39.455 [info]  Initializing new trader(1614462159452) for NEOUSDT
iex(2)> Naive.stop_trading("NEOUSDT") 
21:43:08.362 [info]  Stopping Elixir.Naive.SymbolSupervisor worker for NEOUSDT
{:ok,
 %Naive.Schema.Settings{
   ...
 }}
iex(3)> Naive.start_trading("HNTUSDT")
21:44:08.689 [info]  Starting Elixir.Naive.SymbolSupervisor worker for HNTUSDT
21:44:08.723 [info]  Starting new supervision tree to trade on HNTUSDT
{:ok, #PID<0.475.0>}
21:44:11.182 [info]  Initializing new trader(1614462251182) for HNTUSDT
BREAK: (a)bort (A)bort with dump (c)ontinue (p)roc info (i)nfo
       (l)oaded (v)ersion (k)ill (D)b-tables (d)istribution
$ iex -S mix
21:47:22.119 [info]  Starting Elixir.Naive.SymbolSupervisor worker for HNTUSDT
21:47:22.161 [info]  Starting new supervision tree to trade on HNTUSDT
21:47:24.213 [info]  Initializing new trader(1614462444212) for HNTUSDT
iex(1)> Naive.shutdown_trading("HNTUSDT")
21:48:42.003 [info]  Initializing shutdown of Elixir.Naive.SymbolSupervisor worker for HNTUSDT
{:ok,
 %Naive.Schema.Settings{
   ...
 }}
```

The above test confirms that we can start, stop, and shut down trading on a symbol as well as autostarting of trading works.

## Use the `Core.ServiceSupervisor` module inside the `streamer` application

As we are happy with the implementation of the `Core.ServiceSupervisor` module we can upgrade the `streamer` application to use it.

We need to start with adding the `core` application to the list of dependencies of the `streamer` application:


```elixir
# /apps/streamer/mix.exs
  defp deps do
    [
      {:binance, "~> 0.7.1"},
      {:core, in_umbrella: true}, # <= core added to deps
      ...
```

We can now move on to the `Streamer.DynamicStreamerSupervisor` where we will remove everything (really everything including `import`s, `alias`es and even `require`) beside the `start_link/1` and the `init/1`. As with the `Naive.DynamicSymbolSupervisor` we will `use` the `Core.ServiceSupervisor` and pass all required options - *full* implementation of the `Streamer.DynamicStreamerSupervisor` module should look as follows:


```elixir
# /apps/streamer/lib/streamer/dynamic_streamer_supervisor.ex
defmodule Streamer.DynamicStreamerSupervisor do
  use Core.ServiceSupervisor,
    repo: Streamer.Repo,
    schema: Streamer.Schema.Settings,
    module: __MODULE__,
    worker_module: Streamer.Binance

  def start_link(init_arg) do
    Core.ServiceSupervisor.start_link(__MODULE__, init_arg, name: __MODULE__)
  end

  def init(_init_arg) do
    Core.ServiceSupervisor.init(strategy: :one_for_one)
  end
end
```

Not much to add here - we are `use`ing the `Core.ServiceSupervisor` module and passing options to it so it can macro generates `streamer` application-specific wrappers(like `start_worker/1` or `stop_worker/1` with required repo, schema etc) around generic logic from the `Core.ServiceSupervisor` module.

Using the `Core.Servicesupervisor` module will have an impact on the interface of the `Streamer.DynamicStreamerSupervisor` as it will now provide functions like `start_worker/1` instead of `start_streaming/1` etc.

As with the `naive` application, we need to update the Task function inside the `Streamer.Supervisor` module:


```elixir
# /apps/streamer/lib/streamer/supervisor.ex
      ...
      {Task,
       fn ->
         Streamer.DynamicStreamerSupervisor.autostart_workers()
       end}
      ...
```

As well as main `Streamer` module needs to forward calls instead of delegating:


```elixir
  # /apps/streamer/lib/streamer.ex
  alias Streamer.DynamicStreamerSupervisor

  def start_streaming(symbol) do
    symbol
    |> String.upcase()
    |> DynamicStreamerSupervisor.start_worker()
  end

  def stop_streaming(symbol) do
    symbol
    |> String.upcase()
    |> DynamicStreamerSupervisor.stop_worker()
  end
```

We can run a quick test to confirm that indeed everything works as expected:


```bash
$ iex -S mix
iex(1)> Streamer.start_streaming("NEOUSDT")
22:10:38.813 [info]  Starting Elixir.Streamer.Binance worker for NEOUSDT
{:ok, #PID<0.465.0>}
iex(2)> Streamer.stop_streaming("NEOUSDT")
22:10:48.212 [info]  Stopping Elixir.Streamer.Binance worker for NEOUSDT
{:ok,
 %Streamer.Schema.Settings{
   __meta__: #Ecto.Schema.Metadata<:loaded, "settings">,
   id: "db8c9429-2356-4243-a08f-0d0e89b74986",
   inserted_at: ~N[2021-02-25 22:15:16],
   status: "off",
   symbol: "NEOUSDT",
   updated_at: ~N[2021-02-27 22:10:48]
 }}
iex(3)> Streamer.start_streaming("LTCUSDT") 
22:26:03.361 [info]  Starting Elixir.Streamer.Binance worker for LTCUSDT
{:ok, #PID<0.490.0>}
BREAK: (a)bort (A)bort with dump (c)ontinue (p)roc info (i)nfo
       (l)oaded (v)ersion (k)ill (D)b-tables (d)istribution
^C
$ iex -S mix
...
22:26:30.775 [info]  Starting Elixir.Streamer.Binance worker for LTCUSDT
```

This finishes the implementation for both the `streamer` and the `naive` application. We are generating dynamic functions(metaprogramming) using Elixir macros which is a cool exercise to go through and feels like superpowers ;)

[Note] Please remember to run `mix format` to keep things nice and tidy.

Source code for this chapter can be found at [Github](https://github.com/frathon/create-a-cryptocurrency-trading-bot-in-elixir-source-code/tree/chapter_13)