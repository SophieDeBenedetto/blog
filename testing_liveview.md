# Testing LiveView

## Introducing LiveView's Powerful Testing Tools

If you've worked with LiveView, you've already experienced how productive you and your team can be with a framework that let's you build interactive UIs while keeping your brain firmly focused on the server-side. Testing LiveView is no different--you can exercise the full functionality of your live views with pure Elixir tests written in ExUnit with the help of the `LiveViewTest` module. So your tests are fast, concurrent, and continue to keep your focus firmly on server-side code. LiveView's testing framework empowers you to quickly write robust and comprehensive tests that are highly stable. With a high degree of test coverage firmly in hand, you and your team will be able to move quickly and confidently when building out LiveView applications

In this post, we'll explore some more of what makes LiveView testing so powerful, understand the principles for approaching live view tests, write some unit and integration tests, and even test a distributed, real-time update feature in LiveView. Let's get going!

## Principles for Testing LiveView
The basic principles for testing LiveView don't differ a whole lot from testing in other languages and frameworks. Any given test has three steps:

- Set up preconditions
- Provide a stimulus
- Compare an actual response to expectations

In this post, we're going to write two kinds of tests--unit tests and integration tests--and we'll use this procedure for both types. Let's dig a little deeper into what these two kinds of tests look like in LiveView now.

A **unit test** is designed to test an individual unit of code. Pure unit tests call one function at a time, and then check expectations with one or more assertions. Unit tests encourage *depth*. Such tests don't require much ceremony so programmers can write more of them and cover more scenarios quickly and easily. Unit tests also allow *loose coupling* because they don't rely on specific interactions. We'll write one unit test today to verify the behavior of the independent reducer functions that our live view will implement to establish socket state under different conditions.

Integration tests, on the other hand, validate the interactions between different parts of your system. These kind of tests offer testing *breadth* by exercising a wider swath of your application. The cost is tighter coupling, since integration tests rely on specific interactions between parts of your system. Of course, that coupling will exist whether we test it or not. We'll write two integration tests that help us verify the behavior of the overall live view. Our first integration test will validate the interactions within a single live view process, and the second will verify an PubSub-backed interaction *between* two live views.

You might be surprised to hear that none of the tests we'll write today will be testing JavaScript. This is because the LiveView framework is specifically designed to handle JavaScript for us, so we don't have to. For the average live view that _doesn't_ implement any custom JavaScript, there's not need to test JS code or write any tests that use JavaScript. We can trust that the JS in the LiveView framework itself works as expected.

We're just about ready to start writing our very first LiveView test, and we'll start with some unit tests. It's a good idea to start with unit test coverage before moving on to integration testing. By exercising individual functions in unit tests with many different inputs, you can exhaustively cover corner cases. This lets you write a smaller number of integration tests to confirm that the complex interactions of the system work as you expect them to.

So, to recap, we can apply the following principles to LiveView testing:

* Follow the three step procedure of setting up preconditions, providing a stimulus and validating expectations.
* Write both unit and integration tests.
* Don't test JavaScript you didn't write.
* Start by writing comprehensive unit tests.

Alright, with these principles in hand, we're ready to start testing.

## Unit Testing LiveView
In this section, we'll use our three-step testing procedure to write a few unit tests for the behavior of a LiveView component. Along the way, we'll see how composing our LiveView component out of single-purpose reducer functions helps make our code easy to test.

### The Feature
The testing examples that we'll be looking at in this post are drawn from the "Test Your Live Views" chapter in my book, [Programming LiveView](https://pragprog.com/titles/liveview/programming-phoenix-liveview/), co-authored with Bruce Tate. Check it out for an even deeper dive into LiveView testing and so much more.

In this example, we have a an online game store that allows users to browse and review products. We provide our users with a survey that asks them for some basic demographic input, along with star ratings of each game. We have a LiveView component, `SurveyResultsLive`, that displays the results of this survey as a chart that graphs the games and their star ratings, on a scale of 1 to 5. Here's a look at the component UI:

