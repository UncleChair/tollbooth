[![GoDoc](https://godoc.org/github.com/didip/tollbooth?status.svg)](http://godoc.org/github.com/didip/tollbooth)
[![license](http://img.shields.io/badge/license-MIT-red.svg?style=flat)](https://raw.githubusercontent.com/didip/tollbooth/master/LICENSE)

## Tollbooth

This is a generic middleware to rate-limit HTTP requests.

**NOTE 1:** This library is considered finished.

**NOTE 2:** Major version changes are backward-incompatible. `v2.0.0` streamlines the ugliness of the old API.

## Versions

**v1.0.0:** This version maintains the old API but all the thirdparty modules are moved to their own repo.

**v2.x.x:** Brand-new API for the sake of code cleanup, thread safety, & auto-expiring data structures.

**v3.x.x:** Apparently we have been using golang.org/x/time/rate incorrectly. See issue #48. It always limits X number per 1 second. The time duration is not changeable, so it does not make sense to pass TTL to tollbooth.

**v4.x.x:** Float64 for max requests per second

**v5.x.x:** go.mod and go.sum

**v6.x.x:** Replaced `go-cache` with `github.com/go-pkgz/expirable-cache` because `go-cache` leaks goroutines.

**v7.x.x:** Replaced `time/rate` with `embedded time/rate` so that we can support more rate limit headers.

**v8.x.x:** Address `RemoteIP` vulnerability concern by replacing `SetIPLookups` with `SetIPLookup`, an explicit way to pick the IP address. New `HTTPMiddleware` function which is compatible with standard routers.

## Five Minute Tutorial

```go
package main

import (
	"net/http"

	"github.com/didip/tollbooth/v8/limiter"
)

func HelloHandler(w http.ResponseWriter, req *http.Request) {
	w.Write([]byte("Hello, World!"))
}

func main() {
	// Create a request limiter per handler.
	lmt := tollbooth.NewLimiter(1, nil)

	// New in version >= 8, you must explicitly define how to pick the IP address.
	lmt.SetIPLookup(limiter.IPLookup{
		Name:           "X-Real-IP",
		IndexFromRight: 0,
	})

	// New in version >= 8, HTTPMiddleware is a standard router compatible alternative to the previously used LimitFuncHandler.
	http.Handle("/", tollbooth.HTTPMiddleware(lmt)(http.HandlerFunc(HelloHandler)))
	// Old syntax:
	// http.Handle("/", tollbooth.LimitFuncHandler(lmt, HelloHandler))

	http.ListenAndServe(":12345", nil)
}

```

## Features

1. Rate-limit by request's remote IP, path, methods, custom headers, & basic auth usernames.

    ```go
    import (
        "time"

        "github.com/didip/tollbooth/v8"
        "github.com/didip/tollbooth/v8/limiter"
    )

    lmt := tollbooth.NewLimiter(1, nil)

    // or create a limiter with expirable token buckets
    // This setting means:
    // create a 1 request/second limiter and
    // every token bucket in it will expire 1 hour after it was initially set.
    lmt = tollbooth.NewLimiter(1, &limiter.ExpirableOptions{DefaultExpirationTTL: time.Hour})

    // New in version >= 8, you must explicitly define how to pick the IP address.
    // If IP address cannot be found, rate limiter will not be activated.
    lmt.SetIPLookup(limiter.IPLookup{
        // The name of lookup method.
        // Possible options are: RemoteAddr, X-Forwarded-For, X-Real-IP, CF-Connecting-IP
        // All other headers are considered unknown and will be ignored.
        Name: "X-Real-IP",

        // The index position to pick the ip address from a comma separated list.
        // The index goes from right to left.
        //
        // When there are multiple of the same headers,
        // we will concat them together in the order of first to last seen.
        // And then we pick the IP using this index position.
        IndexFromRight: 0,
    })

    // In version >= 8, lmt.SetIPLookups and lmt.GetIPLookups are removed.

    // Limit only GET and POST requests.
    lmt.SetMethods([]string{"GET", "POST"})

    // Limit based on basic auth usernames.
    // You add them on-load, or later as you handle requests.
    lmt.SetBasicAuthUsers([]string{"bob", "jane", "didip", "vip"})
    // You can remove them later as well.
    lmt.RemoveBasicAuthUsers([]string{"vip"})

    // Limit request headers containing certain values.
    // You add them on-load, or later as you handle requests.
    lmt.SetHeader("X-Access-Token", []string{"abc123", "xyz098"})
    // You can remove all entries at once.
    lmt.RemoveHeader("X-Access-Token")
    // Or remove specific ones.
    lmt.RemoveHeaderEntries("X-Access-Token", []string{"limitless-token"})

    // By the way, the setters are chainable. Example:
    lmt.SetMethods([]string{"GET", "POST"}).
        SetBasicAuthUsers([]string{"sansa"}).
        SetBasicAuthUsers([]string{"tyrion"})
    ```

2. Compose your own middleware by using `LimitByKeys()`.

3. Header entries and basic auth users can expire over time (to conserve memory).

    ```go
    import "time"

    lmt := tollbooth.NewLimiter(1, nil)

    // Set a custom expiration TTL for token bucket.
    lmt.SetTokenBucketExpirationTTL(time.Hour)

    // Set a custom expiration TTL for basic auth users.
    lmt.SetBasicAuthExpirationTTL(time.Hour)

    // Set a custom expiration TTL for header entries.
    lmt.SetHeaderEntryExpirationTTL(time.Hour)
    ```

4. Upon rejection, the following HTTP response headers are available to users:

    * `X-Rate-Limit-Limit` The maximum request limit.

    * `X-Rate-Limit-Duration` The rate-limiter duration.

    * `X-Rate-Limit-Request-Forwarded-For` The rejected request `X-Forwarded-For`.

    * `X-Rate-Limit-Request-Remote-Addr` The rejected request `RemoteAddr`.

   Upon both success and rejection [RateLimit](https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-ratelimit-headers) headers are sent:

   * `RateLimit-Limit` The maximum request limit within the time window (1s).

   * `RateLimit-Reset` The rate-limiter time window duration in seconds (always 1s).

   * `RateLimit-Remaining` The remaining tokens.

5. Customize your own message or function when limit is reached.

    ```go
    lmt := tollbooth.NewLimiter(1, nil)

    // New in version >= 8, you must explicitly define how to pick the IP address.
    lmt.SetIPLookup(limiter.IPLookup{
        Name:           "X-Forwarded-For",
        IndexFromRight: 0,
    })

    // Set a custom message.
    lmt.SetMessage("You have reached maximum request limit.")

    // Set a custom content-type.
    lmt.SetMessageContentType("text/plain; charset=utf-8")

    // Set a custom function for rejection.
    lmt.SetOnLimitReached(func(w http.ResponseWriter, r *http.Request) { fmt.Println("A request was rejected") })
    ```

6. Tollbooth does not require external storage since it uses an algorithm called [Token Bucket](http://en.wikipedia.org/wiki/Token_bucket) [(Go library: golang.org/x/time/rate)](https://godoc.org/golang.org/x/time/rate).

## Other Web Frameworks

Sometimes, other frameworks require a little bit of shim to use Tollbooth. These shims below are contributed by the community, so I make no promises on how well they work. The one I am familiar with are: Chi, Gin, and Negroni.

* [Chi](https://github.com/didip/tollbooth_chi)

* [Echo](https://github.com/didip/tollbooth_echo)

* [FastHTTP](https://github.com/didip/tollbooth_fasthttp)

* [Gin](https://github.com/didip/tollbooth_gin)

* [GoRestful](https://github.com/didip/tollbooth_gorestful)

* [HTTPRouter](https://github.com/didip/tollbooth_httprouter)

* [Iris](https://github.com/didip/tollbooth_iris)

* [Negroni](https://github.com/didip/tollbooth_negroni)

## My other Go libraries

* [ErrStack](https://github.com/didip/errstack): A small library to combine errors and also display filename and line number.

* [Stopwatch](https://github.com/didip/stopwatch): A small library to measure latency of things. Useful if you want to report latency data to Graphite.

* [LaborUnion](https://github.com/didip/laborunion): A dynamic worker pool library.

* [Gomet](https://github.com/didip/gomet): Simple HTTP client & server long poll library for Go. Useful for receiving live updates without needing Websocket.

## Contributions

Before sending a PR with code changes, please make sure altered code is covered with tests which are passing, and that golangci-lint shows no errors.

To check the linter output, [install it](https://golangci-lint.run/usage/install/#local-installation) and then run `golangci-lint run` in the root directory of the repository.
