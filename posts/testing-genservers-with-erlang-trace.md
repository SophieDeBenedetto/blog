# Testing GenServers with Erlang Trace
When working on a messaging system with RabbitMQ for my upcoming workshop at ElixirConf, I ran into a common Elixir testing challenge--testing that a GenServer received a message.

## The Problem
ExUnit provides an [`assert_receive/3`](https://hexdocs.pm/ex_unit/ExUnit.Assertions.html#assert_receive/3), but that only allows you to check the mailbox of the current process, i.e. the process running the test. So, how can we check that a GenServer running in our application received a certain message? This is the issue I encountered when testing our RabbitMQ consumer GenServer.

We have a GenServer that runs when the application starts up and consumes messages from a RabbitMQ queue. How can we set an expectation that the consumer does in fact receive and process a given message sent by a publisher to that queue?

We can do exactly that with the help of ExUnit's [`start_supervised/2`](https://hexdocs.pm/ex_unit/ExUnit.Callbacks.html#start_supervised/2) callback and Erlang's [`trace/3`](http://erlang.org/doc/man/erlang.html#trace-3) function.

## Introducing Erlang Trace

Erlang's `trace/3` function is pretty powerful. It allows us to attach a trace to a specified process. What does this mean? If we trace a given process, we are telling Erlang to send a message to a calling process (in our case, the test), whenever the trace process receives a message. Sneaky!

We'll use ExUnit's `start_supervised/2` function to start our GenServer and capture its PID. Then, we'll use `trace/3` to ensure that whenever the GenServer PID receives a message, our test gets notified. With that in place, we can assert that the _test_ received a message from the trace, thereby testing that our GenServer received a message.

Let's do it!

### Step 1. Start the GenServer with `start_supervised/2`

First, we'll start up our GenServer and capture its PID

```elixir
defmodule SpyRadio.IntegrationTest do
  use ExUnit.Case

  test "the consumer receives a message when one is published to its queue" do
    consumer_pid = start_supervised!(SpyRadio.RabbitMQ.Consumer)
  end
end
```

### Step 2: Start the trace

Next, we'll use Erlang's `trace/3` function to trace the GenServer PID such that the test process receives a message whenever the GenServer PID does.

```elixir
defmodule SpyRadio.IntegrationTest do
  use ExUnit.Case

  test "the consumer receives a message when one is published to its queue" do
    consumer_pid = start_supervised!(SpyRadio.RabbitMQ.Consumer)
    :erlang.trace(consumer_pid, true, [:receive])
  end
end
```

### Step 3: Set the Assertion
Now we're ready to enact the code that we expect to result in our GenServer receiving a message--publishing to a RabbitMQ queue.

```elixir
defmodule SpyRadio.IntegrationTest do
  use ExUnit.Case

  test "the consumer receives a message when one is published to its queue" do
    consumer_pid = start_supervised!(SpyRadio.RabbitMQ.Consumer)
    :erlang.trace(consumer_pid, true, [:receive])
    SpyRadio.RabbitMQ.Publisher.publish("some message")
  end
end
```

Publishes a message _should_ cause our consumer GenServer to receive the `:basic_consume_ok` message. This will in turn send the following message to the test process which started the trace:

```elixir
{:trace, consumer_pid, :receive, {:basic_consume_ok, %{consumer_tag: _tag}}
```
So, we can use ExUnit's [`assert_receive/3`](https://hexdocs.pm/ex_unit/ExUnit.Assertions.html#assert_receive/3) function to assert that the test receives this message:

```elixir
defmodule SpyRadio.IntegrationTest do
  use ExUnit.Case

  test "the consumer receives a message when one is published to its queue" do
    consumer_pid = start_supervised!(SpyRadio.RabbitMQ.Consumer)
    :erlang.trace(consumer_pid, true, [:receive])
    SpyRadio.RabbitMQ.Publisher.publish("some message")

    assert_receive {:trace, ^consumer_pid, :receive, {:basic_consume_ok, %{consumer_tag: _tag}}},
                   1000
  end
end
```

In this way, we _can_ in fact test that our GenServer received a certain message.

## Conclusion
Erlang's `trace/3` function adds a powerful tool to our Elixir testing arsenal. It helps us solve a common testing problem--that of asserting that your GenServers received a certain message. Together with ExUnit's `start_supervised/2` callback and `assert_receive/3` function, we were able to write exactly the test we needed for our RabbitMQ messaging system.

## Special Thanks
Special thanks to Steven Nuñez who turned me on to Erlang's `trace/3` function and who generally gives me all of my ideas. Check out his [recent post on managing RabbitMQ connections in Elixir with ExRabbitPool](https://hostiledeveloper.com/2020/08/09/connection-pools-and-rabbitmq.html) and learn more about working with RabbitMQ and Elixir by signing up for our [ElixirConf 2020 workshop](https://2020.elixirconf.com/trainers/3/course)! 