![image]

Notice the age group filter drop down menu at the top of the chart. Users can select an age group by which to filter the survey results. When the page loads, the value of the age group filter should default to "all" and the chart should show all of the results. But when a user selects an age group from the menu, an event will be sent to the live view and the results will be filtered. Before we write our test, let's take a look at the code we're testing.

### The Code

The `update/2` function of the `SurveyResultsLive` component sets the initial state with the help of some single-purpose reducer functions. A reducer function is a function that takes in some input and returns an updated version of that input. We can use single-purpose reducer functions that take in a live view socket, add some state to that socket, and return an updated socket, to build nice clean pipelines for managing state in LiveView. Here's our `update/2` function:

```elixir
  def update(assigns, socket) do
    {:ok,
     socket
     |> assign(assigns)
     |> assign_gender_filter()
     |> assign_age_group_filter()
     |> assign_products_with_average_ratings()
     |> assign_chart_svg()}
  end
```

The details of most of these reducer functions are not important, but we will take a closer look at the `assign_age_group_filter` reducer right now.

```elixir
 def assign_age_group_filter(socket) do
  assign(socket, :age_group_filter, "all")
end
```

Our reducer is pretty simple, it adds a key `:age_group_filter` to the socket and sets it to the default value of "all". However, we need a second version of this function that we can call in the `handle_event/3` that gets invoked when the user selects an age group from the drop down menu. This version of the function should take the selected age group from the event and set _that_ as the value in the `:age_group_filter` key of socket assigns, like this:

```elixir
def handle_event("age_group_filter", %{"age_group_filter" => age_group_filter}, socket) do
  {:noreply,
    socket
    |> assign_age_group_filter(age_group_filter)
    |> assign_products_with_average_ratings()
    #...
    }
end

def assign_age_group_filter(socket, age_group_filter) do
  assign(socket, :age_group_filter, age_group_filter)
end
```

Okay, with a basic understanding of this bit of code in place, we're ready to write our test.

### The Test
First up, we need to implement our test module in a file, `gamestore/test/gamestore/live/survey_results_live_test.exs`, like this:

```elixir
defmodule GamestoreWeb.SurveyResultsLiveTest do
  use ExUnit.Case
  alias GamestoreWeb.SurveyResultsLive
end
```

Here, we're aliasing the `SurveyResultsLive` component to make it easier to refer to in our tests and using the `ExUnit.Case` behavior to gain access to ExUnit testing functionality. We're not importing any other code or even any other testing utilities, since we can write our unit test in pure Elixir without even any database interactions required.

Next up, we'll establish a fixture that we can share across test cases. Every test will assert some expectations about a LiveView socket. So, we'll add a fixture that returns a socket, like this:

```elixir
defmodule GamestoreWeb.SurveyResultsLiveTest do
  use ExUnit.Case
  alias GamestoreWeb.SurveyResultsLive

  defp create_socket(_) do
    %{socket: %Phoenix.LiveView.Socket{}}
  end
end
```

Now that our test module is defined and we've implemented a helper function to create test data (i.e. our socket), we're ready to write our very first test. We'll start with a test that verifies the default socket state when no age group filter is supplied. Open up a describe block and add a call to the `setup/1` function with the a call to the helper function that returns a socket struct, like this:

```elixir
defmodule GamestoreWeb.SurveyResultsLiveTest do
  use ExUnit.Case
  alias GamestoreWeb.SurveyResultsLive

  defp create_socket(_) do
    %{socket: %Phoenix.LiveView.Socket{}}
  end

  describe "Socket state" do
    setup do
      create_socket()
    end
  end
end
```

This ensures the `socket` will be available to all of our test cases. Now, we're ready to write the test itself. Create a test block within the `describe` block that pattern matches the `socket` out of the context we created in the setup function:

