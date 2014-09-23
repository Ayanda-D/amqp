# AMQP

[![Build Status](https://travis-ci.org/pma/amqp.png?branch=master)](https://travis-ci.org/pma/amqp)
[![Hex Version](http://img.shields.io/hexpm/v/amqp.svg)](https://hex.pm/packages/amqp)

Simple Elixir wrapper for the Erlang RabbitMQ client.

The API is based on Langohr, a Clojure client for RabbitMQ.

Caveat: The somewhat more idiomatic API adds some overhead. Input parameters
are keyword lists and need to be converted to Erlang records. Output parameters and
callback parameters are converted from Erlang records into Maps/Structs.

## Usage

Add AMQP as a dependency in your `mix.exs` file.

```elixir
def deps do
  [{:amqp, "0.0.4"}]
end
```

After you are done, run `mix deps.get` in your shell to fetch and compile AMQP. Start an interactive Elixir shell with `iex -S mix`.

```iex
iex> {:ok, conn} = AMQP.Connection.open
{:ok, %AMQP.Connection{pid: #PID<0.165.0>}}
iex> {:ok, chan} = AMQP.Channel.open(conn)
{:ok, %AMQP.Channel{conn: %AMQP.Connection{pid: #PID<0.165.0>}, pid: #PID<0.177.0>}
iex> AMQP.Queue.declare chan, "test_queue"
{:ok, %{consumer_count: 0, message_count: 0, queue: "test_queue"}}
iex> AMQP.Exchange.declare chan, "test_exchange"
:ok
iex> AMQP.Queue.bind chan, "test_queue", "test_exchange"
:ok
iex> AMQP.Basic.publish chan, "test_exchange", "", "Hello, World!"
:ok
iex> {:ok, payload, meta} = AMQP.Basic.get chan, "test_queue"
iex> payload
"Hello, World!"
iex> AMQP.Queue.subscribe chan, "test_queue", fn(payload, _meta) -> IO.puts("Received: #{payload}") end
{:ok, "amq.ctag-5L8U-n0HU5doEsNTQpaXWg"}
iex> AMQP.Basic.publish chan, "test_exchange", "", "Hello, World!"
:ok
Hello, World!
```

### Slightly more advanced usage: Setup a consumer GenServer

```elixir
defmodule Consumer do
  use GenServer
  use AMQP

  def start_link do
    GenServer.start_link(__MODULE__, [], [])
  end

  @exchange    "gen_server_test_exchange"
  @queue       "gen_server_test_queue"
  @queue_error "#{@queue}_error"

  def init(_opts) do
    {:ok, conn} = Connection.open
    {:ok, chan} = Channel.open(conn)
    # Limit unacknowledged messages to 10
    Basic.qos(chan, prefetch_count: 10)
    Queue.declare(chan, @queue_error, durable: true)
    # Messages that cannot be delivered to any consumer in the main queue will be routed to the error queue
    Queue.declare(chan, @queue, durable: true, arguments: [{"x-dead-letter-exchange", :longstr, ""}, {"x-dead-letter-routing-key", :longstr, @queue_error}])
    Exchange.fanout(chan, @exchange, durable: true)
    Queue.bind(chan, @queue, @exchange)
    # Register the GenServer process as a consumer
    Basic.consume(chan, @queue)
    {:ok, chan}
  end

  def handle_info({payload, %{delivery_tag: tag, redelivered: redelivered}}, chan) do
    spawn fn -> consume(chan, tag, redelivered, payload) end
    {:noreply, chan}
  end

  defp consume(channel, tag, redelivered, payload) do
    try do
      number = String.to_integer(payload)
      if number <= 10 do
        Basic.ack channel, tag
        IO.puts "Consumed a #{number}."
      else
        Basic.reject channel, tag, requeue: false
        IO.puts "#{number} is too big and was rejected."
      end
    rescue
      exception ->
        # Requeue unless it's a redelivered message.
        # This means we will retry consuming a message once in case of exception
        # before we give up and have it moved to the error queue
        Basic.reject channel, tag, requeue: not redelivered
        IO.puts "Error converting #{payload} to integer"
    end
  end

end
```

```iex
iex> Consumer.start_link
{:ok, #PID<0.261.0>}
iex> {:ok, conn} = AMQP.Connection.open
{:ok, %AMQP.Connection{pid: #PID<0.165.0>}}
iex> {:ok, chan} = AMQP.Channel.open(conn)
{:ok, %AMQP.Channel{conn: %AMQP.Connection{pid: #PID<0.165.0>}, pid: #PID<0.177.0>}
iex> AMQP.Basic.publish chan, "gen_server_test_exchange", "", "5"
:ok
Consumed a 5.
iex> AMQP.Basic.publish chan, "gen_server_test_exchange", "", "42"
:ok
42 is too big and was rejected.
iex> AMQP.Basic.publish chan, "gen_server_test_exchange", "", "Hello, World!"
:ok
Error converting Hello, World! to integer
Error converting Hello, World! to integer
```
