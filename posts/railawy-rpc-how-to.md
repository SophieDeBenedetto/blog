# Railway Messaging Part II: Building RPC Features with Railway

In a [previous post], we introduced the Railway family of libraries for building messaging patterns between applications on our Learn.co ecosystem. We saw how these tools allowed us build robust async messaging systems in Elixir and Ruby with very little hand-rolled code. In this post, we'll learn how to use Railway to enact synchronous messaging patterns, also knows as 'RPC' (Remote Procedure Calls).

## The Problem: A Scary Story

Before we talk about what RPC is and how Railway allows us to enact it, I want to share a scary story. If you recall from our last post, we are responsible for building out the a feature in which an admin user of our Course Conductor app creates a student cohort, thereby sending a message to our Learn.co LMS to create a corresponding "batch" through which students will access lessons, *and* a corresponding message to our Registrar app through which Admissions reps can admit students.

Okay, on to our story.

Once upon a time, Course Conductor sent the `"CreateBatch"` message to Learn.co, which promptly created the batch and sent the `"BatchCreated"` message. But oh no! Course Conductor had a bug!  It failed to handle the `"BatchCreated"` message! 💀👻 💀👻 💀👻 💀👻.

The bug went undetected for a horrifying and spooky three days! After the fix was shipped, Course Conductor had no way to get the missed batches from Learn.co 💀👻.

If only there was _some way_ for Course Conductor to synchronously request the latest batches from Learn.co so that it could bring its system up to date...

Luckily for us, Railway exposes an API for just this kind of situation––RPC!

## The Solution

RPC messaging, or Remote Produce Call, is a pattern in which one application can request the execution of a service in another application. The Railway libraries expose an RPC API that allows one application to make a synchronous remote produce call to another application and get make the requested info.

In the Railway RPC pattern, clients send a request to a server over that server’s exchange and wait for a response on their own "reply to" queue. A server is listening for messages on a queue and will publish responses to the client who sent the message.

Let's take a look at how we can build an RPC feature with Railway.

## Building RPC Features with Railway

Here's a look at the feature we're going to support, per our spooky story.

1. Course Conductor will send a `"BatchesRequest"` to Learn.co over the Learn.co server's dedicated "batches request" exchange.
2. Learn.co will respond to this request over the Course Conductor client's "reply to" queue.

![pic]

### Defining an RPC Client in Elixir

We will define a publisher module that uses the `Railway.Publisher` behavior and knows the queue and exchange of the server to which it will publish the request. Under the hood, Railway will establish this queue and bind it to the exchange if no such queue exists.

Then, we will call the `publish_sync/1` function that the `Railway.Publisher` exposes.

```elixir
defmodule Ipc.BatchesRequestPublisher do
  use RailwayIpc.Publisher,
      queue: "ipc:batches:requests"
      exchange: "ipc:batches:requests"

  def get_batches do
    response = publish_sync(request_message)
    IO.inspect response
  end
end
```

Now let's take a look at the RPC server in our Learn.co Rails app that will receive this request.

### Defining an RPC Server in Ruby

We need to do the following to create our RPC server:

* Specify the queue to which the server will subscribe
* Specify the message it is expecting and the service it will use to respond, i.e. the "responder"
* Define the responder class
* Specify the error adapter class
* Define the error adapter class

#### Specifying the Subscription Queue

We will define a server class that inherits from `RailwayIpc::Server` and knows the queue to which to subscribe.

```ruby
module Ipc
  module Servers
    class BatchServer < RailwayIpc::Server
      listen_to queue: "ipc:batch:requests"
    end
  end
end
```

#### Specifying the Expected Message and Responder

Now we define the type of message our server expects to receive and the responder class with which it will respond. Here we're expecting a protobuf message, `LearnIpc::Requests::Batches`.

```ruby
module Ipc
  module Servers
    class BatchServer < LearnIpc::Server
      listen_to queue: "ipc:batch:requests"

      respond_to LearnIpc::Requests::Batches,
        with: Ipc::Responders::BatchesResponder
    end
  end
end
```

#### Defining the Responder Class

We define a responder class that inherits from "`RailwayIpc::Responder`" and implements a `respond` method that takes a block. This block must return a `LearnIpc::Documents` protobuf message instance.


```ruby
module Ipc
  module Responders
    class BatchesResponder < RailwayIpc::Responder
      respond do |message|
        batches = Batch.all
        LearnIpc::Documents::Batches.new(batches: batches)
      end
    end
  end
end
```

#### Specifying the Error Adapter
The error adapter is responsible for telling your Railway server what kind of error message protobuf to respond with in the event of an error processing the request.

```ruby
module Ipc
  module Servers
    class BatchServer < RailwayIpc::Server
      listen_to queue: "ipc:batch:requests"

      respond_to LearnIpc::Requests::Batches,
        with: Ipc::Responders::BatchesResponder

      rpc_error_adapter Ipc::DataAdapters::RpcErrorAdapter
    end
  end
end
```