```elixir
defmodule GamestoreWeb.SurveyResultsLiveTest do
  use ExUnit.Case
  alias GamestoreWeb.SurveyResultsLive

  defp create_socket(_) do
    %{socket: %Phoenix.LiveView.Socket{}}
  end

  describe "Socket state" do
    setup do
      create_socket()
    end

    test "when no age group filter is provided", %{socket: socket} do
    end
  end
end
```

Let's pause and think through what we're testing here and try to understand what behavior we *expect* to see. This test covers the function `assign_age_group_filter/0`. If it's working correctly, the socket should contain a key of `:age_group_filter` that points to a value of `"all"`. Now that we understand our expectation, let's finish up the test:

```elixir
test "when no age group filter is provided", %{socket: socket} do
  socket =
    socket
    |> SurveyResultsLive.assign_age_group_filter()

  assert
    socket.assigns.age_group_filter == "all"
end
```

Great. We can see that our three-step testing process is represented here like this:

- Set up preconditions by calling the `setup` function to establish our initial socket and make it available to the test case.
- Provide the stimulus by calling the `SurveyResultsLive.assign_age_group_filter/1` function to return a new socket.
- Validate the expectations regarding the new socket's state.

Let's quickly add another test that validates the socket state when an age group filter _is_ provided:


```elixir
test "when age group filter is provided", %{socket: socket} do
  socket =
    socket
    |> SurveyResultsLive.assign_age_group_filter("18 and under")

  assert
    socket.assigns.age_group_filter == "18 and under"
end
```

Simple enough. Let's tackle a more complex scenario. Recall that our `update/2` and `handle_event/3` functions both invoke `assign_age_group_filter` and then pipe the resulting socket into a call to the `assign_products_with_average_ratings/1` function. We won't worry about the details of this function for now, just know that it queries for the product ratings given the filter info that is set in socket assigns. The composable nature of our single-purpose reducer functions let's us write multi-stage unit tests that exercise the behavior of the reducer pipeline as a whole. Let's write a test that validates the product ratings assignment under different filter conditions.

Start by defining a new test and providing it with the `socket` from the setup function, like this:

```elixir
test "ratings are filtered by age group", %{socket: socket} do

end
```

First, we'll set the socket state with a call to `SurveyResultsLive.assign_age_group_filter/1` and validate that the products with average ratings are set correctly:

```elixir
test "ratings are filtered by age group", %{socket: socket} do
  socket =
    socket
    |> SurveyResultsLive.assign_age_group_filter()
    |> SurveyResultsLive.assign_products_with_average_ratings()

  assert
    socket.assigns.products_with_average_ratings == all_products_with_ratings # don't worry about the value of `all_products_with_ratings`, just assume its the expect list of all products with their average ratings
end
```

Great. Next up, we'll update the socket's `:age_group_filter` state with the help of the `assign_age_group_filter/2` reducer function, pipe the updated socket into another call to `assign_products_with_average_ratings/1`, and validate another assertion:

```elixir
test "ratings are filtered by age group", %{socket: socket} do
  socket =
    socket
    |> SurveyResultsLive.assign_age_group_filter()
    |> SurveyResultsLive.assign_products_with_average_ratings()

  assert
    socket.assigns.products_with_average_ratings == all_products_with_ratings # don't worry about the value of `all_products_with_ratings`, just assume its the expect list of all products with their average ratings

  socket =
    socket
    |> SurveyResultsLive.assign_age_group_filter("18 and under")
    |> SurveyResultsLive.assign_products_with_average_ratings()

  assert
    socket.assigns.products_with_average_ratings == filtered_products_with_ratings # don't worry about the value of `filtered_products_with_ratings`, just assume its the expect list of all products with their average ratings provided by users in the specified age group
end
```

This tests works, but its pretty verbose. We can clean it up by taking some inspiration from the clean, readable reducer pipelines we used in the `SurveyResultsLive` component to set state. Let's start by defining a helper function for use in our test, `assert_keys`:

