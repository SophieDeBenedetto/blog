# Eloquent Control Flow and Efficient Time Complexity with Elixir
While tackling the Day 1 challenge from this year's Advent of Code in Elixir, I was reminded of some of the many ways that Elixir let's us write concise, efficient, and eloquent code. In this post, I break down my solution and dive into how you can use recursion, pattern matching and custom guard clauses to implement even complex logic and control flow in an easy-to-reason about way that also avoids common time complexity pitfalls.

## Advent of Code Challenge
[Advent of Code](https://adventofcode.com/2020/about) is...

> ...an Advent calendar of small programming puzzles for a variety of skill sets and skill levels that can be solved in any programming language you like. People use them as a speed contest, interview prep, company training, university coursework, practice problems, or to challenge each other.

You can complete Advent of Code challenges in any language and submit your answers to "win" stars :star: :star: :star:

It's a fun way to play around with a new language that you're just starting to learn or to refine your skills in a language that you're already familiar with.

After being ~pestered~ kindly reminded about it by [someone who is definitely not annoying](https://hostiledeveloper.com/), I decided to try out the Day 1 puzzle in Elixir.

I had a lot of fun putting my solution together and, unsurprisingly, I found that Elixir's features allowed me to implement even complex logic and control flow in a way that was remarkably concise, as well as efficient with regards to time complexity. Keep reading to see how to leverage pattern matching, recursion and custom guard clauses to write some super clean and extremely elegant Elixir code. :soap: :star: :soap:

## The Prompt
The Day 1 Advent of Code prompt can be simply stated as:

> Find the two elements in a given list that sum to 2020 and return the product of those two numbers.

> For example, of the following list: `[979, 1721, 366, 299, 675, 1456]`,

> `1721 + 299 == 2020`, and `1721 * 299 = 514579`.

> So, your code should return `514579`

## Attempt 1: Lots of Iterating
Before writing any code, I attempted to conceptualize my approach. The first conceptualization that jumped out at me was heavily reliant on iteration and went something like this:

* Iterate over the list
* For each element in the list, iterate over the remaining elements in the list
* For each pair of elements, add them up. If the sum equals 2020, stop.
* If the sum does not equal 2020, keep going

So, taking the example of our list from above, it would look something like this:

* Does 979 + 1721 = 2020? Nope!
* Does 979 + 366 = 2020? Nope!
* Does 979 + 299 = 2020? Nope!
* Does 979 + 675 = 2020? Nope!
* Does 979 + 1456 = 2020? Nope!

Okay, move on to the second element in the list...
* Does 1721 + 366 = 2020? Nope!
* ...

This approach uses nested iteration and is not very efficient. For each item in the list you have to iterate over the remainder of the list and execute the check to see if the two numbers sum to `2020`. In other words, the _outer loop_ executes N times, once for each element in the list. And every time the outer loop executes, the inner loop executes M times, where M is however many steps it must complete to check the current outer loop element against the remaining list elements. As a result, the "check to see if the two numbers sum to `2020`" statement executes a total of N * M times.

So, the number of operations your code has to do will grow exponentially for each element added to the list. This represents a high degree of time complexity. Thanks to Elixir, we can do better.

We'll use recursion and pattern matching to avoid the need to perform 💸 💸 💸 expensive nested iterations 💸 💸 💸. Keep reading to find out how!

## Attempt 2: Pattern Matching and Recursion
Once I recognized the time complexity of the "lots of iterating" approach, I knew I needed to cut down on iterations. Luckily, Elixir provides us a way to pull elements from a list without iterating over that list--pattern matching. In the next section, we'll use pattern matching and recursion to peel off list elements and perform out "sum to `2020`" check on them.

## Efficient Code with Pattern Matching
First, let's walk through how Elixir's pattern matching can be applied to list elements such that we can perform our "check if sum is `2020`" statement against _all_ of the list elements,  _without_ iterating. This will give us the ability to solve our Advent of Code problem with code that is not overly time-complex.

### Pattern Matching List Elements: The Concept

In order to illustrate how we can use pattern matching in this way, let's focus on the goal of checking to see if the first element in our list, plus any other element in the list, equals `2020`.

With Elixir's pattern matching, we can separate out list elements into a "head", i.e. the first element, and a "tail", i.e. everything after the first element.

Something like this:

```elixir
iex> [head | tail] = [1, 2, 3]
iex> head
1
iex> tail
[2, 3]
```

Using this approach, we can match a variable to the first list element, the second list element, and then everything else like this:

```elixir
iex> list = [979, 1721, 366, 299, 675, 1456]
[979, 1721, 366, 299, 675, 1456]
iex> [first | [second | rest] = tail] = list
[979, 1721, 366, 299, 675, 1456]
iex> first
979
iex> second
1721
iex> rest
[366, 299, 675, 1456]
iex> tail
[1721, 366, 299, 675, 1456]
```

In this way, we can establish a variable, `first`, set equal to the first list element, and another variable, `second`, bound to the value of the second list element.

Then, we can check if the sum of these two numbers equals `2020`. If so, great! We're done.

If _not_, we can construct a new list using the _same_ first element, and the list remainder stored in `rest`, cutting out the second element entirely.

Like this:

```elixir
iex> new_list = [first | rest]
[979, 366, 299, 675, 1456]
```

Now, we can repeat the step above to see if the first element in the list plus the _new_ second element in the list equals `2020`:

```elixir
iex> [first | [second | rest]] = new_list
[979, 366, 299, 675, 1456]
iex> first
979
iex> second
366
```

From here, we repeat the process. Does `first + second == 2020`? If so, great! We're done. If not...construct a new list using the same first element, and the list remainder stored in `rest`.

Eventually, if the first element cannot be added to any other list element to get the sum of `2020`, then we end up with a list that contains only one element. We'll have cut out all the other elements until only the first element remains.

What do we want to do then? We want to revisit the _initial starting list_ and pick up with the _second_ element there. The tail of the original list will be a list that _starts_ with the original list's second element.

In other words, if our original list read `[979, 1721, 366, 299, 675, 1456]`, and we didn't find any other number added to `979` to equal `2020`, then the tail of that list should read:

```elixir
[1721, 366, 299, 675, 1456]
```

We already matched a variable, `tail` to the original list's tail above, like this:

```elixir
 iex> list = [979, 1721, 366, 299, 675, 1456]
[979, 1721, 366, 299, 675, 1456]
iex> tail
[1721, 366, 299, 675, 1456]
```

So, using the list stored in the `tail` variable, we can simply repeat the process described above.

Now that we have a basic understanding of what we need to do, let's write some code.

### Pattern Matching List Elements: The Code

We'll define a module `Accountant`, that implements one public function, `product_of_equals_twenty_twenty`. This function will take in a list and return the product of the two numbers in the list that sum to `2020`. The public interface of our module will work like this:

```elixir
Accountant.product_of_equals_twenty_twenty([979, 1721, 366, 299, 675, 1456])
=> 514_579
```

Let's begin implementing it now.

```elixir
defmodule Accountant do
  @sum 2020

  def product_of_equals_twenty_twenty([_head | tail] = list) do
    product = get_two(list)
    # coming soon!
  end
end
```

The function head will use pattern matching to pull out the tail of the list and save it for later in a variable, `tail`. Then it will pass the list to a helper function that is responsible for stepping through the process we described above. Let's revisit that process now by taking a closer look at `get_two/1`.

### Pattern Matching Function Heads
We'll implement a few versions of the `get_two/1` function that use pattern matching in the function head to determine how to behave. We'll also see recursion make a guest appearance. Let's take a look!

```elixir
defmodule Accountant do
  @sum 2020
  # ...
  defp get_two(list) when length(list) == 1, do: nil

  defp get_two([first | [second | _rest]]) when first + second == @sum do
    first * second
  end

  defp get_two([first | [_second | rest]]) do
    get_two([first | rest])
  end
end
```

Let's examine each of the versions of this function. Some of this code should look familiar from our discussion of pattern matching list elements above. One function version pattern matches the first and second elements of the list and uses a guard clause to check if they sum to `2020`. More on guard clauses in a bit.

```elixir
defp get_two([first | [second | _rest]]) when first + second == @sum do
  first * second
end
```

If the first and second list element do sum to `2020`, then the function body will execute and return the product of the two numbers.

If the first and second list element do _not_ sum to `2020`, i.e. if our guard clause does not evaluate to `true`, then we hit the next version of the function implementation:

```elixir
defp get_two([first | [_second | rest]]) do
  get_two([first | rest])
end
```

Here, we build a _new_ list constructed from the first list element and the _remainder_ of that list, minus the second element. That new list is given as an argument to a recursive call to `get_two/1`. This will continue until we either hit the guard clause and return the product of the two elements. Or, until we have removed every element after the first one, resulting in a list with a length of `1`. In that case, we will return `nil`:

```elixir
defp get_two(list) when length(list) == 1, do: nil
```

So, we have a function that, when invoked with a given list, will call itself recursively until it finds two elements that sum to `2020`--in which case it returns their product--or until there is only one list element left--in which case it returns `nil`.

By calling `get_two/1` with the list `[979, 1721, 366, 299, 675, 1456]`, we will have successfully checked to see if `979` plus any other number in the list equals `2020`.

This brings us back to the public function, `product_of_equals_twenty_twenty/1`. If this first invocation of `get_two/1` returns something that is not `nil`, then we're done! We found the product of the two numbers that sum to `2020`. Let's implement some logic to that effect now:

```elixir
defmodule Accountant do
  @sum 2020

  def product_of_equals_twenty_twenty([_head | tail] = list) do
    product = get_two(list)
    if product do
      product
    else
      # keep going
    end
  end
end
```

If the first `get_two/1` invocation does _not_ return a number and does return `nil`, we need to start the whole process again, this time with the _tail end_ of our original list.


```elixir
defmodule Accountant do
  @sum 2020

  def product_of_equals_twenty_twenty([_head | tail] = list) do
    product = get_two(list)
    if product do
      product
    else
      product_of_equals_twenty_twenty(tail)
    end
  end
end
```

This will restart the process, this time with a list that reads: `[1721, 366, 299, 675, 1456]`. Once again, our code will try to sum each number with the first list element, this time `1721`. If it returns some product, then we're done! If not, we'll keep invoking `product_of_equals_twenty_twenty/1` with the next list tail, and the next, until we find what we're looking for.

To wrap up this code, we'll add a function head for `product_of_equals_twenty_twenty/1` that can handle being invoked with an empty list--that is what will happen if our list does not contain any two numbers that equal `2020` and we continue to invoke `product_of_equals_twenty_twenty/1` until we have a tail that is empty.

```elixir
defmodule Accountant do
  @sum 2020
  def product_of_equals_twenty_twenty([]), do: :noop
  def product_of_equals_twenty_twenty([_head | tail] = list) do
    product = get_two(list)
    if product do
      product
    else
      product_of_equals_twenty_twenty(tail)
    end
  end
end
```

### Putting It All Together

Putting all together, our code reads:

```elixir
defmodule Accountant do
  @sum 2020
  def product_of_equals_twenty_twenty([]), do: :noop
  def product_of_equals_twenty_twenty([_head | tail] = list) do
    product = get_two(list)
    if product do
      product
    else
      product_of_equals_twenty_twenty(tail)
    end
  end

  defp get_two(list) when length(list) == 1, do: nil

  defp get_two([first | [second | _rest]]) when first + second == @sum do
    first * second
  end

  defp get_two([first | [_second | rest]]) do
    get_two([first | rest])
  end
end
```

Our code works, and it's highly efficient. We're never iterating over the list, never mind iterating over it in a nested fashion. Instead, we are recursively pulling the first and second elements off of a list, and shrinking the list each step of the way.

This looks pretty clean, but I think we can do even better. Anytime I see an `if` condition in Elixir, I wonder if I can replace it with recursion and pattern matching. Elixir allows us to combine recursion and pattern matching into an elegant solution for control flow. Could we implement `product_of_equals_twenty_twenty/1` such that it can handle the case of a "found product"? Let's give it a shot!

## Clean Control Flow with Pattern Matching and Recursion
We'll take a similar approach here to the one we used with our `get_two/1` implementation. A set of function heads will use pattern matching to determine how to behave. One such function will leverage recursion to continue code flow, while other function heads will determine when the code will stop executing and return. In this way, Elixir pairs recursion and pattern matching to implement control flow--without `if` conditions or `while` loops.

Let's take a look.

```elixir
defmodule Accountant do
  @sum 2020

  def product_of_equals_twenty_twenty(list, product \\ nil)
  def product_of_equals_twenty_twenty(_list, product) when product != nil, do: product

  def product_of_equals_twenty_twenty([_head | tail] = list, _product) do
    product_of_equals_twenty_twenty(tail, get_two(list))
  end
  # ...
end
```

We've changed the arity of `product_of_equals_twenty_twenty` to take in two arguments. The second argument will be the product of the two numbers that sum to `2020`, and it will default to `nil`.

So, when our function is invoked with a list,

```elixir
Accountant.product_of_equals_twenty_twenty([979, 1721, 366, 299, 675, 1456])
```

The `product` argument evaluates to `nil`, and we find ourselves in this version of our function:

```elixir
def product_of_equals_twenty_twenty([_head | tail] = list, _product) do
  product_of_equals_twenty_twenty(tail, get_two(list))
end
```

Here, we kick off the process by recursively invoking `product_of_equals_twenty_twenty/2` with the _tail_ of the original list and the result of calling `get_two/1` with the original list.

If `get_two/1` returns a product that is not `nil`, then we'll find ourselves in this other version of the `product_of_equals_twenty_twenty/1` function:

```elixir
def product_of_equals_twenty_twenty(_list, product) when product != nil, do: product
```

In which case, we break out of our recursive function calls, stop code execution, and return the product.

Let's step through this code in detail.

1. The user calls `Accountant.product_of_equals_twenty_twenty([979, 1721, 366, 299, 675, 1456])`

At this point, we hit this function,

```elixir
def product_of_equals_twenty_twenty([_head | tail] = list, _product) do
  product_of_equals_twenty_twenty(tail, get_two(list))
end
```

where `list` evaluates to `[979, 1721, 366, 299, 675, 1456]`, `tail` is set to `[1721, 366, 299, 675, 1456]` and `product` is `nil`.

2. `get_two/1` is called with the `list`
3. Since `979` is _not_ one of the numbers that sums to `2020`, this call returns `nil`.
4. So, we call `product_of_equals_twenty_twenty/2` with `tail` and `nil`
5. This brings us back to this same function body:

```elixir
def product_of_equals_twenty_twenty([_head | tail] = list, _product) do
  product_of_equals_twenty_twenty(tail, get_two(list))
end
```

_This time around_, `list` is equal to `[1721, 366, 299, 675, 1456]`, `tail` is `[366, 299, 675, 1456]` and `product` is still `nil`

6. `get_two/1` is called with the `list`
7. Since `1721` plus `299` _does_ equal `2020`, this will return the result of `1721 * 299`, which is `514_579`
8. So, we call `product_of_equals_twenty_twenty/2` will `tail` and `514_579`.
9. This brings us to this other function body:

```elixir
def product_of_equals_twenty_twenty(_list, product) when product != nil, do: product
```

So, we stop recursing, code execution is done, and the product is returned.

Phew! Seems like a fair amount of complexity, but when we put it altogether, we see some very clean and concise code.

```elixir
defmodule Accountant do
  @sum 2020

  def product_of_equals_twenty_twenty(list, product \\ nil)
  def product_of_equals_twenty_twenty(_list, product) when product != nil, do: product

  def product_of_equals_twenty_twenty([_head | tail] = list, _product) do
    product_of_equals_twenty_twenty(tail, get_two(list))
  end

  defp get_two(list) when length(list) == 1, do: nil

  defp get_two([first | [second | _rest]]) when first + second == @sum do
    first * second
  end

  defp get_two([first | [_second | rest]]) do
    get_two([first | rest])
  end
end
```

In just about a dozen lines of code, we've implemented an efficient, iteration-free solution--all thanks to the beauty of Elixir's pattern matching. By pattern matching list elements, we were able to avoid expensive iterations. By using pattern matching in function heads, along with recursion, we were able to implement control flow that didn't rely on `if` conditions or `while` loops.

Before we go, we'll do just a bit more refactoring for readability with the help of custom guard clauses.

## Refactoring with Custom Guard Clauses

Guard clauses allows us to apply more complex checks to pattern matching function heads. We're using a number of guard clauses throughout our code, for example:

```elixir
defp get_two(list) when length(list) == 1, do: nil
```

Here, we've implement a version of the `get_two/1` function that will execute _if_ the length of the provided `list` argument is equal to `1`.

Guard clauses give us even more fine-grained control over which code to execute under which conditions. This is yet another way that we can implement complex control flow without verbose and hard-to-reason-about `if` and nested `if` conditions.

Only a certain set of expressions are allowed for usage in guard clauses, see docs [here](https://hexdocs.pm/elixir/guards.html#list-of-allowed-expressions). But, we can define custom guard clauses to wrap up more complex guard logic. The guard clause we wrote to check if two numbers sum to `2020` is a great candidate for a custom guard clause. By wrapping up that logic in a custom guard clause, we can name the concept to make it easier to read and reason about.

To define a custom guard clause, we'll use the `defguard` macro:

```elixir
defmodule Accountant do
  @sum 2020
  defguard equals_twenty_twenty(first, second) when first + second == @sum
```

Now, we can use our guard clause like this:

```elixir
defp get_two([first | [second | _rest]]) when equals_twenty_twenty(first, second) do
  first * second
end
```

Custom guard clauses give us the ability to implement even complex control flow with pattern matching function heads, while keeping our code readable.

Putting it all together:

```elixir
defmodule Accountant do
  @sum 2020
  defguard equals_twenty_twenty(first, second) when first + second == @sum

  def product_of_equals_twenty_twenty(list, product \\ nil)
  def product_of_equals_twenty_twenty(_list, product) when product != nil, do: product

  def product_of_equals_twenty_twenty([_head | tail] = list, _product) do
    product_of_equals_twenty_twenty(tail, get_two(list))
  end

  defp get_two(list) when length(list) == 1, do: nil

  defp get_two([first | [second | _rest]]) when equals_twenty_twenty(first, second) do
    first * second
  end

  defp get_two([first | [_second | rest]]) do
    get_two([first | rest])
  end
end
```

## Elixir Encourages Efficient and Eloquent Code
By using Elixir's pattern matching against our list of numbers, we were able to write efficient code that avoided the time complexity of expensive nested iterations.

By using that same pattern matching feature, paired with guard clauses and recursion, we were able to implement complex logic and control flow in a way that is 🧼 clean 🧼 and 🗣 eloquent 🗣. The code speaks for itself by being readable and easy to reason about. No messy, nested `if` conditions to deal with.

This Advent of Code challenge really shows off some of Elixir's simplest, but most powerful features.