#### Defining the Error Adapter

Our Error adapter must conform to the following API:

* Implement an `error_message` method that takes in an error object and a protobuf message object.
* Return the protobuf error message instance of your choosing.

```ruby
module Ipc
  module DataAdapters
    class RpcErrorAdapter
      def self.error_message(error, message)
        LearnIpc::ErrorMessage.new(
          correlation_id: message.correlation_id,
          user_uuid: message.user_uuid,
          error_message: error.message
        )
      end
    end
  end
end
```

And that's it! Now our Course Conductor app can send a synchronous RPC "get batches" request to Learn.co, which will respond with the batches document.

In this example, we defined a Railway RPC client in Elixir and an RPC server in Ruby. Before we conclude, let's take a look at how we can use Railway to define a Ruby client and an Elixir server.

### Defining an RPC Client in Ruby

In order to define our client, we need to do three things:

* Specify the queue and exchange to which to publish
* Specify the type of message we expect to receive in response
* Specify the error adapter class

We will define a client that knows the queue and exchange to which it will publish. And it will specify the type  of message it expects to get in response and the error adapter class

#### Specifying the Queue and Exchange

We define a client class that inhertis from the `RailwayIpc::Client` and publishes to a given queue and exchange.

```ruby
module Ipc
  module Clients
    class CohortClient < RailwayIpc::Client
      publish_to queue: "ipc:cohort:requests", exchange: "ipc:cohort:requests"
    end
  end
end
```

#### Specifying the Expected Response Message

Next, we will use the `handle_response` class method to define the type of message we are expecting the corresponding server to send back.

```ruby
module Ipc
  module Clients
    class CohortClient < RailwayIpc::Client
      publish_to queue: "ipc:cohort:requests", exchange: "ipc:cohort:requests"

      handle_response LearnIpc::Documents::Cohorts
    end
  end
end
```

#### Specifying the Error Adapter

Lastly, we need to specify the error adapter that our client should use in the event of a client-side RPC failure:

```ruby
module Ipc
  module Clients
    class CohortClient < RailwayIpc::Client
      publish_to queue: "ipc:cohort:requests", exchange: "ipc:cohort:requests"
      handle_response LearnIpc::Documents::Cohorts
      rpc_error_adapter Ipc::DataAdapters::RpcErrorAdapter
    end
  end
end
```

The error adapter should be defined according to the API described in the error adapter section above.

#### Publishing the Message

Now our client is ready to send it RPC request using the `request` method:

```ruby
response = Ipc::Clients::CohortClient.request(request_message)
```

Let's take a look at how we can define a Railway RPC server in Elixir.

### Defining an RPC Server in Elixir

To define our server, we need to do three things:

* Specify the queue and exchange to which it will subscribe
* Implement a `handle_in/1` function that matches the expected message
* Respond with a protobuf document message

#### Specifying the Queue and Exchange

Our RPC consumer uses the `RailwayIpc.RequestsConsumer` behaviour and knows its queue and exchange.  

```elixir
defmodule Ipc.CohortsRequestConsumer do
  use RailwayIpc.RequestsConsumer,
      exchange: "ipc:cohorts:requests",
      queue: "ipc:cohorts_requests"
end
```

#### Implementing the `handle_in/1` Function

Our consumer knows how handle the `LearnIpc.Requests.Cohorts` protobuf request message.

```elixir
defmodule Ipc.CohortsRequestConsumer do
  use RailwayIpc.RequestsConsumer,
      exchange: "ipc:cohorts:requests",
      queue: "ipc:cohorts_requests"

  def handle_in(%LearnIpc.Requests.Cohorts{} = _req) do
  end
end
```

#### Responding with a Protobuf Document

Our consumer responds with the `LearnIpc.Documents.Cohorts` protobuf document message.

```elixir
defmodule Ipc.CohortsRequestConsumer do
  use RailwayIpc.RequestsConsumer,
      exchange: "ipc:cohorts:requests",
      queue: "ipc:cohorts_requests"

  def handle_in(%LearnIpc.Requests.Cohorts{} = _req) do
    cohorts = Cohort.all
    response = LearnIpc.Documents.Cohorts.new(cohorts: cohorts)
    {:reply, response}
  end
end
```

And that's it!

### Conclusion

The Railway libraries allow us build out robust synchronous messaging systems in Elixir and Ruby with very little hand-rolled code. And since all communication is backed by RabbitMQ, we don't need to worry about building authentication flows like we would with HTTP.

Railway offers a sane and easy-to-use API for defining clients and servers in both languages, and, together with Google Protobuf, helps our team enforce consistent message contracts between applications. We hope that this tools will allow us to quickly and easily meet our synchronous inter-app communication needs with minimal mental overhead. 