```elixir
defp assert_keys(socket, key, value) do
  assert socket.assigns[key] == value
  socket
end
```

This function performs takes in a socket, performs a test assertion, and then returns the socket. Since it takes in a socket and returns a socket, it behaves just like a reducer. Now, we can use our helper function to string together a beautiful, clean testing pipeline, like this:

```elixir
test "ratings are filtered by age group", %{socket: socket} do
  socket
  |> SurveyResultsLive.assign_age_group_filter()
  |> SurveyResultsLive.assign_products_with_average_ratings()
  |> assert_keys(:products_with_average_ratings, all_products_with_ratings)
  |> SurveyResultsLive.assign_age_group_filter("18 and under")
  |> SurveyResultsLive.assign_products_with_average_ratings()
  |> assert_keys(:products_with_average_ratings, filtered_products_with_ratings)
end
```

Much better. Our usage of small, single-purpose reducer functions not only helped us write a clean, readable and organized live view, it also helped us quickly write comprehensive unit tests. Further, we were able to string together a _test_ pipeline that functions just like the real reducer pipeline in our live view in order to write a unit test that exercises the pipeline's functionality as a whole. You can easily imagine writing additional test cases for the other reducer functions implemented in the LiveView component, as well as additional pipeline tests that exercise the behavior of the reducer pipelines that are used in the `update/2` and `handle_event/3` functions of our component. For both kinds of unit tests, we applied our three-step testing process to end up with clean, readable test cases.

Next up, we'll leverage this same process to write an integration test.

## Integration Testing LiveView

In this section, we'll write an integration test that validates interactions within *a single live view*. We'll focus on testing the the behavior of the survey results chart filter we discussed above. We'll use the `LiveViewTest` module's functions to simulate liveView connections without a browser. Your tests can mount and render live views, trigger events, and then execute assertions against the rendered view. That's the whole LiveView lifecycle. Let's get started.

### The Feature

The `SurveyResultsLive` component we examined in the previous section is rendered within a live view `AdminDashboardLive`, that lives at the `/admin-dashboard` route. Our test will simulate a user's visit to `/admin-dashboard`, followed by their filter selection of the `18 and under` age group. The test will verify an updated survey results chart that displays product ratings from users in that age group.

Because components run in their parent's processes, we'll focus our tests on the `AdminDashboardLive` view, which is the `SurveyResultsLive` component's parent. We'll use `LiveViewTest` helper functions to run our admin dashboard live view and interact with the survey results component. Along the way, you'll get a taste for the wide variety of interactions that the `LiveViewTest` module allows you to test.

Let's begin by setting up a LiveView test for our `AdminDashboardLive` view.

### The Test
It's best to segregate unit tests and integration tests into their own modules, so create a new file `test/gamestore_web/live/admin_dashboard_live_test.exs` and define the module, like this:

```elixir
defmodule GamestoreWeb.AdminDashboardLiveTest do
  use GamestoreWeb.ConnCase

  import Phoenix.LiveViewTest
  alias Gamestore.{Accounts, Survey, Catalog}

  @create_product_attrs %{description: "test description", name: "Test Game", sku: 42, unit_price: 120.5}
  @create_user_attrs %{email: "test@test.com", password: "passwordpassword"}
  @create_user2_attrs %{email: "test2@test.com", password: "passwordpassword"}
  @create_user3_attrs %{email: "test3@test.com", password: "passwordpassword"}
  @create_demographic_attrs %{gender: "female", year_of_birth: DateTime.utc_now.year - 15}
  @create_demographic_over_18_attrs %{gender: "female", year_of_birth: DateTime.utc_now.year - 30}

  defp product_fixture do
    {:ok, product} = Catalog.create_product(@create_product_attrs)
    product
  end

  defp user_fixture(attrs \\ @create_user_attrs) do
    {:ok, user} = Accounts.register_user(attrs)
    user
  end

  defp demographic_fixture(user, attrs) do
    attrs =
      attrs
      |> Map.merge(%{user_id: user.id})
    {:ok, demographic} = Survey.create_demographic(attrs)
    demographic
  end

  defp rating_fixture(user, product, stars) do
    {:ok, rating} = Survey.create_rating(%{stars: stars, user_id: user.id, product_id: product.id})
    rating
  end

  defp create_product(_) do
    product = product_fixture()
    %{product: product}
  end

  defp create_user(_) do
    user = user_fixture()
    %{user: user}
  end

  defp create_demographic(user, attrs \\ @create_demographic_attrs) do
    demographic = demographic_fixture(user, attrs)
    %{demographic: demographic}
  end

  defp create_rating(user, product, stars) do
    rating = rating_fixture(user, product, stars)
    %{rating: rating}
  end
end
```

