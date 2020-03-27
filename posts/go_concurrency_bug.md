# Synchronizing Go Routines with Channels and WaitGroups

In debugging a mysteriously hanging Go function recently, I learned something new about how to use WaitGroups and Channels to synchronize Go routines. Keep reading to learn more!

## Intro

As a new relatively new Go programmer, I was stumped recently by a bug that causes a Go function running multiple Go routines to hang indefinitely and fail to ever return. After take a deeper dive into the usage of WaitGroups and Channels to manage the behavior of Go routines, I finally understood the _right_ way to leverage these tools and was able to resolve the bug.

In this post we'll

* Examine the buggy code and break down why it doesn't work
* Walk through the _right_ way to synchronize Go routines with a simple example
* Fix our bug!

## The Blocking Bug

In an application responsible for displaying GitHub usage analytics, we have a function, `MergeContributors`, that is responsible for taking two GitHub user accounts that belong to the same person and "merging" them into one so that the UI can accurately reflect their contributions to project commits, pull requests and merges. This "merge" entails the following:

* Update all of the analytic engine's stored Git commits so that _all_ user's commits from _both_ accounts are correctly associated to one primary account
* Update all of the analytic engine's stored pull requests so that _all_ user's PRs from _both_ accounts are correctly associated to one primary account
* Update all of the analytic engine's stored PR merges so that _all_ user's merges from _both_ accounts are correctly associated to one primary account
* Finally, if all of those actions succeeded, update a record indicating that the two accounts have been "merged"

The function was bugging out in the following way:

* If an error occurred in any of the first three steps
* Then the function would block, hanging forever and never returning

Yikes! Let's take a cloesr look at the original code so that we can diagnose the issue:

```go
func MergeCOntributors(primaryAccount, secondaryAccount) error {
  // Create a wait group to manage the goroutines.
	var waitGroup sync.WaitGroup
	c := make(chan error)

	// Perform 3 concurrent transactions against the database.
	waitGroup.Add(3)

	go func() {
		waitGroup.Wait()
		close(c)
	}()

	// Transaction #1, merge "commit" records
	go func() {
		defer waitGroup.Done()

		err := mergeCommits(primaryAccount, secondaryAccount)
		if err != nil {
			c <- err
		}
  }()

  // Transaction #2, merge "pull request" records
	go func() {
		defer waitGroup.Done()

		err := mergePullRequests(primaryAccount, secondaryAccount)
		if err != nil {
			c <- err
		}
  }()

  // Transaction #3, merge "merge" records
	go func() {
		defer waitGroup.Done()

		err := mergePullRequestMerges(primaryAccount, secondaryAccount)
		if err != nil {
			c <- err
		}
  }()

  waitGroup.Wait()

  for err := range c {
		if err != nil {
			return err
		}
  }

  return markMerged(primaryAccount, secondaryAccount)
}
```

### Breaking Down The Bug

Let's walk through what's happening here:

* We establish a wait group, `waitGroup` and a channel that expects to receive errors

```go
var waitGroup sync.WaitGroup
c := make(chan error)
```

* We increment the wait group count to three, since we will be using it to orchestrate our three synchronous DB transaction Go routines

```go
waitGroup.Add(3)
```

* We spin off a separate Go routine that will block until the `waitGroup`'s count is back down to zero. It will then close the channel.

```go
go func() {
  waitGroup.Wait()
  close(c)
}()
```

* We spin off our three concurrent Go routines, one for each database transaction. Each Go routine runs a function that is responsible for making some database call, sending an error to the channel if necessary, and decrementing the `waitGroup`'s count before the function returns.

* Then, we have a call to `waitGroup.Wait()`. This will block the execution of the main function until the `waitGroup`'s count is back down to zero.

* After this blocking call, we're using `range` to iterate over the messages sent to the channel. The call to `range` will continue listening for messages to the channel until the channel is closed. This is a blocking operation.

* Once the channel is closed by our earlier Go routine, the one that is waiting for the `waitGroup` count to get down to zero and then calling `close(c)`, the function will proceed to run, either returning an error if one was received by the channel or moving on to the last piece of work, the call to `markMerged`.

Have you spotted the problem yet?

### Understanding The Problem

In order to spot the bug, we need to understand something about how sending and receiving messages over a channel works. When we send a message over a channel, that call to send the message is _blocking_, until the message is read from the channel.

Taking a closer look at one of our DB transaction Go routines:

```go
// Transaction #1, merge "commit" records
go func() {
  defer waitGroup.Done()

  err := mergeCommits(primaryAccount, secondaryAccount)
  if err != nil {
    c <- err
  }
}()
```

We can understand that the line in which we send a message to the channel, `c <- error`, will block the execution of the anonymous function running in our Go routine _until that message is read_.

Where are we reading messages? Via our `range` call, which comes _after_ a call to `waitGroup.Wait()`

Its our duplicated call to `waitGroup.Wait()` here:

```go
...
waitGroup.Wait()

for err := range c {
  if err != nil {
    return err
  }
}

return markMerged(primaryAccount, secondaryAccount)
```

Here's the problem:

* The anonymous function sends a message to the channel, which blocks until the message is read
* The message-reading code, our `range` call, won't run until _after_ the `waitGroup` count is back down to zero, because it comes _after_ a synchronous call to `waitGroup.Wait()`
* Since the message can't be read yet, the anonymous function running in our go routine _can't_ finish running--it won't call the defered `waitGroup.Done()`.
* This means that the `waitGroup` count will never get back down to zero, which in turn means the call to `waitGroup.Wait()` that comes right before our `range` call _will never unblock_.
* Since `waitGroup.Wait()` will never unblock, we can't call `range`, the message will never be read from the channel, and we're back where we started--in an infinite block!

## The Right Way To Synchronize Go Routiens

In order prevent this block, we need to ensure that our `range` call will run and read messages from the channel _while_ the Go routines are running. This will ensure that any Go routine that sends a message to a channel will _not_ block on that send, thereby allowing the Go routine's anonymous function to call `waitGroup.Done()`.

Let's take a look at a simple example:

```go
package main

import "fmt"

func main() {
  // Create a wait group to manage the goroutines.
	var waitGroup sync.WaitGroup
	c := make(chan string)

	// Perform 3 concurrent transactions against the database.
	waitGroup.Add(3)

	go func() {
		waitGroup.Wait()
		close(c)
	}()

	go func() {
		defer waitGroup.Done()

		c <- "one"
  }()

  go func() {
		defer waitGroup.Done()

		c <- "two"
  }()

  go func() {
		defer waitGroup.Done()

		c <- "three"
  }()

  for str := range c {
		fmt.Println(str)
  }
}
```

Let's break this down:

* We establish a wait group and set its count to three
* We establish a channel that can receive messages that are strings
* Spin off a Go routine that will wait under the `waitGroup`'s count is zero, then close the channel
* Create three separate Go routines, each of which will write a message to the channel and, once that message is read, decrement the `waitGroup` count
* Then, _at the same time as the running of these Go routines_, we are range-ing over the channel, reading any incoming messages and printing them to STDOUT.
* Since our range call is running _while_ the Go routines are running, the messages each routine sends to the channel are read immediately. The calls in each routine to send the message therefore do not block for long, and each routine's anonymous function is able to invoke the `waitGroup.Done()` call.
* Once each Go routine concludes, having decremented the `waitGroup` count, the _first_ Go routine (the one that called `waitGroup.Wait()`) will unblock and close the channel.
* Once the channel is closed and all its messages are read, the range will _stop_ listening for messages and the main function will finish running.

If we run the code above, we'll see that the function doesn't block improperly. Instead, we see the following output successfully printed:

```
one
two
three
```

Now that we understand the right way to synchronize our Go routines, let's fix our bug!

## Fixing the Bug

We need to get rid of that second, synchronous call to `waitGroup.Wait()`. This call is preventing messages from getting read from the channel, which in turn prevents any calls to `waitGroup.Done()`. This is the cause of the block in our function.

If we remove the offending line, we're left with:

```go
 func MergeCOntributors(primaryAccount, secondaryAccount) error {
  // Create a wait group to manage the goroutines.
	var waitGroup sync.WaitGroup
	c := make(chan error)

	// Perform 3 concurrent transactions against the database.
	waitGroup.Add(3)

	go func() {
		waitGroup.Wait()
		close(c)
	}()

	// Transaction #1, merge "commit" records
	go func() {
		defer waitGroup.Done()

		err := mergeCommits(primaryAccount, secondaryAccount)
		if err != nil {
			c <- err
		}
  }()

  // Transaction #2, merge "pull request" records
	go func() {
		defer waitGroup.Done()

		err := mergePullRequests(primaryAccount, secondaryAccount)
		if err != nil {
			c <- err
		}
  }()

  // Transaction #3, merge "merge" records
	go func() {
		defer waitGroup.Done()

		err := mergePullRequestMerges(primaryAccount, secondaryAccount)
		if err != nil {
			c <- err
		}
  }()

  // This line is bad! Get rid of it!
  // waitGroup.Wait()

  for err := range c {
		if err != nil {
			return err
		}
  }

  return markMerged(primaryAccount, secondaryAccount)
}
```

Now, we're ensuring the following behavior:

* Run a Go routine that is blocking, via a call to `waitGroup.Wait()`, until the wait group count is down to zero
* Each "DB transaction" Go routine's anonymous function will, if there is an error, send a message to the channel
* The call to `range` is listening for such messages, and reading them as they arrive
* Back in the Go routine, the function is un-blocked, and the call to `waitGroup.Done()` will run, decrementing the wait group's count
* Once the wait group's count hits zero, the first Go routine will un-block and close the channel via a call to `close(c)`
* This will tell the `range` call to stop listening to the channel, and therefore stop blocking, allowing the main function to continue execution

## Conclusion

What havoc one misplaced `waitGroup.Wait()` can wreak! The key takeaway here is that when you send a message to a channel, it will block until that message is read. We need to ensure that we're reading from any channels we're writing to, in order to successfully synchronize your Go routines.

In the case of our bug, we were improperly blocking the execution of our function, preventing any messages from getting read from the channel. By removing our extra `waitGroup.Wait()`, we guaranteed that messages sent the channel would be read, allowing the `waitGroup`'s counter to be decremented. This in turn ensured that the channel would be closed, causing the range over that channel's messages to unblock, and allowing the main function to continue executing.

This bug certainly gave me a better understanding of Go routine synchronization, and I hope it was helpful to you too.

Happy coding!