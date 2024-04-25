# Onyx-HTTP-Server

While the name is certainly not catchy, this package provides 
a simple, composable architecture for creating HTTP applications,
similar to [ExpressJS](https://expressjs.com). This package aims
to be relatively simple and unopinionated, allowing you the
flexibility to create the application you want.


## Pipelines

This package uses a pipeline based architecture to process
HTTP requests. Each request is passed into a pipeline, for
there it can be routed, and processed in a piece-meal way.
This allows for maximum composability of existing pieces of
code. Here is an example.

```onyx
use http.server

main :: () {
    pipeline := server.pipeline();

    pipeline->pipe((req, res) => {
        printf("{} was requested!\n", req.base_endpoint);
    });

    pipeline->pipe((req, res) => {
        res->status(200);
        res->html("Working!");
    });

    app := server.tcp(&pipeline);
    app->serve(8080);
}

```

In this simple application, every request is routed through two
pipes. The first pipe simply prints out the endpoint that was
requested. The second pipe responds to the event with a status
of 200, and the HTML text of "Working!".

```onyx

pipelines :: () {
    pipeline := http.server.pipeline();

    pipeline->add((req, res) => {
        // ...
    });
}

```

So far, the only parts of a pipeline shown are simple functions.
However, stages of a pipeline (henceforth called a Pipe) can do
much more. Anything that defines an overload for http.convert_to_pipe()
can be used as a pipe. There are several pipes included with this
package (described below). In fact, pipelines themselves can be
used as pipes.

The algorithm governing the pipeline process is very simple. There
are two simple rules that you need to know:

- Every pipe will be executed, until the response is marked as completed.
- A pipe can set execute_after_request if it should still be run
    after the response has been marked completed. For example, a logger.

Using this algorithm, a simple pipe that ensures a user is logged in,
or that a CSRF token is valid, can be placed before any route handler
that requires that functionality.


## Router

Obviously, not all requests should be handled the same. For this
reason, there is the router. The router simply matches all or part
of the requested endpoint, and sends the request down one of any number
of pipelines. This is an important distinction: routers are just pipes
that send the request to another pipeline. Here is a simple example
of a router.

```onyx
use http

// A typical HTTP application
main :: () {
    router := http.server.router();

    // You can specify the HTTP Method using the enum value.
    router->route(.GET, "/index", (req, res) => {
        res->status(200);
        res->html("Index");
    });

    // You can also call get, post, etc. directly.
    router->get("/test", (req, res) => {
        res->status(200);
        res->html("Test");  
    });

    app := http.server.tcp(&router);
    app->serve(8080);
}

```

The code above should be relatively simple if you have worked with
a modern HTTP server framework before. The only "gotcha" in the code
would be that you have to pass a pointer to a router, and not the
router directly to app->add(). Otherwise, this looks like it could
be PHP.


The above code declares routes as you probably would in something
like ExpressJS. This can create a lot of code clutter in main() or
wherever you introduce all the routes for your application. To combat
this, this package provides an alternate way to automatically register
routes based on tags on procedures throughout the program. In this way,
it feels more like programming in Flask with Python's decorator syntax.

To use this way, simply declare a procedure with tag of type http.route.
Then, after you create your router, call router->collect_routes(). This
will automcatically find all http.route tags and add them to the router.
The "gotcha" with this method is you HAVE to specify the types of the
two arguments, req and res, which must be ^http.Request and ^http.Response
respectively.

```onyx

// This prevents you from having to type
// http.Request and http.Response over and
// over again...
use http.server {
    package,
    Req :: Request,
    Res :: Response,
}

#tag http.route.{.GET, "/index"}
(req: ^Req, res: ^Res) {
    res->status(200);
    res->html("From a tag!");
}

main :: () {
    router := server.router();
    router->collect_routes();

    app := server.tcp(&router);
    app->serve(8080);
}

```
