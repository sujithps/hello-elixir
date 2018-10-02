# HelloElixir

**TODO: Add description**

## Installation

If [available in Hex](https://hex.pm/docs/publish), the package can be installed
by adding `hello_elixir` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:hello_elixir, "~> 0.1.0"}
  ]
end
```

Documentation can be generated with [ExDoc](https://github.com/elixir-lang/ex_doc)
and published on [HexDocs](https://hexdocs.pm). Once published, the docs can
be found at [https://hexdocs.pm/hello_elixir](https://hexdocs.pm/hello_elixir).



Elixir Programming: Creating a Simple HTTP Server in Elixir
Over the past year or so, I’ve heard more and more about a relatively new programming language called Elixir. The site describes the language as follows:

Elixir is a dynamic, functional language designed for building scalable and maintainable applications.

The “dynamic, functional language” piece of this description really enticed me as I’ve been wanted to dive deeper into the functional programming paradigm. In order to facilitate my learning, I set out to create a simple web service that would receive webhook requests from various external web services and take actions based on the source and content of the webhook. If you’re unfamiliar with webhooks, I recommend reading this description.

So, today I’m sharing a bit of what I’ve learned about Elixir and putting into some simple example code. All of the sample code used in this blog is available on GitHub, feel free to clone the repo or follow along with this tutorial. This sample code is simplified from the code we’re now running to consume webhooks, but this is the core of the server portion of the application. To keep this post short we won’t be going into a ton of detail on the basics of the Elixir language, if you like what you see, I recommend going to Elixir School for a more thorough introduction to the language.

Install Elixir
The first thing you’ll need to do is to get Elixir installed on your computer. Follow the official installation guide, available here. For this tutorial we’re using Elixir 1.6.4.

Set Up the Project
First, let’s create a new directory and navigate into it:

$ mkdir simple_server && cd simple_server
The we need to setup the project. Thankfully, Elixir’s built in build tool Mix will scaffold everything up for us with one simple command:

$ mix new . --sup --app simple_server
Breaking this command down: new . tells mix to create a new project structure in the current directory --sup tells mix that this is going to be a “supervised” application. Since we’re building an HTTP server, choosing to have a supervisor manage the main server process is a great idea because that supervisor process will handle the starting and stopping of the server. Elixir’s documentation describes a supervisor as follows:

A supervisor is a process which supervises other processes, which we refer to as child processes. Supervisors are used to build a hierarchical process structure called a supervision tree. Supervision trees provide fault-tolerance and encapsulate how our applications start and shutdown.

Lastly, --app simple_server tells Mix to use the name simple_server for the application/project we’re creating.

You should see output similar to this:
```
* creating README.md
* creating .formatter.exs
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/simple_server.ex
* creating lib/simple_server/application.ex
* creating test
* creating test/test_helper.exs
* creating test/simple_server_test.exs```

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    ```mix test```

Run "mix help" for more commands.
The files/directories we’ll be focusing on in this tutorial are:

lib/ - Where our application code will live
lib/simple_server/application.ex - Where we’ll describe how the application should be started and how it should be supervised
mix.exs - Where we’ll describe the configuration of our application and it’s dependencies Now that Mix has scaffolded up our project, let’s add the dependencies we need to set up our server.
Adding Dependencies
We’re going to be using several dependencies in this simple project, these dependencies will enable us to setup our HTTP server, listen on a specific port, parse JSON request bodies, and send responses to the clients that send requests to our server. Elixir uses the Hex package manager to make finding and managing external dependencies as simple as possible.

First, we need to install Hex by running the following command:

```$ mix local.hex```
We’ll be using the following dependencies in our project: Cowboy - Cowboy is a small, fast and modern HTTP server for Erlang/OTP. Plug - A specification and conveniences for composable modules between web applications Poison - An incredibly fast, pure Elixir JSON library

In order to use these dependencies, let’s add them to our mix.exs file.

Inside the deps function, add the packages we intend to use.

```defp deps do
[
  {:cowboy, "~> 1.0.0"},
  {:plug, "~> 1.5"},
  {:poison, "~> 3.1"}
]
end
Inside the application function, add our dependencies to extra_applications

