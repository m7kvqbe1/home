+++
title = 'Unix Sockets and Protobuf'
date = 2024-04-05T16:49:10+01:00
draft = false
+++

Recently, I explored Unix Sockets and Protobuf with Go. Here's a brief overview of some of the things I learned.

Find the related source code here:

https://github.com/m7kvqbe1/unix-sockets-protobuf-play

## Unix Sockets: Inter-Process Communication

Unix Sockets offer a method for inter-process communication (IPC) on the same machine. It's faster and leaner than leveraging a network protocol like TCP. They're straightforward to set up in Go:

```go
listener, _ := net.Listen("unix", "/tmp/example.sock")

conn, _ := net.Dial("unix", "/tmp/example.sock")
```

Lets accept incoming connections and handle each one with its own Go routine:

```go
for {
  conn, err := listener.Accept()
  if err != nil {
    fmt.Println("Error accepting:", err.Error())
    continue
  }

  go handleServerConnection(conn)
}
```

We can transmit and receive from the socket like so:

```go
if _, err = conn.Write(data); err != nil {
  fmt.Println("Client write error:", err)
  return
}

buf := make([]byte, 1024)

n, err := conn.Read(buf)
if err != nil {
  fmt.Println("Server read error:", err)
  return
}
```

We read a stream from the socket into a buffer.

Lets read it in chunks to make sure we can handle a stream of arbitrary size:

```go
buf := make([]byte, 1024)
var data []byte

for {
  // Reads the stream for len of buf,
  // n = number bytes read into the buffer
  n, err := conn.Read(buf)
  if err != nil {
    // Finished reading the stream
    if err == io.EOF {
      break
    }

    fmt.Println("Server read error:", err)
    return
  }

  // ... Spreads buffer into the data slice
  data = append(data, buf[:n]...)
}
```

## Protobuf: Data Serialization

Protocol Buffers (protobuf) by Google are used for serializing structured data. Defining a data structure is also easy:

```protobuf
syntax = "proto3";

package main;

option go_package = "./pb";

message SimpleMessage {
  string content = 1;
}
```

Using `protoc`, this schema is turned into Go code, allowing for serialization and deserialization of Go structs.

```bash
protoc --go_out=. --go_opt=paths=source_relative ./pb/message.proto
```

## Implementing Protobuf over Unix Sockets

Combining Unix Sockets with protobuf involved serializing data into protobuf format and transmitting it through the socket.

The [google.golang.org/protobuf/proto](https://google.golang.org/protobuf/proto) package provides Marshal and Unmarshal methods similar to `encoding/json` from the Go standard library.

Lets encode it in our client:

```go
msg := &pb.SimpleMessage{Content: "Hello from client"}

data, err := proto.Marshal(msg)
if err != nil {
  fmt.Println("Client protobuf encode error:", err)
  return
}

if _, err = conn.Write(data); err != nil {
  fmt.Println("Client write error:", err)
  return
}
```

We can then decode it like so on our server:

```go
var msg pb.SimpleMessage

if err := proto.Unmarshal(data, &msg); err != nil {
  fmt.Println("Server protobuf decode error:", err)
  return
}
```
