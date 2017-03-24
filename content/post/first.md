+++
date = "2017-03-09T23:40:13Z"
draft = false
title = "gRPC streams and Golang io"
+++

One common question I hear on forums is how to implement file upload
or download with gRPC. With traditional HTTP servers this is trivial.
gRPC supports bidirectional streaming of messages between a client
and server. Yet this streaming must be of well-defined protocol-buffer
messages.

The `io.Reader` and `io.Writer` interfaces in Go's standard IO library
provide powerful abstractions that can be plugged into many different
components. Unfortunately there is no easy way to attach them to a
gRPC server or client.

Let's begin with a simple file upload service defined in the following
protoc file:

```protoc
// fileserver.protoc
syntax = 'proto3';
package fileserver;

message File {
    string name    = 1;
    int16  mode    = 2;
    bytes  payload = 3;
}

message UploadFileResponse {
    int64  size = 1;
    string SHA1 = 2;
}

service FileServer {
    rpc UploadFile(stream File) returns (UploadFileResponse) {}
}

```
The client uploads a file with name and permission information
to the server. The server responds with the total size of the file
in bytes and a SHA1 checksum for integrity verification.

```shell
$ protoc -I . ./fileserver.proto --go_out=plugins=grpc:.
```

Invoking the protoc compiler with this will generate a gRPC client
and interfaces for the server implement. Now we would like some way
to generate an `io.Writer` for the client side of the RPC call and
an `io.Reader` for the server. The writer is the simplist so we'll
start with it:

```go
package stream

type Container interface {
	SetPayload(b []byte)
	GetPayload() (b []byte)
}

type Writer io.Writer

type writer struct {
	stream grpc.Stream
	req    Container
}

func (w *writer) Write(b []byte) (n int, err error) {
	w.req.SetPayload(b)
	return len(b), w.stream.SendMsg(w.req)
}

func NewWriter(s grpc.Stream, r Container) (w Writer) {
	return &writer{stream: s, req: r}
}

type WriteCloser interface {
    io.WriteCloser
    Recv() (interface{}, error)
}

type writeCloser struct {
    *writer
    rsp interface{}
}

func (w *writeCloser) Close() (err error) {
    if err := w.stream.CloseSend(); err != nil {
        return err
    }
    return w.stream.RecvMsg(w.rsp)
}

func (w *writeCloser) Recv() (rsp interface{}, err error) {
    if w.rsp == nil {
        err = errors.New("No response.")
    }
    return w.rsp, err
}

func NewWriteCloser(s grpc.ClientStream,
    r Container, rsp interface{}) (w WriteCloser) {
	return &writeCloser{stream: s, req: r, rsp: rsp}
}
```

//TODO clean this up and verify the NewWriteCloser

The `Container` defines how we will interact with the message types.
*Note that in this case the non-payload metadata will be sent with
each iteration; you might want to zero out those fields in your
implementation to save bandwidth.*

If each message type that we want to use for streaming
has a `payload` field the `GetPayload()` function will be implemented
for us. We will have to manually write a `SetPayload([]byte)` function
in the same package as the gRPC server:

```go
package fileserver

func (r *File) SetPayload(b []byte) {
    r.Payload = b
}
```

Simple enough. I have 




