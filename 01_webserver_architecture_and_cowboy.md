# WEB SERVER FUNDAMENTALS

## THE WEB SERVER ARCHITECTURE AND COWBOY

> \enquote{
  Web servers are written in C, and if they're not, they're written in Java or
  C++, which are C derivatives, or Python or Ruby, which are implemented in C.
}

> -- Rob Pike, Co-creator of Go

\

The web server is a key component of any modern-day web framework. Following the
quote above by Rob Pike, the web server __Cowboy__, written in __Erlang__, is
also in a way implemented in C. In this chapter, we will not be learning C
unfortunately, but we will take a closer look at how a web server is designed.
We will provide some background on how a web server is built and set up to
communicate with a client using __HyperText Markup Language (HTML)__.

We will also learn the fundamentals of how HTTP requests and responses work,
including their anatomy. We will then learn how to construct an HTTP response
and send it using a web server. Moreover, we will learn the fundamentals of web
server architecture by examining the components of Cowboy. Furthermore, we will
learn ways to test a web server and measure its performance. Doing this will
better position us to build our own web server in future chapters.

\newpage

Following are the topics we will cover in this chapter:

* What is a web server?

* Fundamentals of client-server architecture

* Fundaments of HTTP

* How an HTTP server works

* Using Cowboy to build a web server

* Using dynamic routes with Cowboy

* Serving HTML

* Testing the web server

Going through these topics and looking at Cowboy will allow us to build our own
HTTP server in _Chapter 2, Understanding the HTTP Server_.

### TECHNICAL REQUIREMENTS

The best way to read through this chapter is by following along with the code on
your computer. So, having a computer with Elixir and Erlang ready to go would be
ideal. I recommend using a version manager like [__asdf__](
  https://asdf-vm.com/#/
) to install _Elixir 1.11.x_ and _Erlang 23.2.x_, to get similar results as the
code written in the book. We will also be using an HTTP client like `curl` or
`wget` to make HTTP requests to our server, and a web browser to render HTML
responses.

Although most of the code in this chapter is relatively simple, basic knowledge
of Elixir and/or Erlang would also come in handy. It will allow the readers to
get more out of this chapter, while setting the foundation for other chapters.

Since most of this chapter isn't coding, you can also choose to read without
coding, but the same doesn't apply for other chapters.

\newpage

### WHAT IS A WEB SERVER?

A web server is an entity that delivers the content of a site to the end-user.
It's typically a long- running process, listening for requests on a port, and
upon receiving a request, the web server responds with a document. This way of
communication is standardized by the __Transmission Control Protocol/Internet
Protocol (TCP/IP)__ model, which provides a set of communication protocols used
by any communication network. There are other layers of standardization like
__Hypertext Transfer Protocol (HTTP)__ and __File Transfer Protocol (FTP)__,
which dictate standards of communication at the application layer based on
different applications like web applications (HTTP) and file transfer (FTP),
while still using TCP/IP at the network layer. We will be primarily focusing on
a web server using HTTP at the application layer, in this book.

> \begin{longfbox}
> \textbf{Example HTTP Server}
>
> If you have Python3 installed on your machine (you likely do), you can
> quickly spin up a web server that serves a static HTML document by creating
> a file index.html in a new directory, and running a simple HTTP
> python server. Here are the commands:
>
> \$ mkdir test-server \&\& cd test-server \&\& touch index.html
>
> \$ echo "<h1>Hello World</h1>" > index.html
>
> \$ python -m http.server 8080
>
> Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) …
>
> If you are on Python2 replace http.server with SimpleHTTPServer.
>
> Now, once you navigate to http://localhost:8080/ on your web browser, you
> should see "Hello World" as the response. You should also be able to see the
> server logs when you navigate back to the terminal.
>
> To stop the HTTP server, press Ctrl + C.
> \end{longfbox}

The primary goal of web servers is to respond to a client’s request with
documents in the form of HTML or JSON. These days, however, web servers do much
more than that. Some have analytical features like admin UI and some have the
ability to generate dynamic documents. For example, Phoenix’s web server has
both of those features.

### EXPLORING THE CLIENT-SERVER ARCHITECTURE

