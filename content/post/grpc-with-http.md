+++
title = "Serving gRPC and HTTP services on the same port"
date = "2017-03-28T00:00:00Z"
tags = ["golang","grpc"]

clearReading = true
autoThumbnailImage = false
coverImage = "/images/torres01.jpg"
coverCaption = "Torres del Paine, Chile"
comments = false
showSocial = false
showPagination = false
+++

Recently I needed to build a service that would host both an HTTPS REST API
and a gRPC service. I was already using [Let's Encrypt](https://letsencrypt.org/)
[certificates for the gRPC service](/post/acme) and while it may be technically
possible to reuse the same certificates on multiple ports, having
a single open port is preferable.

<!--more-->

A Google search for turned up a [closed github issue](https://github.com/grpc/grpc-go/issues/75).
And a search through the [gRPC Go documentation](https://godoc.org/google.golang.org/grpc)
revealed that a `net.ServeHTTP` method was added to the `grpc.Server`
[in 2016](https://github.com/grpc/grpc-go/pull/514). These also pointed to a still open
bug to [document how to use ServeHTTP](https://github.com/grpc/grpc-go/issues/549).
So how, exactly, do we use such a ServeHTTP method?

My initial naive attempt used the `http.ServeMux` which does not
work because gRPC services expect to sit at the root path in HTTP
requests. Reading through all of the comments in issue #514, I found
[one implementation](https://github.com/grpc/grpc-go/issues/549#issuecomment-191455642)
that works. Here it is edited for brevity:

<pre><code class="go">// grpcHandlerFunc returns an http.Handler that delegates to
// grpcServer on incoming gRPC connections or otherHandler
// otherwise. Copied from cockroachdb.
func grpcHandlerFunc(rpcServer *rpc.Server, other http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ct := r.Header.Get("Content-Type")
        if r.ProtoMajor == 2 && strings.Contains(ct, "application/grpc") {
            rpcServer.ServeHTTP(w, r)
        } else {
            other.ServeHTTP(w, r)
        }
    })
}
</code></pre>

This works perfectly---assuming I suppose that you are not serving
HTTP/2 application/grpc content over your REST API but then why
would you?

One issue that I would like to further explore is whether it is possible
to combine this with the [gRPC Gateway](https://github.com/grpc-ecosystem/grpc-gateway)
plugin. That plugin generates a REST API for a gRPC service. It should
be possible to bind the generated gateway to the same port serving the gRPC
service. However, I have not yet tried that.


### Endnote -- Multiple gRPC services

I've seen the question asked in multiple places "Can I host multiple
gRPC services on the same port?" The answer, for Go at least, is
yes.

The generated code for the "Greeter" service provides a function
`RegisterGreeterService` to which you pass the `grpc.Server` and your
implementation of the "GreeterService" interface. That function binds your
implementation to that service name. The only restriction here is
that *you cannot have two services with the same name on the same
port.* I think this is fairly sensible.

Let's take the [two example services](https://github.com/grpc/grpc-go/tree/master/examples)
provided in the gRPC source code: "Greeter" and "RouteGuide". We can setup a
single network socket that listens for both services:

<pre><code class="go">package server

import (
    "fmt"
    "net"

    "golang.org/x/net/context"
    "google.golang.org/grpc"

    pbHello "google.golang.org/grpc/examples/helloworld/helloworld"
    pbRoute "google.golang.org/grpc/examples/route_guide/routeguide"
)

// Note that you have to provide your own implementations of each service.
var route pbRoute.RouteServer
var hello pbHello.GreeterServer

func Serve() error {
    lis, err := net.Listen("tcp", fmt.Sprintf(":%d", 443))
    if err != nil {
        return nil
    }

    // You should setup TLS certificates here
    var conf *tls.Config
    creds := credentials.NewTLS(conf)
    opts := []grpc.ServerOption{grpc.Creds(creds)}

    grpcServer := grpc.NewServer(opts...)

    pbRoute.RegisterRouteGuideServer(grpcServer, route)
    pbHello.RegisterGreeterServer(grpcServer, hello)

    return grpcServer.Serve(lis)
}</code></pre>