Let's break this down. First, our test module uses the `GamestoreWeb.ConnCase` behavior. This let's us route to live views using the test connection by giving our tests access to a context map with a key of `:conn` pointing to a value of the test connection. Then, we import the `LiveViewTest` module to give us access to LiveView testing functions. Lastly, we define some fixtures we will use to create our test data. The nitty-gritty details of those fixture functions aren't import. Just understand that they create the database records we need in order to log in users and render the admin dashboard page to display a chart with product ratings.

Now that our module is set up, we'll add a `describe` block to encapsulate the feature we're testing---the survey results chart functionality:

```elixir
describe "Survey Results" do
  setup [:register_and_log_in_user, :create_product, :create_user]

  setup %{user: user, product: product} do
    create_demographic(user)
    create_rating(user, product, 2)

    user2 = user_fixture(@create_user2_attrs)
    create_demographic(user2, @create_demographic_over_18_attrs)
    create_rating(user2, product, 3)
    :ok
  end
  # test coming soon!
end
```

Two calls to `setup/1` seed the test database with a product, users, demographics, and ratings. One of the two users is in the `18 and under` age group and the other is in another age group. Then, we create a rating for each user.

We're also using a test helper provided for us way by the [Phoenix authentication generator](https://github.com/aaronrenner/phx_gen_auth)---`register_and_log_in_user/1`. This function creates a context map with a logged in user, a necessary step because visiting the `/admin-dashboard` route requires an authenticated user.

Now that our setup is completed, we'll define our test:

```elixir
describe "Survey Results" do
  # ...
  test "it filters by age group", %{conn: conn} do
  end
end
```

Before we write the body of our test, let's make a plan. In order to test this feature, we need to:

* Mount and render the live view
* Find the age group filter drop down menu and select an item from it
* Assert that the re-rendered survey results chart has the correct data and markup

This is the pattern you'll always apply to testing live view features. Run the live view, simulate some interaction, validate the rendered result. This pattern should sound familiar--it neatly matches up to the three-step testing process we've been using so far:

- Set up preconditions
- Provide input
- Validate your expectations

Let's begin with the first step, mounting and rendering the LiveView. We'll call the `LiveViewTest.live/2` function which takes in the test `conn` and spawns a simulated live view process:

{:language="elixir"}
~~~
test "it filters by age group", %{conn: conn} do
  {:ok, view, _html} = live(conn, "/admin-dashboard")
end
~~~

The call to `live/2` returns a three element tuple with `:ok`, the LiveView process, and the rendered HTML returned from the live view's call to `render/1`. We don't need to access that HTML in this test, so we ignore it.

Components run in their parent's process. That means the test *must* start up the `AdminDashboardLive` view, rather than rendering _just_ the `SurveyResultsLive` component. By spawning the `AdminDashboardLive` view, we're _also_ rendering the components that the view is comprised of. So, by interacting the the `view` variable representing the `AdminDashboardLive` process above, we'll be able to interact with elements within the `SurveyResultsLive` component and test that it behaves appropriately in response to events. This is the correct way to test LiveView component behavior within a live view page.

The test has a running live view now, so we're ready to provide our input by selecting the `18 and under` age filter. This will require two steps: We need to find the age group filter drop down menu and select an item from it.

