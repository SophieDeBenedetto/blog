# JWT Storage in Rails + React The Right Way
Local or session storage in the browser might _feel_ like the right place to store a [JWT](https://jwt.io/) when authenticating your client-side app against a backend API. Maybe it feels like the right place because I _told you to do that_. But its not right! Its wrong and its insecure. Instead, use an HTTP-Only cookie to store your JWT. Find out how below!

## What's Wrong With Web Storage?
Any JavaScript running on your domain can access the web storage mechanisms of `localStorage` or `sessionStorage`. These types of storage are therefore vulnerable to [cross-site scripting (XSS)](https://en.wikipedia.org/wiki/Cross-site_scripting) attacks. In such an attack, a bad actor will attempt to inject JavaScript to be run on your page. If they succeed, they could access _everything_ in local and session storage--including any JWTs you store there. Oh no!

## HTTPOnly Cookies to the Rescue!
[HTTPOnly cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) are _not_ accessible via JavaScript and items stored in such cookies are not vulnerable to exposure via an XSS attack.

An HTTPOnly cookie is a small package of data that is sent by the server to the browser. It is not accessible via JavaScript in the browser, but the browser can hold on to it and send it _back_ to the server in subsequent requests. Such cookies are secure ways to hold on to stateful information, like who the user is and whether they are logged in, in the stateless world of HTTP communication.

## Don't Do This
One popular approach to JWT auth between a JavaScript client and a backend server goes something like this:
* Log in in via JavaScript
* Server responds with the JWT
* Client stores that JWT in web storage
* Client grabs JWT from web storage and uses it in an authorization header included in subsequent XHR requests to the backend.
* Server grabs the JWT from this authorization header to authorize incoming requests.

We _don't_ want to expose our JWT in local storage. So, we'll reconfigure this approach a bit.

## Do This
Our approach will work something like this:
* Log in via JavaScript, requesting that the server send the HTTPOnly cookie to the browser
* Server will set the HTTPOnly cookie to include the encoded JWT after authenticating the user and respond with a header that makes the cookie available to the browser (but NOT to JavaScript)
* Tell all subsequent XHR requests to include the HTTPOnly cookie
* Server will grab the JWT from the HTTPOnly cookie to authorize incoming requests.

Let's build it!

## Set Up
We won't be taking a close look at setting up the React app, Rails API or React router. We'll assume you have your applications set up and focus solely on the authentication between the client and server.

## Configuring The Rails API

### Using Cookies
We need to make a couple of changes to our standard Rails API to get this working.

If you generated a Rails api-only application, we have to take some steps to get our app to use cookies. First, tell Rails to use the `ActionDispatch::Cookies` middleware. Add the following to your middleware stack in `config/appliction.rb`:

```ruby
# application.rb
...
config.middleware.use ActionDispatch::Cookies
```

Now our Rails app will be able to use the cookie-based session store. In order to set cookies in our Rails controller, we need to include the `::ActionController::Cookies` module in the Application Controller:

```ruby
# app/controllers/application_controller.rb
class ApplicationController
  include ::ActionController::Cookies
end
```

Now we can access the `cookies` method to set, get and delete HTTPOnly cookies!

### CORS
CORS, or [Cross Origin Request Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), is a mechanism that uses HTTP headers to allow one application to receive and accept requests from another application from a different domain.  

We need to use CORS to tell our Rails app to accept requests from our JavaScript app. We can do so via the `Rack::Cors` middleware that is available in our Rails 5 API. If you're working with an earlier version of Rails, you can include the [Rack Cors gem](https://github.com/cyu/rack-cors).

In an initializer, `config/initializers/cors.rb`, we'll set the following:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins 'http://localhost:4000' # your client's domain

    resource '*',
    headers: :any,
    methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

#### Gotcha: CORS and HTTPOnly Cookies

Notice that we've specified the origin of our request as the domain of the JavaScript client app, as opposed to the less strict, `*` (i.e. "allow all") character. That is because we _must specify_ the domain in order to allow our client to request the response headers the browser needs in order to receive the HTTPOnly cookie. More on this in a bit.

## Requesting The HTTPOnly Cookie
When JavaScript sends the "login" POST request, it needs to explicitly tell the server to to include the cookies in the response. We do this via the [`Request.credentials`](https://developer.mozilla.org/en-US/docs/Web/API/Request/credentials) property. We'll give our `credentials` property a value of `"include"`, which means "include cookies in the response from a cross-origin request".

If you're using [Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) to send your login request to the server, your request should look something like this:

```javascript
function login (loginParams) {
  return fetch(`${baseUrl}/api/v1/auth`, {
    method: 'POST'
    credentials: 'include',
    body: JSON.stringify(loginParams)
  }).then(res => res.json())
}
```

If you inspect the traffic in your browser's Network tab, you'll notice that the response to this request includes the [Access-Control-Allow-Credentials header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials). This is the response header that makes the cookies available to the browser. But keep in mind that just because the cookie is available to the browser _doesn't_ mean JavaScript can access it. If its an HTTPOnly cookie, it will still be safe from sneaky JS scripts.

Let's take a look at setting that cookie on the server now.

## Setting the HTTPOnly Cookie on the Server
We'll tell our Rails controller to set a signed HTTPOnly cookie `jwt: the.valid.jwt`, if the user is authenticated successfully.

```ruby
# app/controllers/api/v1/auth_controller.rb

def create
  user = User.find_by(params[:email])
  if user && user.authenticate(params[:password])
    created_jwt = issue_token({id: user.id})
    cookies.signed[:jwt] = {value:  created_jwt, httponly: true}
    render json: {username: user.username}
  else
    render json: {
      error: 'Username or password incorrect'
    }, status: 404
  end
end
```

Here is where the magic happens:

```ruby
cookies.signed[:jwt] = {value:  created_jwt, httponly: true}
```

This sets a signed cookie, which prevents users from tampering with its value. We can access signed cookies on the server by calling `cookies.signed[:key_name]`.

Now that we're setting the cookie on the server, and making sure its available to the browser via our request's `credentials`, let's look at sending subsequent authorized requests using the cookie.

## Sending Authorized Requests
All we have to do on the client-side to ensure that all subsequent XHR requests include the cookies is amend any `fetch` requests to include `credentials: "include"`.

Let's say we have a "get current user request". We'll update the `fetch` to include the credentials property:

```javascript
function currentUser () {
  return fetch(`${baseUrl}/api/v1/current_user`, {
    credentials: 'include'
  }).then(res => res.json())
}
```

Then, on the server-side, we'll define a `before_action` that grabs the JWT from the relevant key in the cookie:

```ruby
# app/controllers/api/v1/current_user_controller.rb
class Api::V1::CurrentUserController < ApplicationController
  before_action :authenicate_user, only: [:show]
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController
  def authenticate_user
    jwt = cookies.signed[:jwt]
    decode_jwt(jwt)
  end
end
```
If the `:jwt` key is present in our cookies, `cookies.signed[:jwt]` will return the JWT we stored under that key. Otherwise, it will return `nil`.

Lastly, let's talk about logging out.

## Logging Out
In order to log out, all we need to do is tell our server to delete the `:jwt` cookie.

We'll define a logout function on the client-side that sends the logout request:

```javascript
function logout () {
  return fetch(`${baseUrl}/api/v1/auth`, {
    method: 'DELETE',
    credentials: 'include'
  }).then(res => res.json())
}
```

On the server-side, we'll match this route to a controller action that responds to the request by deleting the `:jwt` from the cookie:

```ruby
# app/controllers/api/v1/auth_controller.rb
def destroy
  cookies.delete(:jwt)
end
```

And that's it! The server will respond to the request with the cookies, _minus_ the deleted `:jwt` key/value pair. Subsequent requests to the server will _not_ include that particular key in the cookie. This will cause the lookup of `cookies.signed[:jwt]` to return `nil`, and the server will understand the request to be unauthorized.

## Conclusion
Web storage is exactly that--storage. It doesn't enforce any security measures and as such its _not_ the right place for sensitive information. While it might be appropriate to store a flag that _indicates_ whether or not a user is logged in in local or session storage, its _not_ a good place to put the JWT itself. To store JWTs in the browser so that our client can send JWT-authenticated requests to the server, we should leverage HTTPOnly cookies.

Requesting, setting and using such tokens isn't too difficult, and I hope this post was able to surface some common misconceptions and "gotchas", particularly in a Rails api-only application.
