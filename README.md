# [Martini][1] Strict Mode


[1]: //github.com/go-martini/martini

This repo contains a set of utilities that help you make a well-behaving,
strict API using the awesome Martini framework. The are tested and ready-to-use
handlers for the following responses:

* 404 Not Found with empty body
* 405 Method Not Allowed + Allow header
* 406 Not Acceptable
* 415 Unsupported Media Type

There is also a helper function to negotiate the request content type,
according to [RFC2616 Section 14][4].


## Usage

Here is a complete working example that uses the `strict` package together with
the [`render`][2] contrib package. In particular, the `render` and [`bind`][3]
contrib packages work very nicely together with `strict`.

[2]: https://github.com/martini-contrib/render
[3]: https://github.com/martini-contrib/bind

```go
package main

import (
	"net/http"

	"github.com/attilaolah/strict"
	"github.com/go-martini/martini"
	"github.com/martini-contrib/render"
)

type animal struct {
	ID   int
	Name string
}

var zoo = []animal{animal{1, "Alex"}, animal{2, "Marty"}}

func main() {
	m := martini.Classic()
	m.Use(strict.Strict)
	m.Use(render.Renderer())

	m.Get("/zoo", strict.Accept("application/json", "text/html"), getZOO)
	m.Post("/zoo", strict.ContentType("application/json", "text/xml", ""), postZOO)

	m.Router.NotFound(strict.MethodNotAllowed, strict.NotFound)

	m.Run()
}

func getZOO(n strict.Negotiator, r render.Render) {
	if n.Accepts("application/json") > n.Accepts("text/html") {
		r.JSON(http.StatusOK, zoo)
		return
	}
	r.HTML(http.StatusOK, "zoo", zoo)
}

func postZOO(n strict.Negotiator, r *http.Request) (int, string) {
	if n.ContentType("application/json") {
		// JSON-encoded body
	} else if n.ContentType("text/xml") {
		// XML-encoded body
	}
	return http.StatusCreated, ""
}
```


#### The `strict.Strict` handler

By telling Martini to `m.Use(strict.Strict)`, the `strict.Negotiator` interface
becomes availabe in handlers. The negotiator can be used for two things:

* It can check whether the client accepts a content type by parsing the
  `Accept` header (if present) and returning the corresponding `q` value. See
  [RFC2616 Section 14][4] for details.

[4]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html

Calling `n.Accepts("text/html")` will return the client's preference to accept
the `text/html` content type. This works even if the client accepts `text/*` or
`*/*`, or if the Accept header is missing, in which case `q` defaults to 1.

* It can also check the content type of the request.

Calling `n.ContentType("application/json", "text/html")` will return `true` if
the Content-Type header was set to either `application/json` or `text/html`. It
will also work if the header includes the charset, e.g. with `application/json;
charset=UTF-8`.


#### The `strict.ContentType` handler factory

In the above example, we add `strict.ContentType("application/json",
"text/xml", "")` to the list of handlers when calling `m.Post(…)`.
This will create a handler that will write a *415 Unsupported Media Type*
response if the request content type is none of the acceptable content types
passed in as arguments. The empty string means that we also want to accept
requests with no `Content-Type` header set.


#### The `strict.Accept` handler factory

In the above example, we add `strict.Accept("application/json", "text/html")`
to the list of handlers when calling `m.Get(…)`. This will create a handler
that will write a *406 Not Acceptable* response if the Accept request header
does not permit any of the supported content types we pass in as arguments.


#### The `strict.MethodNotAllowed` handler

Passing `strict.MethodNotAllowed` to `m.Router.NotFound(…)` will tell Martini
to return a *405 Method Not Allowed* instead of a *404 Not Found* when an route
is called with a method that it is not registered with. In addition to the
response status code, the Allow header will be set to the list of allowed
methods.

Note that `strict.MethodNotAllowed` will never write a 404 response, so you
have to also add a not found handler when calling `m.Router.NotFound(…)`.
A good candidate for that is `strict.NotFound`.


#### The `strict.NotFound` handler

It is similar to `http.NotFound`, except it does not write a response body. The
response will be an empty *404 Not Found* response.


#### Constants


The following constant is included for convenience:

```go
const StatusUnprocessableEntity = 422
```
