# Elixir Test Mocking with Mox
How we successfully mocked HTTPoison, used our mocks in GenServer processes and more!

## Why We Need Mox
In a [recent post](https://medium.com/flatiron-labs/rolling-your-own-mock-server-for-testing-in-elixir-2cdb5ccdd1a0), we talked about the age-old question: How can you test code that relies on external services, like an API? We don't want slow-running tests that make web requests to external services, potentially impacting things like API rate limits, production data and the overall reliability of our test suite.

This consideration often leads us to reach for a test mock. However, we want to be careful about how we think about "mocking". When we mock certain interactions by creating unique function stubs in a given test example, we establish a dangerous pattern. We couple the run of our tests to the behavior of a particular dependency, like an API client. We avoid defining shared behavior among our mocking code/stubbed functions. We make it harder to iterate on our tests.

Instead, [Jose Valim advocates](http://blog.plataformatec.com.br/2015/10/mocks-and-explicit-contracts/) that we change the way we thinking about test mocks; that we think about a mock as a noun, instead of a verb:

> Mocks are simulated entities that mimic the behavior of real entities in controlled ways…I always consider “mock” to be a noun, never a verb.

This reconceptualization has us identifying way to create a mock entity that our app is configured to use in the test environment, rather than writing code that performs a mock or stubbed function in a given test example.

In our first attempt at applying this advice to tests for a GitHub client app that we built in Elixir, [we hand-rolled our own test server](https://medium.com/flatiron-labs/rolling-your-own-mock-server-for-testing-in-elixir-2cdb5ccdd1a0). Our test server acted as a stand-in for the GitHub API in our test environment and it implemented a controller that responded with the expected payloads under the given conditions. Our test server became our mock.  

This approach had the drawback of obscuring the behavior of our mock. In order to write new tests that relied on GitHub API interactions, a developer would have to be aware of the test controller and comb through the test controller code to find the payload that their code is expected to operate on in a given test example.

The existence of a GitHub API mock server also allowed us to avoid defining an explicit contract for the api client that we were testing. Since we never had to mock the api client itself, it wasn't strictly necessary to define an explicit interface for the behavior of that client.

By leveraging the Mox library to define mocks for our api client, we were able to mitigate both of these drawbacks. This resulting in easy-to-read and easy-to-iterate on tests that allow each developer to define their own expectations against the mock client. It also forced us to define an explicit set of behaviors for our api client, in order for Mox to mock it.

Keep reading to see how we did it, and learn about some interesting "gotchas" we faced along the way!

## Getting Started

Our very first "gotcha" came when we installed Mox and tried to use it in one of the child apps of our Elixir umbrella app. We added the Mox dependency to the `mix.exs` file of the `GithubClient` child app:

```elixir
 {:mox, "~> 0.5.0", only: :test}
```

Then, when we defined a mock and called on it in our test suite (more on that soon), we came across this error:

```elixir
** (exit) exited in: GenServer.call(Mox.Server, {:verify_on_exit, #PID<0.353.0>}, 30000)
         ** (EXIT) no process: the process is not alive or there's no process currently associated with the given name, possibly because its application isn't started
```

This told us that the Mox application was _not_ getting started when our app started up in the test environment. Although we weren't explicitly calling `start_link/1` on the Mox application, we understood that Mix will attempt to start every dependent app listed in our `dependencies` function that responds to `start_link`.

This is true, _unless_ you define an explicit `applications` function in your Mix file. In that case, Mix will only start the applications listed there. So, if you have such a function, remember to add `:mox` to the list.

## Mocking our Github Api Client

### Defining a Behaviour

In order to use Mox to create a mock of a given module, that module must be an [Elixir behaviour](https://elixirschool.com/en/lessons/advanced/behaviours/). Behaviours allow us to define a specific API that a set of modules _must_ adhere to. We need to turn the module we want to create a mock out of into a behavior so that Mox can use the behavior to build the mock.

The module responsible for talking to the GitHub API is `GithubClient.ApiClient`. We want to write tests for areas of our code that use `GithubClient.ApiClient`. So, we need to be able to define a mock for this module. Let's take a look at our module:

```elixir
defmodule GithubClient.ApiClient do
  use HTTPoison.Base
  alias GithubClient.GithubClientError

  defp access_token, do: Application.get_env(:github_client, :gh_token)
  def base_url, do: Application.get_env(:github_client, :base_url)

  def find_or_create_organization(org_name, org_owner) do
    case find_organization(org_name) do
      {:ok, org} -> {:ok, org}
      {:error, "Not Found"} -> create_org(org_name, org_owner)
    end
  end

  def find_organization(org_name) do
    "/orgs/#{org_name}"
    |> do_get
  end

  def create_org(org_name, admin_name) do
    "/admin/organizations"
    |> do_post(%{admin: admin_name, login: org_name})
  end

  defp handle_response(resp) do
    case resp do
      {:ok, %{body: body, status_code: 200}} ->
        {:ok, body}
      {:ok, %{body: body, status_code: 201}} ->
        {:ok, body}
      {:ok, %{body: body, status_code: 204}} ->
        {:ok, body}
      {:ok, %{body: body, status_code: 301}} ->
          {:ok, body}
      {:ok, %{body: body, status_code: 404}} ->
        {:error, create_error_message(body)}
      {:ok, %{body: body, status_code: 422}} ->
        {:error, create_error_message(body)}
      {:ok, %{body: body, status_code: status}} ->
         github_error(status, body["message"])
      {:error, error} ->
        github_error(error.id, error.reason)
    end
  end

  def create_error_message(body = %{"errors" => errors}) when is_list(errors) do
    body["message"] <> " (" <> List.first(errors)["message"] <> ")"
  end

  def create_error_message(body) do
    body["message"]
  end

  def process_url(url) do
    base_url() <> url
  end

  defp headers do
     [
       {"Authorization", "token #{access_token()}"},
       {"Accept", " application/json"},
       {"Content-Type", "application/json"}
     ]
  end

  def process_response_body(""), do: ""
  def process_response_body(body) do
    body
    |> Jason.decode!
  end

  defp github_error(status, "") do
    raise GithubClientError, message: "#{status}"
  end

  defp github_error(status, message) do
    raise GithubClientError, message: "#{status}, #{message}"
  end

  defp do_post(url, params) do
    url
    |> post(Jason.encode!(params), headers())
    |> handle_response
  end

  defp do_get(url) do
    url
    |> get(headers())
    |> handle_response
  end
end
```

*Note: This is an abbreviated look at the api client module, only including a handful of functions for finding/creating GitHub orgs*.

Our module uses `HTTPoison` to enact web requests to the GitHub API. It also wraps up the logic around how to make certain requests and parse the responses from those requests.

Let's define a behaviour that we can use in this module:

```elixir
defmodule GithubClient.ApiClientBehaviour do
  @moduledoc false
  @callback find_or_create_organization(String.t(), String.t()) :: tuple()
  @callback create_org(String.t(), String.t()) :: tuple()
  @callback find_organization(String.t()) :: tuple()
end
```

Now that we have our behaviour, we'll tell our api client module to use it:

```elixir
defmodule GithubClient.ApiClient do
  @behaviour GithubClient.ApiClientBehaviour
  ...
end
```

Now that our api client module implements a behavior, Mox can use that behavior to create a mock.

### Defining a Mock
Telling Mox to define a mock for a given behavior is pretty straightforward. We'll define our mock in `github_client/test/support/mocks.ex`

```elixir
Mox.defmock(ApiClientBehaviourMock, for: GithubClient.ApiClientBehaviour)
```

Note that we are mocking the behavior directly, _not_ the module that uses it.

*Note: In order for the `test/support/mocks.ex` file to be compiled by our application, we need to add the following to our mix file:*

```elixir
# mix.exs
...
def project do
  [
    app: :github_client,
    version: "0.1.0",
    build_path: "../../_build",
    config_path: "../../config/config.exs",
    deps_path: "../../deps",
    lockfile: "../../mix.lock",
    elixir: "~> 1.8",
    elixirc_paths: elixirc_paths(Mix.env()),
    start_permanent: Mix.env() == :prod,
    deps: deps()
  ]
end

# Specifies which paths to compile per environment.
defp elixirc_paths(:test), do: ["lib", "test/support"]
defp elixirc_paths(_), do: ["lib"]
...
```

Now that our mock behavior is defined, we need to teach our application to use the mock in the test environment, _instead_ of the real api client.

Let's say we have the following function in our `GithubClient` module that calls on `GithubClient.ApiClient`:

```elixir
defmodule GithubClient do
  alias GithubClient.ApiClient
  def create_lesson_repo(org_name, repo_name, username, team_name) do
    with {:ok, %{team: team}} <- ApiClient.find_or_create_org(org_name, username, team_name),
         {:ok, repo}          <- ApiClient.create_repo(org_name, repo_name),
         {:ok, ""}            <- ApiClient.add_repo_to_team(org_name, team["id"], repo_name)
    do
      {:ok, %Repo{id: repo["id"], url: repo["html_url"]}}
    else
      err -> err
    end
  end
end
```

*Note: This function calls on some `ApiClient` functions not included in our abbreviated `ApiClient` module or behavior. We've excluded their definitions for brevity*.

Our `GithubClient` should be configured to grab the `GithubClient.ApiClient` in the dev and production environments, and the `ApiClientBehaviourMock` in the test environment.

In our `config/dev.exs`:

```elixir
config :github_client, api_client: GithubClient.ApiClient
```

And in our `config/test.exs`

```elixir
config :github_client, api_client: ApiClientBehaviourMock
```

We'll teach the `GithubClient` module to grab the correctly configured api client from the environment, instead of calling on `ApiClient` directly:


```elixir
defmodule GithubClient do
  def api_client, do: Application.get_env(:github_client, :api_client)

  def create_lesson_repo(org_name, repo_name, username, team_name) do
    with {:ok, %{team: team}} <- api_client().find_or_create_org(org_name, username, team_name),
         {:ok, repo}          <- api_client().create_repo(org_name, repo_name),
         {:ok, ""}            <- api_client().add_repo_to_team(org_name, team["id"], repo_name)
    do
      {:ok, %Repo{id: repo["id"], url: repo["html_url"]}}
    else
      err -> err
    end
  end
end
```

Now `GithubClient` will use the mock in the test environment, and we're ready to define some expectations!

### Setting Expectations Against The Mock

Let's say we have the following test for the function above:

```elixir
defmodule GithubClientTest do
  use ExUnit.Case
  alias GithubClient.Repo

  @repo_name "intro-to-ruby"
  @org_name "flatiron-labs"
  @author_username "AuthorUsername"

  describe "create_lesson_repo" do
    test "it creates a repo in github when given valid params" do
      result = GithubClient.create_lesson_repo(@org_name, @repo_name, @author_username)
      assert {:ok, %GithubClient.Repo{url: "https://github.com/#{@org_name}/#{@repo_name}"}} == result
    end
  end
end
```

We already know that `GithubClient` will call on our `ApiClientBehaviourMock` when the tests run. So, we'll need to set some expectations against this mock that will allow our tests to run. In order for this happy-path test to run, we'll need to tell our mock to expect to receive certain input when `find_or_create_org/3` is called, and to return some valid response. *Note: We won't go into mocking the other functions called on our client here, instead we'll just focus on this one example*.


```elixir
defmodule GithubClientTest do
  use ExUnit.Case
  alias GithubClient.Repo
  import Mox

  setup :verify_on_exit!

  @repo_name "intro-to-ruby"
  @org_name "flatiron-labs"
  @author_username "AuthorUsername"
  @org %{"id" => "org456", "login" => @org_name}

  describe "create_lesson_repo" do
    test "it creates a repo in github when given valid params" do
      ApiClientBehaviourMock
      |> expect(:find_or_create_organization, fn _org_name, _username ->
        {:ok, @org}
      end)
      result = GithubClient.create_lesson_repo(@org_name, @repo_name, @author_username)
      assert {:ok, %GithubClient.Repo{url: "https://github.com/#{@org_name}/#{@repo_name}"}} == result
    end
  end
end
```

We expect `ApiClientBehaviourMock` to receive the `find_or_create_organization/3` function with any args and return the tuple, `{:ok, @org}`. And that's it!

## Refactoring our Api Client to Mock HTTP Interactions
Defining a behavior for `GithubClient.ApiClient` allowed us to create a mock we could use when testing code that calls on this api client module. But what about testing the api client itself? After all, this module implements a fair amount of its own logic. So, how can we test the logic in our api client module, but still create a mock to handle the actual HTTP interactions?

We'll refactor `GithubClient.ApiClient` to abstract away the code that actually makes web requests. By separating out the concerns of GitHub API-specific logic from the responsibility of enacting HTTP requests, we end up with cleaner code that we can easily mock in our test suite.

First, we'll define a new module responsible for using `HTTPoison` to make HTTP requests:

```elixir
defmodule GithubClient.HttpAdapter do
  use HTTPoison.Base

  defp access_token, do: Application.get_env(:github_client, :gh_token)
  def base_url, do: Application.get_env(:github_client, :base_url)

  def process_url(url) do
    base_url() <> url
  end

  def post(url, params) do
    post(url, Jason.encode!(params), headers())
  end

  def get(url) do
    get(url, headers())
  end

  defp headers do
     [
       {"Authorization", "token #{access_token()}"},
       {"Accept", " application/json"},
       {"Content-Type", "application/json"}
     ]
  end

  def process_response_body(""), do: ""
  def process_response_body(body) do
    body
    |> Jason.decode!
  end
end
```

Next, we'll remove HTTP-specific code from `GithubClient.ApiClient` and teach it to use our new adapter module instead:

Let's start by removing anything specific to HTTP from our `GitHubClient.ApiClient`

```elixir
defmodule GithubClient.ApiClient do
  alias GithubClient.GithubClientError
  alias GithubClient.HttpAdapter
  def find_or_create_organization(org_name, org_owner) do
    case find_organization(org_name) do
      {:ok, org} -> {:ok, org}
      {:error, "Not Found"} -> create_org(org_name, org_owner)
    end
  end

  def find_organization(org_name) do
    "/orgs/#{org_name}"
    |> do_get
  end

  def create_org(org_name, admin_name) do
    "/admin/organizations"
    |> do_post(%{admin: admin_name, login: org_name})
  end

  defp handle_response(resp) do
    case resp do
      {:ok, %{body: body, status_code: 200}} ->
        {:ok, body}
      {:ok, %{body: body, status_code: 201}} ->
        {:ok, body}
      {:ok, %{body: body, status_code: 204}} ->
        {:ok, body}
      {:ok, %{body: body, status_code: 301}} ->
          {:ok, body}
      {:ok, %{body: body, status_code: 404}} ->
        {:error, create_error_message(body)}
      {:ok, %{body: body, status_code: 422}} ->
        {:error, create_error_message(body)}
      {:ok, %{body: body, status_code: status}} ->
         github_error(status, body["message"])
      {:error, error} ->
        github_error(error.id, error.reason)
    end
  end

  def create_error_message(body = %{"errors" => errors}) when is_list(errors) do
  end

  def create_error_message(body) do
  end

  defp github_error(status, "") do
  end

  defp github_error(status, message) do
  end

  defp do_post(url, params) do
    HttpAdapter.post(url, params)
    |> handle_response
  end

  defp do_get(url) do
    HttpAdapter.get(url)
    |> handle_response
  end
end
```

Our api client no longer directly uses `HTTPoison`, it doesn't know how to format request headers or request URLs and it doesn't have to deal with any JSON processing. It is _only_ in charge of kicking off requests to the appropriate GitHub API endpoint and formatting the response so that it can be consumed elsewhere in the application. Our refactored module is much cleaner and more adheren the the Single Responsibility Principle.

### Mocking HTTP Interactions
Now that we have a separate `GithubClient.HttpAdapter` module that we can mock, we're ready to take a look at some `GithubClient.ApiClient` tests.

Let's say we have the following test:

```elixir
defmodule GithubClient.ApiClientTest do
  alias GithubClient.ApiClient
  use ExUnit.Case

  @username "author-guy"
  @org_id "1234"
  @org_name "flatiron-labs"
  test "finds the org when the org exists" do
    result = {:ok, %{"id" => @org_id, "login" => @org_name}}
    assert result == ApiClient.find_or_create_organization(@org_name, @username)
  end
end
```

When this test runs, `GitHubClient.ApiClient` will call `GithubClient.HttpAdapter.get/1`, which will in turn call on `HTTPoison.Base`'s `get/2` function. So we need to define a mock for our adapter's behavior and teach `GithubClient.ApiClient` to use the mock in the test environment, instead of calling on the adapter directly.

But wait, you might be thinking. We haven't implemented a behaviour for our adapter module! Our adpater module _uses_ `HTTPoison.Base` which implements *it's own* behaviour. So, we don't need to define our own! Instead, we will create a mock fo `HTTPoison` directly:

```elixir
# test/support/mocks.exs
...
Mox.defmock(HttpMock, for: HTTPoison.Base)
```

Now, we have to configure our application environments to be aware of the mock:

```elixir
# config/dev.exs
...
config :github_client, http_adatper: GithubClient.HttpAdapter
```

```elixir
# config/dev.exs
...
config :github_client, http_adatper: HttpMock
```

And teach `GithubClient.ApiClient` to grab the correct adapter from the application environment:

```elixir
defmodule GithubClient.ApiClient do
  alias GithubClient.GithubClientError
  def http_adapter, do: Application.get_env(:github_client, :http_adapter)

  ...

  defp do_post(url, params) do
    http_adapter().post(url, params)
    |> handle_response
  end

  defp do_get(url) do
    http_adapter().get(url)
    |> handle_response
  end
end
```

Now we're ready to define expectations against our mock in our test example:

```elixir
defmodule GithubClient.ApiClientTest do
  alias GithubClient.ApiClient
  use ExUnit.Case
  import Mox
  setup :verify_on_exit!

  @username "author-guy"
  @org_id "1234"
  @org_name "flatiron-labs"
  @org %{"id" => @org_id, "login" => @org_name}
  test "finds the org when the org exists" do
    HttpMock
    |> expect(:get, fn _url ->
      {:ok, %{status_code: 200, body: @org}}
    end)
    result = {:ok, @org}
    assert result == ApiClient.find_or_create_organization(@org_name, @username)
  end
end
```

And that's it! By identifying a division within our original api client, and separating out the GitHub API logic from the code that enacts HTTP requests, we were able to write clean, well-organized code that was easy to mock and test. 


## Using Mox with GenServers
(MERYL)
When we ran tests that set an expectation against a mock, and then kicked off code that would call on the mock in a GenServer process, we got this terrible error:
Here’s how we fixed it and why this works.
We could use this as an opportunity to take a deeper dive into what these lines of code do for us:
use ExUnit.Case, async: true
import Mox
setup :verify_on_exit!
setup :set_mox_global
## Any other “gotchas”?
(MERYL)
## Conclusion
(SOPHIE/MERYL)
