# Railway Messaging Part I: Introducing Ruby and Elixir Packages for Inter-App Communication

## You Want IPC

IPC, which stands for Inter-Process Communication, is something you want if you can answer "yes" to any of these questions:

* Does your organization support multiple applications?
* Does your organization have multiple business domains that your engineering team is tasked with supporting?
* Is it difficult to coordinate engineering work across these domains?
* Do you find it hard to innovate new tools and platforms without the fear of breaking existing functionality?

Then you might benefit from a sane, extensible IPC messaging pattern within your technology architecture.

### The Problem

As our business grows, we need to meet the needs of more and more groups of stakeholders. We need to move fast *without* breaking things. It can feel like there are only two solutions:

* Add to your monolith! Thereby creating more mental overhead and making it harder for multiple discrete squads of devs to work on the monolith at a given time.
* Microservice all the things! And expend a ton of time and effort figuring out data syncing requirements, event tracking, cross-app authentication and authorization and more.

What if there was a third way? A way to gain infrastructure flexibility, distributed team autonomy, *and* move fast *without breaking things*?

### The Solution

Adopting asynchronous messaging patterns between applications gives autonomy between teams and encourages us to build small. It allows us to communicate facts and build a set of applications that can run independently of one another. It produces a system that is highly cohesive and lightly coupled.

But this type of communication, which we'll refer to as "IPC", isn't a magic bullet. Having decided to enact messaging between apps still begs the question: How can we build this communication quickly and sanely, in a way that is replicable?

Introducing...

## Railway IPC

The Railway IPC family of tools that we're working on here on The Flatiron School engineering team provides a sane, replicable IPC API with the help of a Ruby gem as well as an Elixir package that you can plug into your Rails or Phoenix app with ease.

These packages leverage RabbitMQ queues and exchanges to publish and consume messages between the various apps in our ecosystem and allow users to build workflows that ascribe to one of two patterns:

* Async messaging
* Synchronous messaging via remote procedure calls (RPC)

**With the help of these libraries, we can write IPC features quicly and easily. We don't have to reinvent the wheel every time we want to support a new set of IPC features and we can enact this type of communication with very little hand rolled code.**

In this series of posts we'll take a look at:

* Part I: How To Use Railway Packages for Async Messaging
* Part II: How to Use Railway Packages for RPC Calls
* Part III: Railway Ruby Gem Under the Hood
* Part IV: Railway Hex Package Under the Hood

At time of writing, the Railway Hex package is available here, but the Railway Ruby Gem is still a work-in-progress.

## Part I: How to Use Railway Packages for Async Messaging

In the Railway async messaging pattern, consumers bind a RabbitMQ queue to an exchange and listen to messages on that exchange, over the queue. Publishers publish messages to an exchange.

In order to illustrate how we can use Railway to build out a messaging system between apps, we'll discuss the following feature:

### Learn.co Cohort Creation

At The Flatiron School, we need to create "cohorts" into which we can enroll students. Let's say we have three apps:

* Course Conductor: An Elixir app that administrators use to create a given cohort for a given course
* Learn.co: A Rails app and our Learning Management System through which students access lessons
* Registrar: An Elixir app for admitting students and billing them

An admin will create a cohort in Course Conductor. This needs to have two side effects:

* Creating a "batch" in our Learn.co LMS, through which admitted students will access lessons
* Creating a "registration cohort" in our Registrar app, so that we can admit students into a cohort for a course and bill them their tuition.

Our IPC flow will look something like this:

1. User creates a cohort by filling out a form in the Course Conductor app. This publishes a "create batch" command message.
2. The Learn.co LMS will consume this message, create the corresponding batch record in its own system and publish a success/failure event accordingly.
3. Course Conductor will consume _this_ message, update its own records and in turn publish a "cohort created" event that the Registrar app can consume.  



Step one of the process requires the CourseConductor app to publish a "CreateBatch" command message that Learn.co can consume. So we'll begin by looking at how we can use the Railway Hex package to define an IPC publisher in Elixir.

### Defining a Publisher in Elixir

Having included the Railway package in your app, simply define a publisher module that specifies the exchange to which it will publish.