We'll use the `LiveViewTest.element/3` function to find the age group drop-down on the page.

Assuming the drop-down menu HTML form element has an ID of `age-group-filter`, we can target it with `element/3` like this:

```elixir
test "it filters by age group", %{conn: conn} do
  {:ok, view, _html} = live(conn, "/admin-dashboard")
  html =
    view
    |> element("#age-group-form")
end
```

`element/3` returns `Phoenix.LiveViewTest.Element` struct that we can pipe into another `LiveViewTest` function in order to simulate the selection of an item and submission of the form:

```elixir
test "it filters by age group", %{conn: conn} do
  {:ok, view, _html} = live(conn, "/admin-dashboard")
  html =
    view
    |> element("#age-group-form")
    |> render_change(%{"age_group_filter" => "18 and under"})
end
```

The `LiveViewTest.render_change/2` function is one of the functions you'll use to simulate user interactions when testing live views. It takes an argument of the selected element, along with some params, and triggers a `phx-change` event. Here, we make sure to call `render_change/2` with the exact params that would be sent to the live view when the user selects an age group filter in the UI.

This event will trigger the associated handler, invoking the reducers that update our socket, and re-rendering the survey results chart with the filtered product rating info.

With our setup and input in place, we're ready to write our assertions. The call to `render_change/2` will return the re-rendered template. Let's add an assertion that the re-rendered chart displays the correct data by validating the presence of an updated title for a product's average rating:

```elixir
test "it filters by age group", %{conn: conn} do
  {:ok, view, _html} = live(conn, "/admin-dashboard")
    view
    |> element("#age-group-form")
    |> render_change(%{"age_group_filter" => "18 and under"})
    |> assert =~ "<title>2.00</title>" # validates that we now display an average rating of 2.00
end
```

Once again, the details of our assertion aren't important, just understand that when the product ratings are filtered by age group, we expect to see this element on the page `<title>2.00</title>`.

And with that, our first integration test is complete! The `LiveViewTest` module provided us with everything we needed to mount and render a connected live view, target elements within that live view---even elements nested within child components---and assert the state of the view after firing DOM events against those elements.

The test code is clean and elegantly composed with a simple pipeline, and all of it is written in Elixir with ExUnit and `LiveViewTest` functions--we didn't need to bring in any JavaScript dependencies. As a result, writing our test was a straightforward process, and we ended up with a test that is easy to read, and that runs fast and reliably.

We only saw a small subset of the `LiveViewTest` functions that support LiveView testing here, but there are many more `LiveViewTest` functions that allow you to send any number of DOM events---blurs, form submissions, live navigation and more. Learn more about them in the docs [here](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveViewTest.html).

Before we go, we'll write one more integration test, this time to exercise interactions between live views.

## Testing Distributed Updates in LiveView

Testing message passing in a distributed application can be painful, but `LiveViewTest` makes it easy to test the PubSub-backed real-time features that you can build into your live views. That is because LiveView tests interact with views via process communication. Since PubSub uses simple Elixir message passing, testing a live view's ability to handle such messages is a simple matter of using `send/2`.

In this section, we'll write an integration test that validates the behavior of the `AdminDashboardLive` when it receives a certain message over PubSub.

### The Feature
Our `AdminDashboardLive` supports the following real-time update feature: When a user anywhere in the world submits a new product rating, then the survey results chart on the admin dashboard page will update to display that new data, in real-time. This is backed by the following code flow: When a user submits a product rating, then a PubSub event, `"rating_created"` is broadcast over a topic. The `AdminDashboardLive` view subscribes to that topic and responds to the event by re-rendering the `SurveyResultsLive` component with updated data from the database. The details of the code aren't important for our purposes today, a high-level understanding is all we need to write our test. Let's get started.