In the context of HTTP servers, clients generally mean the web browsers that
enable end-users to read the information being served. Whereas, servers mean
long-running processes that serve information in the form of documents to those
clients. These documents are generally written in __HTML__, and are used as
means of communication between the client and the server. Clients are
responsible for enabling the end-user to send a request to the server and
display the final response from the server. Browsers allow the users to retrieve
and display information without requiring any knowledge of HTML or web servers,
by just providing an address (the URL).

At a given time, many clients can access a server’s information. This
puts the burden of scaling on the servers to have the ability to respond to
multiple requests within an acceptable period of time. Now that we know what a
web server is and understand their primary goal, let’s move on to what enables
the communication between web servers: HTTP.

### UNDERSTANDING HTTP

HTTP is an application layer protocol that provides communication standards
between web browsers and web servers. This standardization helps browsers and
servers talk to each other as long as the request and the response follow a
specific format.

An HTTP request is a text document with four elements:

* __Request line__: This line contains the HTTP method, the resource requested
  (URI) and HTTP version being used for the request. The HTTP method generally
  symbolizes the intended action being performed on the requested resource.
  For example, `GET` is used to retrieve resource’s information, whereas `POST`
  is used to send new resource’s information as a form.

* __Request headers__: The next group of lines contains headers that contain
  information about the request and the client. These headers are usually used
  for authorization, determining the type of request or resource, storing web
  browser information etc.

* __Line break__: This indicates the end of request headers.

* __Request body (Optional)__: If present, this information contains data to be
  passed to the server. This is generally used to submit domain-specific forms.

Here’s an example of an HTTP request document:

```
GET / HTTP/1.1
Host: localhost:8080
User-Agent: curl/7.75.0
Accept: */*

Body of the request
```

As you can see, the above request was made with the `GET` method to
`localhost:8080` with the body, `"Body of the request."`

Similarly, an HTTP response contains four elements:

* __Response status line__: This line consists of the HTTP version, a status
  code, and a reason phrase which generally corresponds to the status code.
  Status codes are three-digit integers which helps us categorize responses.
  For example, `2XX` status codes are used for a successful response, whereas
  `4XX` status codes are used for errors due to the request.

* __Response headers__: Just like request headers, this group of lines contains
  information about the response and the server’s state. These headers are
  usually used to show the size of the response body, server type, date/time of
  the response etc.

* __Line break__: This indicates the end of response headers.

* __Response body (Optional)__: If present, this section contains the
  information being relayed to the client.

Following is an example of an HTTP response document:

```
HTTP/1.1 404 Not Found
content-length: 13
content-type: text/html
server: Cowboy

404 Not found
```

The above response is an example of a `404 (Not found)` response. Notice that
the content-length shows the number of characters present in the response body.

Now that we know how HTTP facilitates client-server communication, it is time
to build a web server using Cowboy.

### THE COWBOY WEB SERVER

Cowboy is a minimal and fast HTTP web server written in Erlang. It supports
several modern standards like HTTP/2, HTTP/1.1, WebSocket etc. On top of that,
it also has several introspective capabilities, thus enabling easier
development and debugging. Cowboy has a very well-written and well-documented
code base, with a highly extendable design, which is why it is the default
web server for the Phoenix framework.

Cowboy uses __Ranch__, a TCP socket accepter, to create a new TCP connection,
on top of which it uses its router to match a request to a handler. Router and
handler are middleware that are part of Cowboy. Upon receiving a request, Cowboy
creates a stream which is further handled by a stream handler. Cowboy has a
built-in configuration that handles a stream of requests using
`:cowboy_stream_h`. This module spins up a new Erlang process for every request
that makes to the router.

Cowboy also sets up one process per TCP connection. This also allows Cowboy to
be compliant with HTTP/2, which requires concurrent requests. Once a request is
served, the Erlang process is killed without any need for cleanup.

The following image shows the Cowboy request/response cycle:

![Cowboy request/response cycle](
  ./images/01/01_cowboy_request_response_cycle.png
){ width=200px }