```elixir
defmodule CourseConductor.Ipc.BatchCommandsPublisher do
  use RailwayIpc.Publisher, exchange: "ipc:batch:commands"
end
```

That's it! Now we're ready to call on our publisher.

```elixir
CourseConductor.Ipc.BatchCommandsPublisher.publish(message)
```

This message will be consumed by a consumer running in the Learn.co Rails app. Let's take a look at how to use the Railway Ruby Gem to define such a consumer now.

### Defining a Consumer in Ruby

We'll need to write a consumer class that:
* Defines the queue and exchange
* Specifies which messages it cares about consuming
* Specify the handler with which to process those messages

#### Specifying the Queue and Exchange

We define a consumer, inherit it from `RailwayIpc::Consumer`, and specify the queue and exchange with the `listen_to` class method.

```ruby
module Ipc
  module Consumers
    class BatchCommandsConsumer < RailwayIpc::Consumer
      listen_to queue: "ironboard:batch:commands", exchange: "ipc:batch:commands"
    end
  end
end
```

This will create the queue and exchange in RabbitMQ if they did not already exist and bind the queue to the given exchange.

#### Specify the Expected Message and Message Handler
Then, we define the type of message and handler with the `handle` class method.


```ruby
module Ipc
  module Consumers
    class BatchCommandsConsumer < RailwayIpc::Consumer
      listen_to queue: "ironboard:batch:commands", exchange: "ipc:batch:commands"
      handle LearnIpc::Commands::CreateBatch,  with: Ipc::Handlers::CreateBatchHandler
    end
  end
end
```

Now we're ready to implement the message handler, `Ipc::Handlers::CreateBatchHandler`.

#### Implementing The Handler

We define a class that inherits from `RailwayIpc::Handler` and implement a `handle` method that takes a block.
Its up to you what you do in the handle method’s block. But it  must return an object that responds to `#success?`.

```ruby
module Ipc
  module Handlers
    class CreateBatchHandler < RailwayIpc::Handler
      handle do |message|
        Ipc::CreateBatchService.execute(message)
      end
    end
  end
end
```

Now we're ready to start running our consumer.

#### Starting the Consumer

The Railway IPC Ruby Gem provides a handy rake task we can use to start up consumer:

```bash
rake railway_ipc:start_consumers CONSUMERS=Ipc::Consumers::BatchCommandsConsumer
```

And that's it! Now our Elixir publisher will publish messages to the `"ipc:batch:commands"` exchange and these messages will be consumed by our batch commands consumer running our Rails app.

The next step of our feature flow requires the Learn.co Rails app to publish a `"BatchCreated"` or `"FailedToCreateBatch"` event depending on the success/failure of batch creation.

Let's take a look at how we can use the Railway Ruby Gem to create such a publisher now.

### Defining a Publisher in Ruby

Just like when we defined our Elixir publisher, we only need to give our publisher the exchange to which it will publish. We define a class that inherits from `RailwayIpc::Publisher`, and specify the exchange:

```ruby
module Ipc
  module Publishers
    class BatchEventsPublisher < RailwayIpc::Publisher
      exchange 'ipc:batch:events'
    end
  end
end
```

Now we're ready to call on our publisher:

```ruby
Ipc::Publishers::BatchEventsPublisher.instance.publish(message)
```

That's it! If we want the Course Conductor Elixir app to consume this message, we need to define a consumer in that app. Let's take a look at how to define an Elixir consumer with the Railway Hex package.

### Defining a Consumer in Elixir

Just like when we defined our Ruby consumer, we need a consumer that:

* Defines the queue and exchange
* Specifies which messages it cares about consuming
* Specify the handler with which to process those messages

#### Specifying the Queue and Exchange

We define a module that uses the `RailwayIpc.EventsConsumer` behaviour, and specifies the exchange to which it will listen to and the queue to bind to that exchange:

```elixir
defmodule CourseConductor.Ipc.BatchEventsConsumer do
  use RailwayIpc.EventsConsumer,
      exchange: "ipc:batch:events",
      queue: "course_conductor:batch:events"
end
```

