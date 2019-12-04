# Elixir TIL: Running an Umbrella Child App's Test

If you're working on an Elixir umbrella app with multiple children, then you know that running the tests for the entire umbrella app isn't always ideal. It could take a while to run and it can be hard to zero in on one particular set of failures when deving on just of the child apps.

So, how can you run tests for just one specific child app?

### Not Like This

My first attempt to run just one child app's test went a little something like this:

```bash
mix test test/github_client_test.exs
The database for Deployer.Repo has already been created
==> learn_client
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
==> course_suite_client
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
==> github_client
....

Finished in 0.2 seconds
4 tests, 0 failures

Randomized with seed 395181
==> deployer
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
==> deployer_web
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
```
From the root directory, I ran:

```elixir
mix test test/github_client_test.exs
```

This _technically_ works, but its not quite what we want. Any `mix` command we run from the root of our umbrella app is recursively run from the room of each child app in the `apps/` directory. So, while this _will_ run the `github_client_test.exs` test when run in the child app that contains that test, it will also try to run that same tests in *every other child app*. Resulting in these not-so-nice error messages:

```bash
Paths given to `mix test` did not match any directory/file:
test/github_client_test.exs
```

Not ideal.

### Like This
After some googling around I eventually found a response by Jose Valim to a GitHub issue that pointed me in the right direction.

To run _all_ of the tests for just one specific child app, we can run the following from the room of the umbrella app:

```elixir
mix cmd --app child_app_name mix test --color
```

We use `mix cmd --app` to specific the app within which we want to run a given `mix` command. Then we specify the `mix test` command with the given spec we want to run. Additionally, we can specify a test file and even line number to run:

```elixir
mix cmd --app child_app_name mix test test/child_app_name_test.exs:8 --color
```

Where the `mix test` command is followed by the path to the test file you want to run, _from the root of that child app_.

The `--color` flag is important. Without it, we don't get that nice red/green color highlighting, leaving our test output pretty hard to read.

#### Bonus

Typing this command every time we want to run a given child app's tests is kind of a pain. We can define a mix alias to make our lives a little easier. A [mix alias](https://hexdocs.pm/mix/Mix.html#module-aliases) provides us a way to define a custom mix task that will only be available locally, _not_ in packaged versions of our application, i.e. not to devs who install our app as a dependency.

We'll define a mix alias for our child app test command:

```elixir
# mix.exs for root of umbrella app

def project do
    [
      aliases: aliases(),
      ...
    ]
  end

def aliases do
  [
    child_app_name_test: "cmd --app child_app_name mix test --color"
  ]
end
```

Then from root of umbrella app, we can run:

```bash
mix child_app_name_test
```

Or, to run a specific test:

```bash
mix child_app_name_test test/child_app_name_test.exs:6
```

And that's it! A nice, easy-to-use command for running a child app's specs. You could define one such alias for each child app in your umbrella and run those tests with ease.
