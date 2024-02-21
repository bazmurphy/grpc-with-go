# GRPC in Go

`go mod init github.com/bazmurphy/grpc-with-go`

create the proto file `invoicer.proto`
this is the description of the API

to define the syntax we will use

```proto
syntax = "proto3";
```

`proto3` is the version of the Protocol Buffer language specification

when we have a GRPC Service the messages that we will exchange between the server and the client will be encoded using protocol buffers and they have a versioning system, so we say here we will use version 3 of protocol buffers

now we define our `service`

```proto
service Invoicer {

}
```

and inside we define the endpoints - we define the RPC methods

```proto
service Invoicer {
  // [1] the name of the method
  // [2] inside the 1st parenthesis specify the request message
  // [3] returns keyword
  // [4] specify the response message
  rpc Create(CreateRequest) returns (CreateResponse)
}
```

but those messages do not exist yet, so we have to define them

```proto
// create our first message type
message CreateRequest {
  // we put the fields we can have in our message
  // what will be in our input
}

message CreateResponse {
  // and what will be in our output
}
```

now we can fill in the fields, their types and their value is their position

```proto
// create our first message type
message CreateRequest {
  // we put the fields we can have in our message
  // what will be in our input#
  // we make an amount, give it a type and define its position in the message
  int64 amount = 1;
    // and a curency (string) in position 2
  string currency = 2;
}

message CreateResponse {
  // and what will be in our output
  // we return an invoice as pdf (type bytes) (return binary data - sequence of bytes - binary representation of the pdf)
  bytes pdf = 1;
  // invoice as a doc file
  bytes docx = 2;
}
```

we can create other messages that can be used inside other messages, so we can instead abstract away the Amount and then it is re-usable

```proto
// we can abstract this away into another message "Amount" for re-usability
message Amount {
  int64 amount = 1;
  string currency = 2;
}

message CreateRequest {
  // and then use that message type here instead
  // we use the message TYPE "Amount" and name it "amount" and set it at position 1
  Amount amount = 1;

  // who the invoice was received from
  string from = 2;

  // who the invoice will be sent to
  string to = 3;
}
```

at the top of the file we can also add the `option` go_package
this specifies the Go package where the generated code will be placed
tells the protocol buffer compiler (protoc) to generate Go code
and place it in the package when generating Go code from this .proto file

```proto
// note the subdirectory
option go_package = "github.com/bazmurphy/grpc-with-go/invoicer";
```

Now our `.proto` file is ready. We need to generate the Code for the Server and the Client.
In order to do that we will use `protoc`

https://grpc.io/docs/protoc-installation

The protocol buffer compiler, `protoc`, is used to compile `.proto` files, which contain service and message definitions.

```sh
sudo apt install -y protobuf-compiler
```

check the version

```sh
protoc --version
libprotoc 3.21.12
```

Take a message and encode it in the Protocol Buffers way and also decode it
And also generate the Code to create our gRPC server

Call the command `protoc`
pass in the first flag:
`--go_out=` the package we will generate so `invoicer` because above we used `/invoicer`
`--go_opt=` the file that will be generated will be in the same relative directory as the input file (the input file is the `.proto` file)
`--go-grpc_out=` we want our code to be generated into the package `invoicer`
`--go-grpc_opt=` again we specify the path as the same as where the `.proto` file is located
then we specify the input file `invoicer.proto`

(!) we need to make a directory `/invoicer` before running the command

(!!) we also need to install the following two plugins:

```sh
go get google.golang.org/protobuf/cmd/protoc-gen-go
go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
```

(!!!) update the `PATH` (in `~/.profile`) so the `protoc` compiler can find the pluginswe just installed

```sh
export PATH="$PATH:$(go env GOPATH)/bin"
```

```sh
protoc \
--go_out=invoicer \
--go_opt=paths=source_relative \
--go-grpc_out=invoicer \
--go-grpc_opt=paths=source_relative \
invoicer.proto
```

this creates `invoicer_grpc.pb.go` and `invoicer.pb.go` files

we now do a `go get -u google.golang.org/grpc`
(`-u` to get the latest version)

and then do a `go mod tidy` to ensure the go.mod matches the source code in the module

now we can create our `main.go` file

since we are making a gRPC server we have to make a connection in order to receive requests and send responses

we do that using `net.Listen()`
`net` is a standard package from the standard library
it takes as its first parameter "network" which in this case we use `"tcp"`
and then as its second parameter "address" which in this case we give the port `":8080"`
and `net.Listen()` will return a listener and an error

```go
func main() {
  listener, err := net.Listen("tcp", ":8080")
  if err != nil {
    // if we have an error we will exit the program
    log.Fatalf("cannot create the listener: %s", err)
  }
}
```

we make a `Makefile`

```Makefile
generate_grpc_code:
	protoc \
	--go_out=invoicer \
	--go_opt=paths=source_relative \
	--go-grpc_out=invoicer \
	--go-grpc_opt=paths=source_relative \
	invoicer.proto
```

## Summary

1. Create a `.proto` file - the definition of your API

   - define messages
   - define a service
     - define rpc methods

2. Then you generate the Go code that will be used - with `protoc`

   - to encode/decode messages with Protocol Buffers format
   - to handle incoming gRPC requests and answer them

3. Each time you change the `.proto` file you need to regenerate the code

4. Do not touch the generated code, your changes will be removed on the next generation