def application do
[
  extra_applications: [:logger, :cowboy, :plug, :poison],
  mod: {SimpleServer.Application, []}
]
end```
Now that’s we’ve added our dependencies, we need to use Mix to download and install them. In your terminal, run this command the download the dependencies we just added to the mix.exs file:

```$ mix deps.get```
Configuring the Server
Now that we have our project set up and all the necessary dependencies installed, we need to make a few tweaks to the lib/simple_server/application.ex file to describe how the application and our HTTP server should be started.

To ensure that the HTTP server is started when our application is started, add the following line to the children array inside the start function

Plug.Adapters.Cowboy.child_spec(scheme: :http, plug: SimpleServer.Router, options: [port: 8085])
This will ensure that our server is started and will listen for HTTP connections on port 8085. The plug option is used to point to the module we’ll create to describe how to handle and response to various HTTP requests. Let’s create the router by running the following command:

```$ touch lib/simple_server/simple_server_router.ex```
Open this new file and fill it in with the following:

```defmodule SimpleServer.Router do
  use Plug.Router
  use Plug.Debugger
  require Logger

  plug(Plug.Logger, log: :debug)

  plug(:match)
  plug(:dispatch)

  # TODO: add routes!
end ```
This code defines the module SimpleServer.Router and sets up our Plug dependency. Now we need to add some routes for the server.

Adding Routes
We’re going to create 2 very simple routes and 1 default route for our server. Add the following code underneath the TODO from the previous snippet:
```
  # Simple GET Request handler for path /hello
  get "/hello" do
      send_resp(conn, 200, "world")
  end

  # Basic example to handle POST requests wiht a JSON body
  post "/post" do
      {:ok, body, conn} = read_body(conn)
      body = Poison.decode!(body)
      IO.inspect(body)
      send_resp(conn, 201, "created: #{get_in(body, ["message"])}")
  end

  # "Default" route that will get called when no other route is matched
  match _ do
      send_resp(conn, 404, "not found")
  end
 ```
The first route get "/hello" will only be “matched” when an incoming request’s path is /hello and is also a GET request. When a request matches this criteria, we’ll send a response of 200 OK and send back the text “world”.

The second route post "/post" will only be matched when an incoming request’s path is /post and is also a POST request. Let’s break down what we’re doing when this route is matched:

Plug supplies a read_body function into which we’re passing the conn struct. The conn struct is present for every requests and contains all data about the request/connection. If we successfully read the body of the request it will now be assigned to the body variable.
We’ll use our Poison dependency to decode the JSON data sent in the body of the request to an Elxir struct.
IO.inspect(body) will print the body variable to the console. This will be useful in testing/debugging the server.
We’re sending a response with a status code of 201 Created and the text created: plus the value of the message key from the body of the original request.
And finally, the default route: match _. This is essentially a “catch-all” route. If an incoming request does not match either of the above routes, it will fall through to this default route and we’ll send back a 404. the _ in this route is about eqivalent to anything. This is a very simpl example of one of Elixir’s most touted features: pattern matching; this is an amazing feature of Elixir, but outside the scope of this tutorial.

That’s it! Time to fire up the server and send some test requests!

Running the Server
To start the server and have our HTTP server listening on localhost:8085, run the following command:

```$ iex -S mix```

IEx is Elixir’s interactive shell. The -S mix option tells IEx to execute the mix.exs script in the interactive shell. As you’ve seen, this script holds the main configuration of our app and will start it up when executed. You’ll probably see some complication-related output similar to this truncated output:

```Erlang/OTP 20 [erts-9.3] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]
...
==> simple_server
Compiling 2 files (.ex)
Generated simple_server app```

Now that we’re up and running, let’s send a simple GET request. You can do this with curl in your terminal or by going to http://localhost:8085/hello in your browser. If you’re using curl, run this command:

```$ curl -v "http://localhost:8085/hello"```
If everything is working as it should be, you should get back the text world, and in the terminal where IEx is running, you should see some debug logging like this:

```13:25:45.811 [debug] GET /hello
13:25:45.815 [debug] Sent 200 in 3ms```

Let’s do another GET request, but this time we’ll test our route matching and default route by sending a request to http://localhost:8085/thisshouldbea404:

```$ curl -v "http://localhost:8085/thisshouldbea404"```

We should get a response of 404 and the text not found. In our IEx terminal, we should see output like this:

``` [debug] GET /thisshouldbea404
 [debug] Sent 404 in 19µs
yeah… that’s 19 microseconds (or 0.019 milliseconds)!!!```

Finally, we need to test our POST request. Our route for post "/post" isn’t exactly “production-ready” as it’s expecting a very specific request body, and as such our application will crash if it doesn’t receive a JSON body with a "message" key in it.

For extra credit, test out sending JSON that will cause the application to crash, you’ll see the benefit of using the supervisor to run our server.

To send our POST run the following command (this assumes you have curl installed):

```$ curl -v -H 'Content-Type: application/json' "http://localhost:8085/post" -d '{"message": "hello world" }'```

If everything worked as it was supposed to, you would get the following response from your request: created: hello world. You’ll also see the following output in your IEx terminal:

``` [debug] POST /post
%{"message" => "hello world"}
 [debug] Sent 201 in 17ms```

