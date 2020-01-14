# Mocking HTTP Requests in Golang 

Let's take a look at how we can use interfaces to build a shared mock HTTP client that we can use across the test suite of our Golang app. 

## The App 

Let's say we're building an app that interacts with the GitHub API on our behalf. Users of the app can use it to do things like create a GitHub repo, open an issue on repos, fetch information about an organization and more. Our app implements a rest client that makes these GitHub API calls. 

A simplified version of our client, implementing just a `POST` function for now, looks something like this:

```go
package restclient

import (
	"bytes"
	"encoding/json"
	"net/http"
)

// Post sends a post request to the URL with the body
func Post(url string, body interface{}, headers http.Header) (*http.Response, error) {
	jsonBytes, err := json.Marshal(body)
	if err != nil {
		return nil, err
	}
	request, err := http.NewRequest(http.MethodPost, url, bytes.NewReader(jsonBytes))
	if err != nil {
		return nil, err
	}
	request.Header = headers
	client := &http.Client{}
	return Client.Do(request)
}
```

You can see that we've defined a package, `restclient`, that implements a function, `Post`. That function takes in the URL to which we are sending the POST request, the body of the request and any HTTP headers. It uses the [`http` package](https://golang.org/pkg/net/http/) to make a web request and returns a pointer to an HTTP response, `*http.Response`, or an error. 

The function body takes care of the following:

* Convert the given body into JSON with a call to `json.Marshal`
* Make a new `http.Request` with the `http.MethodPost`, the given url, and the JSON body converted into a reader. 
* Add the given headers to the request 
* Create an instance of an `http.Client` struct
* Use the http `Do` function to send the request 

Our `Post` function directly instantiates the client struct and calls `Do` on that instance. This leaves us with a little problem...

## The Problem 

Because our `restclient` package's `Do` function calls directly on the `http.Client` instance, any tests we write that trigger code pathways using this client will *actually make a web request to the GitHub API*. Yikes! This means we will be spamming the *real* github.com with our fake test data, doing thinks like creating test repos *for real* and using up our API rate limit with each test run. 

Further, relying on real GitHub API interactions makes it difficult for us to write legible tests in which a given set of input results in an expected outcome. We end up with tests that are less declarative, that force other developers reading our code to infer an outcome based on the input to a particular HTTP request. 

So, how can we write clear and declarative tests that avoid sending real web requests? We'll build a mock HTTP client and configure our tests to use that mock client. Then, each test can clearly declare the mocked response for a given HTTP request. But wait! In order to build our mock client and teach our code when to use the real client and when to use the mock, we'll need to build an interface.

## We Need an Interface! 

An interface allows us to define a collection of methods. Any custom struct types implementing that same collection of methods will be considered to conform to that interface. 

A struct's ability to satisfy a particular interface is not enforced. Instead, it is implied that a given struct satisfies a given interface if that struct implements all of the methods declared in the interface. 

Interfaces allow us to achieve polymorphism––instead of a given function or variable declaration expected a specific type of struct, it can expect an entity of an interface type shared by one or more structs. In this way we can define functions that accepts or declare variables that can be set equal to a variety of structs that implement a shared behavior. 

If you're unfamiliar with interfaces in Golang, check out [this excellent and concise resource from Go By Example](https://gobyexample.com/interfaces).  

So, what does this have to do with mocking web requests in our test suite? 

We'll refactor our `restclient` package to be less rigid when it comes to its HTTP client. Instead of instantiating the `http.Client` struct directly in our `Post` function body, our code will get a little smarter and a little more flexible. We will teach it to work work any struct that conform to a shared HTTP client interface. Then, we will be able to configure our package to use the `http.Client` struct by default and an instance of our mock client struct (coming soon!) in any tests for which we want to mock web requests. 

## Let's Build It!

### Step 1. Define the Client Interface 

First off, we'll define an interface in our `restclient` package that both the `http.Client` struct and our soon-to-be-defined mock client struct will conform to. 

We'll call our interface `HTTPClient` and declare that it implements just one function, `Do`, since that is the only function we are currently invoking on the `http.Client` struct. 

```go
package restclient

// HTTPClient interface
type HTTPClient interface {
	Do(req *http.Request) (*http.Response, error)
} 
```

Note that we've declared that our interface's `Do` function takes in an argument of a pointer to an `http.Request` and returns either a pointer to an `http.Response` or an error. This is the *exact API of the existing `http.Client`'s `Do` function*. We need to ensure that this is the case since we are defining an interface that the `http.Client` can conform to, along with our as yet to be defined mock client struct. 

Now that we've defined our interface, let's make our package smart enough to operate on any entity that conforms to that interface.

### Step 2. Defining the `Client` Variable 

We'll declare a variable, `Client`, of the type of our `HTTPClient` interface:

```go
package restclient
...

var (
	Client HTTPClient
)
```