### The Test
Just like in our unit test earlier, we can group similar test cases in a single describe block. We already have a `describe` block in our integration test module for `"Survey Results"`. Since the test of the real-time update feature also describes the behavior of the `SurveyResultsLiveComponent`, we'll add another test case to this same describe block, like this:

```elixir
describe "Survey Results" do
  # ...
  test "it updates to display newly created ratings", %{conn: conn, product: product} do
end
```

This time, our test case retrieves both the test `conn` and the `product` from the setup context. We'll use this product to create a new rating to be displayed.

Once again, before we fill in the body of our test, let's make a plan. We'll follow the same three-step process we used for our earlier integration test:

* Mount and render the connected live view
* Interact with that live view---in this case, by sending the `rating_created` message to the live view
* Re-render the view and verify changes in the resulting HTML

Let's start by mounting and rendering the live view:

```elixir
test "it updates to display newly created ratings", %{conn: conn, product: product}
  {:ok, view, html} = live(conn, "/admin-dashboard")
  assert html =~ "<title>2.50</title>"
end
```

Here, we mount and render the live view with the `live/2` function, and we add an intermediate assertion to check the starting state of the product ratings label--this is the value that we will expect to change once we create a new rating and send the `"rating_created"` message to the live view.

Next up, we need to provide our input. This will be a two step process: First, we'll create a new rating for the product. Then, we'll send the `"rating_created"` message to the live view.

```elixir
test "it updates to display newly created ratings", %{conn: conn, product: product}
  {:ok, view, html} = live(conn, "/admin-dashboard")
  assert html =~ "<title>2.50</title>"

  # create a new user + demographic and then create a new rating with those records
  user3 = user_fixture(@create_user3_attrs)
  create_demographic(user3)
  create_rating(user3, product, 3)
  # send the message to the live view
  send(view.pid, %{event: "rating_created"})
end
```

Here, we can use a simple `send/2` to mimic the code flow of PubSub broadcasting the `"rating_creating"` message. Our live view should respond by re-rendering the `SurveyResultsLive` component with fresh data from the DB, including the newly created rating. All we need to do now is add our assertion:

```elixir
test "it updates to display newly created ratings", %{conn: conn, product: product}
  {:ok, view, html} = live(conn, "/admin-dashboard")
  assert html =~ "<title>2.50</title>"

  # create a new user + demographic and then create a new rating with those records
  user3 = user_fixture(@create_user3_attrs)
  create_demographic(user3)
  create_rating(user3, product, 3)
  # send the message to the live view
  send(view.pid, %{event: "rating_created"})

  assert render(view) =~ "<title>2.67</title>"
end
```

And that's it! We established our initial state by mounting and rendering the live view with the call to `live/2`, we provided some input by creating a new rating record and sending a message to the live view with `send/2`, and we validated our expectations by asserting that the re-rendered view had some expected content. Our three-step LiveView testing procedure neatly applied to both integration tests that exercise internal live view behavior, and tests that validate the interactions between live view processes. Let's wrap up.

## Write Robust and Comprehensive LiveView Tests

LiveView empowers you to write robust and comprehensive tests without a big investment of effort. By comprising your individual live views out of reuseable, single-purpose reducer functions, you provide lots of opportunities for deep unit testing that can cover lots of scenarios and edge cases quickly. You can even use the same elegant reducer pipelines in your tests to verify the behavior of those pipelines. For integration tests, LiveView provides all the functionality you need to exercise the full range of LiveView interactions with the `LiveViewTest` module. We can use the functions in that model to apply the same three-step process that guides all of our tests, making it quick and easy to spin up tests for even complex LiveView interactions. Finally, thanks to the fact that LiveView is built on top of OTP, and a live view is really nothing more than a process, we can even easily test interactions _between_ live views by relying on simple message passing.

All of this together makes it possible for your and your team to guarantee comprehensive test coverage for your LiveView applications, ensuring that you can move quickly while maintaining bug-free code. The powerful set of tools in your LiveView testing kit is just one of the many reasons that teams are so productive in LiveView.