As you can see in Figure 1.1, when a client makes a request, Ranch first
converts it to a stream, which is further handled by router and handler
middleware in Cowboy. Traditionally, a response is sent either by the router or
the handler. For example, a handler could handle a request and send a response,
or if no handler is present for a route, the router could also send a 404
response.

Cowboy also generates a few response headers as we will see in the next section
where we build and test a Cowboy-powered web application.

### BUILDING A WEB APPLICATION USING COWBOY

In this section, we will take a look at some of the individual components of
Cowboy web server and use them to build a simple web application in Elixir.

Let's start by creating a new mix Mix project by entering the following in your
terminal:

```bash
$ mix new cowboy_example --sup
```

Passing the `--sup` option to `mix new` command allows us to create a mix
project with a supervision tree. We will be using that supervision tree to start
and supervise our web server process.

Now, we will add Cowboy as a dependency to our project by adding it to the
`mix.exs` file’s dependencies:

\codecaption{mix.exs}
```{.elixir .numberLines}
defmodule CowboyExample.MixProject do
```
```{.elixir .numberLines startFrom="18"}
  # ...
  defp deps do
    [
      {:cowboy, "~> 2.8.0"}
    ]
  end
end
```

\

Follow it up by running  `mix deps.get` from the project’s root, which should
fetch Cowboy as a dependency.

\newpage

Now that we have added Cowboy, it is time to configure it to listen on a port.
We will be using two functions to accomplish that:

* `:cowboy_router.compile/1:` This function is responsible for defining a set of
  requested host, path, and parameters to a dedicated request handler. This
  function also generates a set of rules, known as dispatch rules, to use those
  handlers.

* `:cowboy.start_clear/3:` This function is responsible for starting a listener
   process on a TCP channel. It takes a listener name (an atom), transport
   options such as the TCP port and protocol options like the dispatch rules
   generated using `:cowboy_router.compile/1` function.

Now, let us use these functions to write an HTTP server. We can start by
creating a new module to house our new HTTP server:

\codecaption{lib/cowboy\_example/server.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Server do
  @moduledoc """
  This module defines a cowboy HTTP server and starts it
  on a port
  """

  @doc """
  This function starts a Cowboy server on the given port.

  Routes for the server are defined in CowboyExample.Router
  """
  def start(port) do
    routes = CowboyExample.Router.routes()

    dispatch_rules =
      :cowboy_router.compile(routes)

    {:ok, _pid} =
      :cowboy.start_clear(
        :listener,
        [{:port, port}],
        %{env: %{dispatch: dispatch_rules}}
      )
  end
end
```

The preceding function starts a Cowboy server that listens to HTTP requests at
the given port. By pattern matching on `{:ok, _pid}`, we’re making sure that
`:cowboy.start_clear/3` doesn’t fail silently.

We can try to start our web server by calling `CowboyExample.Server.start/1`
function with a port. But, as you can see above, we will also need to define the
router module `CowboyExample.Router`. This module’s responsibility is to define
routes that can be used to generate dispatch rules for our HTTP server. This can
be done by storing all the routes, parameters and handler tuples in the router
module and passing them to the `:cowboy_router.compile/1` call.

Let’s define the router module with the route for the root URL of the host,
"`/`".

\codecaption{lib/cowboy\_example/router.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router do
  @moduledoc """
  This module defines all the routes, params and handlers.

  This module is also the handler module for the root route.
  """
  require Logger

  @doc """
  Returns the list of routes configured by this web server
  """
  def routes do
    [
      # For now, this module itself will handle root requests
      {:_, [{"/", __MODULE__, []}]}
    ]
  end
end
```

We will also be using `CowboyExample.Router` itself as the handler for that
route, which means we have to define the function `init/2` which takes the
request and its initial state.

\newpage

So, let us define the `init/2` function.

\codecaption{lib/cowboy\_example/router.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router do
```
```{.elixir .numberLines startFrom="17"}
  # ..
  @doc """
  This function handles the root route, logs the requests and responds
  with Hello World as the body
  """
  def init(req0, state) do
    Logger.info("Received request: #{inspect req0}")

    req1 =
      :cowboy_req.reply(
        200,
        %{"content-type" => "text/html"},
        "Hello World",
        req0
      )

    {:ok, req1, state}
  end
end
```

\

As you can see in the preceding code, we have defined the `routes/0` function
that returns the dispatch rules routes for our web application. For the
handler module, we’re currently using the `CowboyExample.Router` module itself
by defining the `init/2` function, which responds with `"Hello World"` and a
status of `200`, whenever invoked. We have also added a call to the `Logger`
module to log all requests to the handler. This will increase visibility while
running it in the development environment.

In order for our web server to start up when the app is started, we need to
add it to our application's supervision tree.

\newpage

### SUPERVISING THE WEB SERVER

Now that we have added a router and a handler to our web server, we can add it
as a child to our supervision tree by updating the list of children in our
application module. For now, I will use a hard-coded TCP port to 4040 for our
server, but we will use application-level configurations to set it later in this
chapter.

\codecaption{lib/cowboy\_example/application.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Application do
  @moduledoc false

  use Application

  @impl true
  def start(_type, _args) do
    children = [
      # Add this line
      {Task, fn -> CowboyExample.Server.start(4040) end}
         ]

    opts = [
      strategy: :one_on_one,
      name: CowboyExample.Supervisor
    ]

    Supervisor.start_link(children, opts)
  end
end
```

In the preceding code, we’re adding to supervised children, a `Task` with the
function to start the Cowboy listener as an argument that eventually gets passed
to `Task.start_link/1`. This makes sure that our web server process is part of
the application’s supervision tree.

\newpage

Now, we can run our web application by running the mix project with the
`--no-halt` option:

```bash
$ mix run --no-halt
```

> \begin{longfbox}
> \textbf{Note}
>
> Passing the --no-halt option to mix run command makes sure that the
> application, along with the supervision tree, is still running event after the
> command has returned. This is generally used for long running processes like
> web servers.
> \end{longfbox}

Without stopping the previous command, in a separate terminal session, we can
make a request to our web server using the command-line utility `curl` with `–v`
option to get a verbose description of our requests and responses.

```bash
$ curl –v http://localhost:4040/
*   Trying ::1:4040...
* connect to ::1 port 4040 failed: Connection refused
*   Trying 127.0.0.1:4040...
* Connected to localhost (127.0.0.1) port 4040 (#0)
> GET / HTTP/1.1
> Host: localhost:4040
> User-Agent: curl/7.75.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-length: 11
< content-type: text/html
< server: Cowboy
<
* Connection #0 to host localhost left intact
Hello world%
```

As we can see in the preceding code, we get the expected `"Hello World"`
response along with the expected status code of `200`. As mentioned in the
previous section, Cowboy adds custom response headers to give us more
information about the how it was processed. We can also see headers for type of
server (Cowboy), content length and content type.

\newpage

We should also see an application-level log corresponding to the request in the
terminal session running the mix project. The logs should look somewhat like
this:

```bash
$ mix run --no-halt

20:39:43.061 [info]  Received request: %{
  bindings: %{},
  body_length: 0,
  cert: :undefined,
  has_body: false,
  headers: %{
    "accept" => "*/*",
    "host" => "localhost:4040",
    "user-agent" => "curl/7.75.0"
  },
  host: "localhost",
  host_info: :undefined,
  method: "GET",
  path: "/",
  path_info: :undefined,
  peer: {{127, 0, 0, 1}, 35260},
  pid: #PID<0.271.0>,
  port: 4040,
  qs: "",
  ref: :listener,
  scheme: "http",
  sock: {{127, 0, 0, 1}, 4040},
  streamid: 1,
  version: :"HTTP/1.1"
}
```

We can see that we’re logging all the details corresponding to the request
including headers, host, URI and the process id of the process processing the
request.

Congratulations, you have now successfully built a `Hello World` web server
using Cowboy. Now, it’s time to add more routes to our web server.

### ADDING ROUTES WITH BINDINGS

Most web applications support the ability to not just serve a static route,
but also dynamic routes with a specific pattern. It’s time to see how we can
Cowboy to add dynamic routes to our router.

Say we want to add a new route to our application, which responds with a custom
greeting for a person whose name is dynamic. Let’s update our router to define a
handler for a new dynamic route. We can also use this opportunity to move our
`Root` handler (`init/2` function) to a different module. This makes our code
more compliant with the single-responsibility principle, further making it
easier to follow:

\codecaption{lib/cowboy\_example/router.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router do
  @moduledoc """
  This module defines all the routes, params and handlers.
  """
  alias CowboyExample.Router.Handlers.{Root, Greet}

  @doc """
  Returns the list of routes configured by this web server
  """
  def routes do
    [
      {:_, [
        {"/", Root, []},
        # Add this line
        {"/greet/:who", [who: :nonempty], Greet, []}
      ]}
    ]
  end
end
```

In the preceding code, we have added a new route which expects a non-empty value
for the variable binding, `:who`. This variable gets bound to a request based on
the URL. For example, for a request with URL `"/greet/Luffy"`, the variable
bound to `:who` will be `"Luffy"` and for a request with URL `"/greet/Zoro"`,
it will be `"Zoro"`.

Now, let’s define the `Root` handler and move the `init/2` function from our
router to the new handler module. This separates the concerns of defining routes
and handling requests:

\codecaption{lib/cowboy\_example/router/handlers/root.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router.Handlers.Root do
  @moduledoc """
  This module defines the handler for the root route.
  """
  require Logger

  @doc """
  This function handles the root route, logs the requests and responds
  with Hello World as the body
  """
  def init(req0, state) do
    Logger.info("Received request: #{inspect req0}")

    req1 =
      :cowboy_req.reply(
        200,
        %{"content-type" => "text/html"},
        "Hello World",
        req0
      )

    {:ok, req1, state}
  end
end
```

Similarly, let’s define the `Greet` handler for our new dynamic route. We know
that the request has a variable binding corresponding to the key `:who` by the
time it gets to this handler. Therefore, we can use `:cowboy_req.binding/2`
function to access the value of `:who` bound to the request.

\codecaption{lib/cowboy\_example/router/handlers/greet.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router.Handlers.Greet do
  @moduledoc """
  This module defines the handler for "/greet/:who" route.
  """
  require Logger

  @doc """
  This function handles the "/greet/:who", logs the requests and responds
  with Hello `who` as the body
  """
  def init(req0, state) do
    Logger.info("Received request: #{inspect req0}")

    who = :cowboy_req.binding(:who, req0)

    req1 =
      :cowboy_req.reply(
        200,
        %{"content-type" => "text/html"},
        "Hello #{who}",
        req0
      )

    {:ok, req1, state}
  end
end
```

In the above code snippet, we get the value bound to `:who` for the request and
use it with string interpolation to call "`Hello :who`". Now, we have two valid
routes for our web server: the root and the dynamic greet route.

We can test our updates by restarting the Mix application. That can be done by
stopping the HTTP server using `Ctrl + C`, followed by running
`mix run --no-halt` again. Now, let’s make a request to test the new route with
"`Elixir`" as `:who`.

```bash
$ curl http://localhost:4040/greet/Elixir
Hello Elixir%
```

Cowboy offers another way to add dynamic behavior to our routes, and that is by
passing query parameters to our URL. Query parameters can be captured by using
the function `:cowboy_req.parse_qs/1`. This function takes a binding name,
`:who` in this case, and the request itself. Let’s update our greet handler to
now take a custom query parameter for greeting which overrides the default
greeting, "`Hello`", which we can put in a module attribute for better code
organization.

\newpage

\codecaption{lib/cowboy\_example/router/handlers/greet.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router.Handlers.Greet do
```
```{.elixir .numberLines startFrom=4}
  # ..
  @default_greeting "Hello"
```
```{.elixir .numberLines startFrom=7}
  # ..
  def init(req0, state) do
```
```{.elixir .numberLines startFrom=11}
    greeting =
      req0
      |> :cowboy_req.parse_qs()
      |> Enum.into(%{})
      |> Map.get("greeting", @default_greeting)

    req1 =
      :cowboy_req.reply(
        200,
        %{"content-type" => "text/html"},
        "#{greeting} #{who}",
        req0
      )

    {:ok, req1, state}
  end
end
```
We have not updated our greet handler to use `:cowboy.parse_qs/1` to fetch query
parameters from the request. We then put those matched parameters into a map and
get the value in the map corresponding to the key "`greeting`", with a default
of "`Hello`". Now, the greet route should take a query parameter "`greeting`"
to update the greeting used to greet someone in the response. We can test our
updates again by restarting the application and making a request to test the
route with custom greeting parameter:

```bash
$ curl http://localhost:4040/greet/Elixir\?greeting=Hola
Hola Elixir%
```
We now have a web server with a fully functional dynamic route. You might have
noticed that we didn’t specify any HTTP method while defining the routes. Let us
see what happens when we try to make a request to the root with the
`POST` method:

\newpage

```bash
$ curl -X POST http://localhost:4040/
Hello World%
```

As you can see in the example, our web server still responded to the `POST`
request in the same manner as `GET`. We don’t want that behavior, so in the next
section we will see how to validate the HTTP method of a request, and restrict
the root of our application to only respond to `GET` requests.

### VALIDATING HTTP METHODS

Most modern web applications have a way of restricting requests to a route
based on the HTTP method. In this section, we will see how to restrict our
handlers to work with a specific HTTP method in a Cowboy-based web application.
The simplest way of accomplishing that in a cowboy handler is by
pattern-matching on the first argument of the `init/2` function, which is the
request. A Cowboy request contains a lot of information, including the HTTP
method, used to make the request. So, by pattern-matching on the request with a
specific HTTP method, we can filter requests based on HTTP methods. However, we
will also be needing a general clause for the `init/2` function, which responds
with a 404.

In the `Greet` handler, let us update `init/2` to match only on requests with
the `GET` method, along with another clause which responds with a
404 (Not Found) error.

\codecaption{lib/cowboy\_example/router/handlers/greet.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router.Handlers.Greet do
```
```{.elixir .numberLines startFrom=7}
  # ..
  def init(%{method: "GET"} = req0, state) do
```
```{.elixir .numberLines startFrom=24}
  # ..
  end

  # General clause for init/2 which responds with 404
  def init(req0, state) do
    Logger.info("Received request: #{inspect req0}")

    req1 =
      :cowboy_req.reply(
        404,
        %{"content-type" => "text/html"},
        "404 Not found",
        req0
      )

    {:ok, req1, state}
  end
end
```

Now, let’s make sure only `GET` requests are accepted by our server for the
route. Let’s first make sure GET requests are working:

```bash
$ curl http://localhost:4040/greet/Elixir\?greeting=Hola
Hola Elixir%
```

It’s time to check that a `POST` request for the greet route returns a 404:

```bash
$ curl http://localhost:4040/greet/Elixir\?greeting=Hola -X POST
404 Not found%
```

This ensures that our route works only for `GET` requests. Another way of
validating HTTP methods of our requests would be by using a Cowboy middleware,
but we will not be covering that in this chapter.

> \begin{longfbox}
> \textbf{Cowboy Middleware}
>
> In Cowboy, a middleware is a way to process an incoming request. Every request
> has to go through two middlewares, the router and the handler, but Cowboy
> allows us to define our own custom middleware module, which gets executed in
> the given order. A custom middleware module just needs to implement the
>  execute/2 callback defined in cowboy\_middleware behavior.
> \end{longfbox}

Great! We have successfully enforced a method type for a route. Next, we will
learn how to serve HTML files instead of raw strings.

\newpage

### RESPONDING WITH HTML FILES

Generally, when we write web servers, we do not write our HTML as strings in
handlers. We write our HTML in separate files that are served by our server. We
will use our application’s `priv` directory to store these static files. So,
let’s create a folder priv/static in the root of our project and add a file
`index.html` in that folder. To add some HTML, we can use this command:

```bash
$ echo "<h1>Hello World</h1>" > priv/static/index.html
```

> \begin{longfbox}
> \textbf{Priv directory in OTP}
>
> In OTP (and Elixir), the priv directory is a directory specific to an
> application, which is intended to store files needed by the application when
> it is running. Phoenix, for example, uses the priv/static directory to
> stored processed JavaScript and CSS assets for runtime usage.
>
> \end{longfbox}

Let’s add an endpoint to our server, which returns a static HTML file.

\codecaption{lib/cowboy\_example/router.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router do
  @moduledoc """
  This module defines routes and handlers for the web server
  """
  alias CowboyExample.Router.Handlers.{Root, Greet, Static}

  @doc """
  Returns the list of routes configured by this web server
  """
   def routes do
    [
      {:_, [
        {"/", Root, []},
        {"/greet/:who", [who: :nonempty], Greet, []},
        # Add this line
        {"/static/:page", [page: :nonempty], Static, []}
      ]}
    ]
  end
end
```

Now, we need a static handler module, which will look for and respond with the
given page in the `/priv/static` folder, and if not found, returns a 404.

\codecaption{lib/cowboy\_example/router/handlers/static.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Router.Handlers.Static do
  @moduledoc """
  This module defines the handler for "/static/:page" route.
  """
  require Logger

  @doc """
  This handles "/static/:page" route, logs the requests and responds
  with the requested static HTML page.

  Responds with 404 if the page isn’t found in the
  priv/static folder.
  """
  def init(req0, state) do
    Logger.info("Received request: #{inspect req0}")

    page = :cowboy_req.binding(:page, req0)

    req1 =
      case html_for(page) do
        {:ok, static_html} ->
          :cowboy_req.reply(
            200,
            %{"content-type" => "text/html"},
            static_page,
            req0
          )

        _ ->
          :cowboy_req.reply(
            404,
            %{"content-type" => "text/html"},
            "404 Not found",
            req0
          )
             end

    {:ok, req1, state}
  end

  defp html_for(page) do
    priv_dir =
      :cowboy_example
      |> :code.priv_dir()
      |> to_string()

    page_path = priv_dir <> "/static/#{page}"

    File.read(page_path)
  end
end
```

In the above module, the `html_for/1` function is responsible for fetching the
HTML files from our application’s `priv` directory, for a given path. If the
file is present, the function returns `{:ok, <file_contents>}, >}`; otherwise it
returns an error, upon which we will respond with a 404 message.

We can test the above route by restarting our server again, and making a request
to the `/static/index.html` path. But this time, let us use the web browser in
order to render the HTML contents properly. Here’s what you should see:

![Successful HTML response](
  ./images/01/02_successful_browser_html_response.png
)

Also, to make sure our 404 handler is working correctly, let’s make a browser
request to `/static/bad.html`, a file not present in our application’s `priv`
directory. You should see a 404 message:

![Failed HTML response](
  ./images/01/03_failed_browser_html_response.png
)

Now, we have a web server that can respond with static HTML files. It’s time to
see how we can go about testing it.

### TESTING THE WEB SERVER WITH EXUNIT

Automated testing is a key part of any software, especially in a dynamic-typed
language like Elixir. It is one of the catalysts for writing deterministic
software while documenting expected behaviors of its components. Due to this
reason, we will be making an effort to test everything we build in this book,
including the Cowboy-powered web application we have built in this chapter.

In order to test our web application, we first need to be able to run our
application on a different port in the test environment. This is to ensure that
other `/static/bad.html`, environments do not interfere with our tests. We also
can use an application-level configuration to set a port on which the cowboy
server listens to all the requests. This will allows us to separate the test
port from the development port.

So, let’s update our application to use the configured port or default it to
4040 using a module attribute `@port`:

\codecaption{lib/cowboy\_example/application.ex}
```{.elixir .numberLines}
defmodule CowboyExample.Application do
  @moduledoc false

  @port Application.compile_env(
          :cowboy_example,
          :port,
          4040
        )

  @impl true
  def start(_type, _args) do
    children = [
      # Add this line
      {Task, fn -> CowboyExample.Server.start(@port) end}
    ]

    opts = [
      strategy: :one_on_one,
      name: CowboyExample.Supervisor
    ]

    Supervisor.start_link(children, application)
  end
end
```

We can make sure that the application configuration is different for different
Mix environments by adding the `config/config.exs` file, and setting a different
port in our config for test environment. We will also be updating the logger to
not log warnings. So, let’s add a config file with the following contents:

\codecaption{config/config.exs}
```{.elixir .numberLines}
use Mix.Config

if Mix.env() == :test do
  config :cowboy_example,
    port: 4041

  config :logger, warn: false
end
```

> \begin{longfbox}
> \textbf{Note}
>
> Mix.Config has been deprecated in newer version of Elixir. You might have to
> use the `Config` module instead.
>
> \end{longfbox}

Now, let’s add tests for our server endpoints. In order to test our web server,
we need to make HTTP requests to it and test the responses. To make HTTP
requests in Elixir we will be using `Finch`, a lightweight and high-performance
HTTP client written in Elixir.

\newpage

So, let’s add `Finch` to our list of dependencies:

\codecaption{mix.exs}
```{.elixir .numberLines}
defmodule CowboyExample.MixProject do
```
```{.elixir .numberLines startFrom="18"}
  # ...
  defp deps do
    [
      {:cowboy, "~> 2.8.0"},
      {:finch, "~> 0.6.2"}
    ]
  end
end
```
\

Running `mix deps.get` will fetch `Finch` and all its dependencies.

Now, let’s add a test file to test our server. In the test file, we will be
setting up by starting `Finch` to make HTTP calls to our server. In this
section, we will only be testing the happy paths (200 responses) of our root and
greet endpoints.

\

\codecaption{test/cowboy\_example/server\_test.exs}
```{.elixir .numberLines}
defmodule CowboyExample.ServerTest do
  use ExUnit.Case

  setup_all do
    Finch.start_link(name: CowboyExample.Finch)

    :ok
  end

  describe "GET /" do
    test "returns Hello World with 200" do
      {:ok, response} =
        :get
        |> Finch.build("http://localhost:4041")
        |> Finch.request(CowboyExample.Finch)

      assert response.body == "Hello World"
      assert response.status == 200
      assert {"content-type", "text/html"} in response.headers
    end
  end

  describe "GET /greeting/:who" do
    test "returns Hello `:who` with 200" do
      {:ok, response} =
        :get
        |> Finch.build("http://localhost:4041/greet/Elixir")
        |> Finch.request(CowboyExample.Finch)

      assert response.body == "Hello Elixir"
      assert response.status == 200
      assert {"content-type", "text/html"} in response.headers
    end

    test "returns `greeting` `:who` with 200" do
      {:ok, response} =
        :get
        |> Finch.build("http://localhost:4041/greet/Elixir?greeting=Hola")
        |> Finch.request(CowboyExample.Finch)

      assert response.body == "Hola Elixir"
      assert response.status == 200
      assert {"content-type", "text/html"} in response.headers
    end
  end
end
```

As you can see in the preceding module, we have added tests for the two
endpoints using Finch. We make calls to our server, running on port 4041 in the
test environment, with different request path and parameters. We then test the
response’s body, status, and headers.

This should give you a good idea of how to go about testing a web server. Over
the next few chapters, we will be building on top of this foundation and coming
up with better ways of testing our web server.

\newpage

### SUMMARY

When I first learned about how a web server works, I was overwhelmed with the
number of things that go into building one. Then, I decided to look at the code
of Puma, a web server written in Ruby which is also used by Rails. I was
surprised by how much more I learned by just looking into Puma than reading
articles about web servers. It is due to that reason that we are kicking off
this book by looking to Cowboy. I believe learning how the basics of Cowboy
works will better position us to build our own web server in the next few
chapters.

In this chapter, we first learned the basics of a web server along with the
client-server architecture. We also looked at the high-level architecture of
Cowboy, and learned about how some of its components like router and handlers
work.  We also added dynamic behavior to our routes by using path variables and
query parameters, followed by serving static HTML files. We finished it up by
learning how to test our routes using an HTTP client. also wanted to cover the
basics of how client-server architecture works in a web application.

### EXERCISES

Some of you might have noticed that we haven’t tested a few aspects of our web
server. Using what you have learned in this chapter:

1) Write test cases for our web server which would lead to 404 responses.

2) Write tests for the static route which responds with HTML files.

There are other (better) ways of testing an HTML response, which we haven’t
covered in this chapter. We will dig deeper into those testing methods later in
this book.