A variable of an interface type can be set equal to any type that implements that interface. Since `http.Client` conforms to the `HTTPClient` interface, we can later set `Client` to an instance of this struct. Later, once we define our mock client and conform it to our `HTTPClient` interface, we will be able to set the `Client` variable to instances of either `http.Client` *or* the mock HTTP client. 

Now that we've defined our `Client` variable, let's teach our `restclient` package to set `Client` to an instance of `http.Client` when it initializes. We'll do this is an `init` function. Recall that a package's `init` function will run just once, when the package is imported (regardless of how many times you import that package elsewhere in your app), and before any other part of the package. Learn more about the `init` function [here](https://www.digitalocean.com/community/tutorials/understanding-init-in-go).

```go
package restclient
...

var (
	Client HTTPClient
)

func init() {
	Client = &http.Client{}
}
```

Our `init` function sets the `Client` var to a newly initialized instance of the `http.Client` struct. 

Lastly, we need to refactor our `Post` function to use the `Client` variable instead of calling `&http.Client{}` directly:

```go
package restclient 
...
// Post sends a post request to the URL with the body
func Post(url string, body interface{}, headers http.Header) (*http.Response, error) {
	jsonBytes, err := json.Marshal(body)
	if err != nil {
		return nil, err
	}
	request, err := http.NewRequest(http.MethodPost, url, bytes.NewReader(jsonBytes))
	if err != nil {
		return nil, err
	}
	request.Header = headers
	return Client.Do(request) // THIS IS NEW AND IMPROVED!
} 
```

Putting it all together, our `restclient` package now looks like this:

```go
package restclient

import (
	"bytes"
	"encoding/json"
	"net/http"
)

// HTTPClient interface
type HTTPClient interface {
	Do(req *http.Request) (*http.Response, error)
}

var (
	Client HTTPClient
)

func init() {
	Client = &http.Client{}
}

// Post sends a post request to the URL with the body
func Post(url string, body interface{}, headers http.Header) (*http.Response, error) {
	jsonBytes, err := json.Marshal(body)
	if err != nil {
		return nil, err
	}
	request, err := http.NewRequest(http.MethodPost, url, bytes.NewReader(jsonBytes))
	if err != nil {
		return nil, err
	}
	request.Header = headers
	return Client.Do(request)
}
```

One important thing to call out here is that we've ensured that our `Client` variable is exported by naming it with a capital letter `C`. Since it is exported, we can operate on it anywhere else in our app where we are importing the `restclient` package. In other words, in any test in which we want to mock calls to the HTTP client, we can do the following:

```go
import "restclient"

restclient.Client = &ourMock{} // Mock definition coming soon!
```

Now that we understand what our interface is allowing us to do, we're ready to define our mock client struct!

### Step 3. Define the Mock Client Struct 

Since its likely that we'll need to mock web requests in tests for various packages in our app––tests for any portion of code flow that makes a web request using our `restclient`--we'll define our mock client in its very own package that can be imported to any test file.

We'll define our mock in `utils/mocks/client.go`. We'll start out by defining a custom struct type, `MockClient`, that implements a `Do` function according to the API of our `HTTPClient` interface:

```go 
package mocks

import "net/http"

// MockClient is the mock client
type MockClient struct {
	DoFunc func(req *http.Request) (*http.Response, error)
}

// Do is the mock client's `Do` func
func (m *MockClient) Do(req *http.Request) (*http.Response, error) {
	// coming soon!
}
```

Note that because we have capitalized the `MockClient` struct, it is exported from our `mocks` package and can be called on like this: `mocks.MockClient` in any other package that imports `mocks`.

Now we have a `MockClient` struct that conforms to the `HTTPClient` interface. It implements a `Do` function just like `http.Client` and we can configure the `restclient` package in a given test to set its `Client` variable to an instance of this mock:

```go
restclient.Client = &mocks.MockClient{}
```

But how can we ensure that a given test's call to `restclient.Client.Do` will return a particular mocked value?

We need to make the return value of our mock client's `Do` function configurable. Let's do it!

### Step 4. Configure the Mock Client's `Do` Function 

We'll define an exported variable, `GetDoFunc`, in our `mocks` package:

```go
package mocks

import "net/http"
...

var (
	// GetDoFunc fetches the mock client's `Do` func
	GetDoFunc func(req *http.Request) (*http.Response, error)
)
```

The `GetDoFunc` can hold any value that is a function taking in an argument of a pointer to an `http.Request` and return either a pointer to an `http.Response` or an error. 

Next up, we'll implement the function body of the `mocks` package's `Do` function to return the invocation of `GetDoFunc` function with an argument of whatever request was passed into `Do`:

```go
package mocks

import "net/http"

// MockClient is the mock client
type MockClient struct {
	DoFunc func(req *http.Request) (*http.Response, error)
}

var (
	// GetDoFunc fetches the mock client's `Do` func
	GetDoFunc func(req *http.Request) (*http.Response, error)
)

// Do is the mock client's `Do` func
func (m *MockClient) Do(req *http.Request) (*http.Response, error) {
	return GetDoFunc(req)
}
```

### Step 5. Using the Mock Client 

So, what does this do for us? In ensures that we can set `mocks.GetDoFunc` equal to any function that conforms to the `GetDoFunc` API. 

In other words, we can define an anonymous function that returns whatever canned response we want for a given test's call to the HTTP client. We can set `mocks.GetDoFunc` equal to that function, thus ensuring that calls to the mock client's `Do` function returns that canned response. Let's take a look.

First, we set `restclient.Client` equal to an instance of our mock struct:

```go
restclient.Client = &mocks.Client{} 
```

Thus, when we invoke a code flow that calls `restclient.Post`, the call to `Client.Do` in that function is really a call to our mock client's `Do` function. 

Next, we set `mocks.GetDoFunc` equal to an anonymous function that returns some response that will help our test satisfy a certain scenario:

```go
mocks.GetDoFunc = func(*http.Request) (*http.Response, error) {
		return nil, errors.New(
			"Error from web server",
		)
	}
```

Thus, when `restclient.Post` calls `Client.Do`, the mock client's `Do` function invokes this anonymous function, returning the return value defined above. 

Let's put it all together in an example test! Let's say our app has a service `repositories`, that makes a `POST` request to the GitHub API to create a new repo. We want to write a test for the happy path--a successful repo creation. Our test will look something like this:

```go
package services

import (
	"bytes"
	"io/ioutil"
	"testing"

	"github.com/SophieDeBenedetto/golang-microservices/src/api/clients/restclient"
	"github.com/SophieDeBenedetto/golang-microservices/src/api/domain/repositories"
	"github.com/stretchr/testify/assert"
)

func TestCreateRepoSuccess(t *testing.T) {
	request := repositories.CreateRepoRequest{Name: "Test Name"}
	resp, err := RepoService.CreateRepo(request)
	assert.NotNil(t, resp)
	assert.Nil(t, err)
	assert.EqualValues(t, "Test Name", resp.Name)
}
```

Our test creates a new `repositories.CreateRepo` request and calls `RepoService.CreateRepo` with an argument of that request. Under the hood of this `CreateRepo` function, our code calls `restclient.Post`. 

As it currently stands, we are not mocking anything, and the call to `RepoService.CreateRepo` in this test will really send a web request to the GitHub API. Oh no!

Let's configure this test file to use our mock client. Then we'll configure this specific test to mock the call to `resclient.Client.Do` with a specific "success" response. 

We'll use an `init` function to set `restclient.Client` to an instance of our mock client struct:

```go
package services

import (
	"net/http"
	"testing"

	"github.com/SophieDeBenedetto/golang-microservices/src/api/clients/restclient"
	"github.com/SophieDeBenedetto/golang-microservices/src/api/domain/repositories"
	"github.com/SophieDeBenedetto/golang-microservices/src/api/utils/mocks"
	"github.com/stretchr/testify/assert"
)

func init() {
	restclient.Client = &mocks.MockClient{}
}
```

Then, in the body of our test function, we'll set `mocks.GetDoFunc` equal to an anonymous function that returns the desired response:

```go
func TestCreateRepoSuccess(t *testing.T) {
	// build response JSON
	json := `{"name":"Test Name","full_name":"test full name","owner":{"login": "octocat"}}`
	// create a new reader with that JSON
	r := ioutil.NopCloser(bytes.NewReader([]byte(json)))
	mocks.GetDoFunc = func(*http.Request) (*http.Response, error) {
		return &http.Response{
			StatusCode: 200,
			Body:       r,
		}, nil
	}
	request := repositories.CreateRepoRequest{Name: "Test Name"}
	resp, err := RepoService.CreateRepo(request)
	assert.NotNil(t, resp)
	assert.Nil(t, err)
	assert.EqualValues(t, "Test Name", resp.Name)
	assert.EqualValues(t, "octocat", resp.Owner)
}
```

Here, we've built out our "success" response JSON, and turned it into a new reader that we can use in our mocked response body with a call to `bytes.NewReader`.

Then, we set `mocks.GetDoFunc` to an anonymous function that returns an instance of `http.Reponse` with a `200` status code and the given body. 

Now, when our call to `RepoService.CreateRepo` calls `restclient.Client.Do` under the hood, that will return the mocked response defined in our anonymous function. This ensures that we can write simple and clear assertions in our test. 

## Conclusion 

Let's recap what we've built before we wrap up. 

By implementing an `HTTPClient` interface, we were able to make our `restclient` package flexible enough to operate on any client struct that conforms to the interface by implementing a `Do` function. This meant that we could configure the `restclient` package to initialize with a `Client` variable set equal to an `http.Client` instance, but reset `Client` to an instance of a mock client struct in any given test suite. 

We defined a mock client struct that conformed to this interface and implemented a `Do` function whose return value was also configurable. This allowed us to set the return value of the mock client's call to `Do` to whatever response helps us create a given test scenario. 

In this way, we are able to mock any web request sent by our `restclient` in simple, declarative tests that are easy to write and read. 