#### Implement the Handler Function for the Expected Message

We need our consumer to implement a `handle_in/1` function that uses function arity pattern matching to handle the expected message, in this case the `"BatchCreated"` message. It will handle this message by doing some work and then publishing the `"CohortCreatedMessage"`.

```elixir
defmodule CourseConductor.Ipc.BatchEventsConsumer do
  ...
  alias LearnIpc.Events.BatchCreated

  def handle_in(%BatchCreated{} = _message) do
    # do work
  end
end
```

#### Starting the Consumer

We want our consumer to start up when the app starts up, and we want our consumer to be supervised so that it can restart in the event of a crash. We'll add the following to our `application.ex` file, in the `start/1` function:

```elixir
children = [
             {RailwayIpc.Connection.Supervisor, [CourseConductor.Ipc.BatchEventsConsumer]},
             ...
           ]
opts = [strategy: :one_for_one, name: CourseConductor.Supervisor]
Supervisor.start_link(children, opts)
```

And that's it!

Before we conclude, we have one more topic to cover.

## Message Contracts with Google Protobuf

How can we ensure that consistent messages are passed between applications and that our packages can reliably decode and operate on them? We're leveraging Google Protobuf to establish shared message contracts across applications.

We created a dedicated repository in which we define shared message contracts using the Google Protobuf protoc DSL and convert those protoc messages into Ruby classes and Elixir modules using the protoc CLI. This allows us to define shared messages in one central location that is tracked by version control.

In order to use our Google Protobuf tool, we define a protoc message, for example:

```protoc
syntax = "proto3";

package learn_ipc.commands;

message CreateBatch {
  string user_uuid = 1;
  string correlation_id = 2;
  string uuid = 3;

  message Data {
    string batch_iteration = 1;
    string organization_uuid = 2;
  }

  map<string, string> context = 4;

  Data data = 5;
}
```

Then run a command line script to compile the protoc message into Ruby and/or Elixir:

```bash
docker-compose run converter ruby commands create_batch
```

Which will produce the following Ruby class:

```ruby
require 'google/protobuf'
Google::Protobuf::DescriptorPool.generated_pool.build do
  add_message "learn_ipc.commands.CreateBatch" do
    optional :user_uuid, :string, 1
    optional :correlation_id, :string, 2
    optional :uuid, :string, 3
    map :context, :string, :string, 4
    optional :data, :message, 5, "learn_ipc.commands.CreateBatch.Data"
  end
  add_message "learn_ipc.commands.CreateBatch.Data" do
    optional :batch_iteration, :string, 1
    optional :organization_uuid, :string, 2
  end
end

module LearnIpc
  module Commands
    CreateBatch = Google::Protobuf::DescriptorPool.generated_pool.lookup("learn_ipc.commands.CreateBatch").msgclass
    CreateBatch::Data = Google::Protobuf::DescriptorPool.generated_pool.lookup("learn_ipc.commands.CreateBatch.Data").msgclass
  end
end
```

We can add this class to our Learn.co Rails app application code and use it by:

* Specifying that our `Ipc::Commands::BatchCommandsConsumer` is listening for this message
* Implementing our `Ipc::Handlers::CreateBatchHandler` to operate on the data within this message.

We can also instantiate and publish such messages in Elixir and Ruby. For example, in Ruby:

```ruby
message = LearnIpc::Commands::CreateBatch.new(user_uuid: user.learn_uuid)
Ipc::Publishers::MyPublisher.instance.publish(message)
```

The Railway packages abstract away all the hard work of encoding and decoding these Protobuf messages for RabbitMQ for us. We can instantiate and publish these messages directly and we can consume and process these messages directly.

## Conclusion

The Railway libraries allow us build out robust async messaging systems in Elixir and Ruby with very little hand-rolled code. They offer sane and easy-to-use APIs for defining consumers and publishers in both languages, and, together with Google Protobuf, help our team enforce consistent message contracts between applications. All in all, we hope that these tools will empower us to meet the complex needs of our growing business, allowing relatively autonomous teams of engineers to grow and change our technology with minimal breakage.
