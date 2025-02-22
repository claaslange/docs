# Dapr and gRPC

Dapr implements both an http API and a gRPC interface.
gRPC is useful for low-latency, high performance scenarios and has deep language integration using the proto clients.

You can find a list of autogenerated client [here](https://github.com/dapr/docs#sdks).

The Dapr runtime implements a [proto service](https://github.com/dapr/dapr/blob/master/pkg/proto/dapr/dapr.proto) that apps can communicate with via gRPC.
In addition to talking to Dapr via gRPC, Dapr can communicate with an application via gRPC.

To do that, the app simply needs to host a gRPC server and implement the [Dapr client service](https://github.com/dapr/dapr/blob/master/pkg/proto/daprclient/daprclient.proto).

## Configuring Dapr to communicate with app via gRPC

### Kubernetes

On Kubernetes, set the following annotations in your deployment YAML:

<pre>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        dapr.io/enabled: "true"
        dapr.io/id: "myapp"
        <b>dapr.io/protocol: "grpc"
        dapr.io/port: "5005"</b>
...
</pre>

This tells Dapr to communicate with your app via gRPC over port `5005`.

### Standalone

When running in standalone mode, use the `--protocol` flag to tell Dapr to use gRPC to talk to the app:

```
dapr run --protocol grpc --app-port 5005 node app.js
```

## Invoking Dapr - Go example

The following steps will show you how to create a Dapr client and call the Save State operation on it:

1. Import the package

```go
package main

import (
    pb "github.com/dapr/go-sdk/dapr"
)
```

2. Create the client

```go
    // Get the Dapr port and create a connection
	daprPort := os.Getenv("DAPR_GRPC_PORT")
	daprAddress := fmt.Sprintf("localhost:%s", daprPort)
	conn, err := grpc.Dial(daprAddress, grpc.WithInsecure())
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()

	// Create the client
	client := pb.NewDaprClient(conn)
```

3. Invoke the Save State method

```go
_, err = client.SaveState(context.Background(), &pb.SaveStateEnvelope{
		Requests: []*pb.StateRequest{
			&pb.StateRequest{
				Key: "myKey",
				Value: &any.Any{
					Value: []byte("My State"),
				},
			},
		},
	})
```

Hooray!

Now you can explore all the different methods on the Dapr client.

## Creating a gRPC app with Dapr

The following steps will show you how to create an app that exposes a server for Dapr to communicate with.

1. Import the package

```go
package main

import (
	pb "github.com/dapr/go-sdk/daprclient"
)
```

2. Implement the interface

```go
// server is our user app
type server struct {
}

// Sample method to invoke
func (s *server) MyMethod() string {
	return "Hi there!"
}

// This method gets invoked when a remote service has called the app through Dapr
// The payload carries a Method to identify the method, a set of metadata properties and an optional payload
func (s *server) OnInvoke(ctx context.Context, in *pb.InvokeEnvelope) (*any.Any, error) {
	var response string
	switch in.Method {
	case "MyMethod":
		response = s.MyMethod()
	}
	return &any.Any{
		Value: []byte(response),
	}, nil
}

// Dapr will call this method to get the list of topics the app wants to subscribe to. In this example, we are telling Dapr
// To subscribe to a topic named TopicA
func (s *server) GetTopicSubscriptions(ctx context.Context, in *empty.Empty) (*pb.GetTopicSubscriptionsEnvelope, error) {
	return &pb.GetTopicSubscriptionsEnvelope{
		Topics: []string{"TopicA"},
	}, nil
}

// Dapper will call this method to get the list of bindings the app will get invoked by. In this example, we are telling Dapr
// To invoke our app with a binding named storage
func (s *server) GetBindingsSubscriptions(ctx context.Context, in *empty.Empty) (*pb.GetBindingsSubscriptionsEnvelope, error) {
	return &pb.GetBindingsSubscriptionsEnvelope{
		Bindings: []string{"storage"},
	}, nil
}

// This method gets invoked every time a new event is fired from a registerd binding. The message carries the binding name, a payload and optional metadata
func (s *server) OnBindingEvent(ctx context.Context, in *pb.BindingEventEnvelope) (*pb.BindingResponseEnvelope, error) {
	fmt.Println("Invoked from binding")
	return &pb.BindingResponseEnvelope{}, nil
}

// This method is fired whenever a message has been published to a topic that has been subscribed. Dapr sends published messages in a CloudEvents 0.3 envelope.
func (s *server) OnTopicEvent(ctx context.Context, in *pb.CloudEventEnvelope) (*empty.Empty, error) {
	fmt.Println("Topic message arrived")
	return &empty.Empty{}, nil
}

```

3. Create the server

```go
func main() {
	// create listiner
	lis, err := net.Listen("tcp", ":4000")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	// create grpc server
	s := grpc.NewServer()
	pb.RegisterDaprClientServer(s, &server{})

	fmt.Println("Client starting...")

	// and start...
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

This creates a gRPC server for your app on port 4000.

4. Run your app

To run locally, use the Dapr CLI:

```
dapr run --app-id goapp --app-port 4000 --protocol grpc go run main.go
```

On Kubernetes, set the required `dapr.io/protocol: "grpc"` and `dapr.io/port: "4000` annotations in your pod spec template as mentioned above.

## Other languages

You can use Dapr with any language supported by Protobuf, and not just with the currently available generated SDKs.
Using the [protoc](https://developers.google.com/protocol-buffers/docs/downloads) tool you can generate the Dapr clients for other languages like Ruby, C++, Rust and others.
