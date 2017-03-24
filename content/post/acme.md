+++
date = "2017-03-24T00:00:00Z"
title = "ACME Let's Encrypt Certificates for gRPC services"

clearReading = true
autoThumbnailImage = false
coverImage = "/images/window.jpg"
coverCaption = "Window, Castillo de San Cristóbal, San Juan"
coverSize = "partial"
comments = false
showSocial = false
showPagination = false
+++

In this post I'll show you how to configure a Go gRPC client and server
using Let's Encrypt TLS certificates. This ended up being very simple thanks
to the [autocert package](https://godoc.org/golang.org/x/crypto/acme/autocert).
*Note that at this point that package is not stable, so you should vendor it in your code.*


Autocert implements the `tls.GetCertificate` function and requires a directory to
cache certificates in to avoid excessive calls to Let's Encrypt.

<pre><code class="go">package acme // import github.com/devoid/example/acme
import (
    "crypto/tls"

    "golang.org/x/crypto/acme/autocert"
)

const contactEmail = "yourname@example.com"

func GetTLS(host, cacheDir string) (*tls.Config, error) {
    manager := autocert.Manager{
        Prompt: autocert.AcceptTOS,
        Cache: autocert.DirCach(cacheDir),
        HostPolicy: autocert.HostWhitelist(host),
        Email: contactEmail,
    }
    return &tls.Config{ GetCertificate: manager.GetCertificate }, nil
}
</code></pre>

On the gRPC server side, you simply use this `*tls.Config` as a
transport credentials object. On the client side, the zero-initialized
`&tls.Config{}` is sufficient since this defaults to the operating-system's
default root certificates.

{{< tabbed-codeblock grpc >}}
    <!-- tab server -->
package server

import(
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"

    "github.com/devoid/example/acme"
)

func NewServer(host, cacheDir string) error {

    tls, err := acme.GetTLS(host, cacheDir)
    if err != nil {
        return err
    }

    creds := credentials.NewTLS(tls)
    server := grpc.NewServer(Creds(creds))

    // Register your gRPC services now
    
    lis, err := net.Listen(":https")
    if err != nil {
        return err
    }

    return server.Serve(lis)
}
    <!-- endtab -->
    <!-- tab client -->
// github.com/devoid/example/cmd/client
package main 

import (
    "net"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)


func main() {
    // Use the operating system default root certificates.
    opts := []grpc.DialOption{
        grpc.WithTransportCredentials(credentials.NewTLS(&tls.Config{})),
    }

    dialstr := net.JoinHostPort("example.com", "443")
    conn, err := grpc.Dial(dialstr, opts...)
    if err != nil {
        os.Exit(1)
    }
    // Use conn now to create gRPC client.
    // Then use client...

    return
}
    <!-- endtab -->
{{< /tabbed-codeblock >}}

