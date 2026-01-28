# Go gRPC - Complete Guide

## Table of Contents
- [Setup & Installation](#setup--installation)
- [Protocol Buffers Basics](#protocol-buffers-basics)
- [Simple gRPC Server](#simple-grpc-server)
- [gRPC Client](#grpc-client)
- [Streaming Examples](#streaming-examples)
- [Middleware (Interceptors)](#middleware-interceptors)
- [Error Handling](#error-handling)
- [TLS/SSL Security](#tlsssl-security)
- [Complete Real-World Example](#complete-real-world-example)
- [Best Practices](#best-practices)

---

## Setup & Installation

### 1. Install Tools

```bash
# Install protoc (Protocol Buffer compiler)
# macOS
brew install protobuf

# Linux
apt-get install -y protobuf-compiler

# Verify
protoc --version

# Install Go plugins for protoc
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Add to PATH
export PATH="$PATH:$(go env GOPATH)/bin"

# Install gRPC library
go get google.golang.org/grpc
go get google.golang.org/protobuf
```

---

## Protocol Buffers Basics

### 1. Create .proto File

**user.proto:**

```protobuf
syntax = "proto3";

package user;

option go_package = "github.com/yourname/yourproject/proto/user";

// User message
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
}

// Request messages
message GetUserRequest {
  int32 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

// Response messages
message GetUserResponse {
  User user = 1;
}

message CreateUserResponse {
  User user = 1;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}

// Service definition
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  
  // Streaming examples
  rpc StreamUsers(ListUsersRequest) returns (stream User);
  rpc RecordUsers(stream CreateUserRequest) returns (CreateUserResponse);
  rpc ChatUsers(stream User) returns (stream User);
}
```

### 2. Generate Go Code

```bash
# Create directory structure
mkdir -p proto/user

# Place user.proto in proto/user/

# Generate code
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/user/user.proto

# Generated files:
# - user.pb.go (message definitions)
# - user_grpc.pb.go (service definitions)
```

---

## Simple gRPC Server

### Implement Server

**server/main.go:**

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "net"
    "sync"
    
    pb "yourproject/proto/user"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type userServer struct {
    pb.UnimplementedUserServiceServer
    mu      sync.RWMutex
    users   map[int32]*pb.User
    nextID  int32
}

func newServer() *userServer {
    return &userServer{
        users:  make(map[int32]*pb.User),
        nextID: 1,
    }
}

// GetUser - Unary RPC
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    log.Printf("GetUser called with ID: %d", req.Id)
    
    s.mu.RLock()
    user, exists := s.users[req.Id]
    s.mu.RUnlock()
    
    if !exists {
        return nil, status.Errorf(codes.NotFound, "user not found: %d", req.Id)
    }
    
    return &pb.GetUserResponse{User: user}, nil
}

// CreateUser - Unary RPC
func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {
    log.Printf("CreateUser called: %s", req.Name)
    
    s.mu.Lock()
    defer s.mu.Unlock()
    
    user := &pb.User{
        Id:    s.nextID,
        Name:  req.Name,
        Email: req.Email,
        Age:   req.Age,
    }
    
    s.users[s.nextID] = user
    s.nextID++
    
    return &pb.CreateUserResponse{User: user}, nil
}

// ListUsers - Unary RPC
func (s *userServer) ListUsers(ctx context.Context, req *pb.ListUsersRequest) (*pb.ListUsersResponse, error) {
    log.Printf("ListUsers called")
    
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    users := make([]*pb.User, 0, len(s.users))
    for _, user := range s.users {
        users = append(users, user)
    }
    
    return &pb.ListUsersResponse{
        Users: users,
        Total: int32(len(users)),
    }, nil
}

// StreamUsers - Server streaming RPC
func (s *userServer) StreamUsers(req *pb.ListUsersRequest, stream pb.UserService_StreamUsersServer) error {
    log.Printf("StreamUsers called")
    
    s.mu.RLock()
    users := make([]*pb.User, 0, len(s.users))
    for _, user := range s.users {
        users = append(users, user)
    }
    s.mu.RUnlock()
    
    for _, user := range users {
        if err := stream.Send(user); err != nil {
            return err
        }
    }
    
    return nil
}

// RecordUsers - Client streaming RPC
func (s *userServer) RecordUsers(stream pb.UserService_RecordUsersServer) error {
    log.Printf("RecordUsers called")
    
    var count int32
    
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // Client finished sending
            return stream.SendAndClose(&pb.CreateUserResponse{
                User: &pb.User{
                    Id:   count,
                    Name: fmt.Sprintf("Recorded %d users", count),
                },
            })
        }
        if err != nil {
            return err
        }
        
        s.mu.Lock()
        user := &pb.User{
            Id:    s.nextID,
            Name:  req.Name,
            Email: req.Email,
            Age:   req.Age,
        }
        s.users[s.nextID] = user
        s.nextID++
        s.mu.Unlock()
        
        count++
    }
}

// ChatUsers - Bidirectional streaming RPC
func (s *userServer) ChatUsers(stream pb.UserService_ChatUsersServer) error {
    log.Printf("ChatUsers called")
    
    for {
        user, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        
        log.Printf("Received user: %s", user.Name)
        
        // Echo back
        response := &pb.User{
            Id:    user.Id,
            Name:  fmt.Sprintf("Echo: %s", user.Name),
            Email: user.Email,
            Age:   user.Age,
        }
        
        if err := stream.Send(response); err != nil {
            return err
        }
    }
}

func main() {
    // Create listener
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("Failed to listen: %v", err)
    }
    
    // Create gRPC server
    grpcServer := grpc.NewServer()
    
    // Register service
    pb.RegisterUserServiceServer(grpcServer, newServer())
    
    log.Println("gRPC server listening on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("Failed to serve: %v", err)
    }
}
```

---

## gRPC Client

### Simple Client

**client/main.go:**

```go
package main

import (
    "context"
    "io"
    "log"
    "time"
    
    pb "yourproject/proto/user"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    // Connect to server
    conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()
    
    // Create client
    client := pb.NewUserServiceClient(conn)
    
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    // Test CreateUser
    createUser(ctx, client)
    
    // Test GetUser
    getUser(ctx, client, 1)
    
    // Test ListUsers
    listUsers(ctx, client)
    
    // Test StreamUsers
    streamUsers(ctx, client)
}

func createUser(ctx context.Context, client pb.UserServiceClient) {
    req := &pb.CreateUserRequest{
        Name:  "Alice",
        Email: "alice@example.com",
        Age:   30,
    }
    
    resp, err := client.CreateUser(ctx, req)
    if err != nil {
        log.Fatalf("CreateUser failed: %v", err)
    }
    
    log.Printf("Created user: ID=%d, Name=%s", resp.User.Id, resp.User.Name)
}

func getUser(ctx context.Context, client pb.UserServiceClient, id int32) {
    req := &pb.GetUserRequest{Id: id}
    
    resp, err := client.GetUser(ctx, req)
    if err != nil {
        log.Fatalf("GetUser failed: %v", err)
    }
    
    log.Printf("Got user: %+v", resp.User)
}

func listUsers(ctx context.Context, client pb.UserServiceClient) {
    req := &pb.ListUsersRequest{
        Page:     1,
        PageSize: 10,
    }
    
    resp, err := client.ListUsers(ctx, req)
    if err != nil {
        log.Fatalf("ListUsers failed: %v", err)
    }
    
    log.Printf("Total users: %d", resp.Total)
    for _, user := range resp.Users {
        log.Printf("  - %s (%s)", user.Name, user.Email)
    }
}

func streamUsers(ctx context.Context, client pb.UserServiceClient) {
    req := &pb.ListUsersRequest{
        Page:     1,
        PageSize: 10,
    }
    
    stream, err := client.StreamUsers(ctx, req)
    if err != nil {
        log.Fatalf("StreamUsers failed: %v", err)
    }
    
    log.Println("Streaming users:")
    for {
        user, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatalf("Stream error: %v", err)
        }
        
        log.Printf("  - %s", user.Name)
    }
}
```

---

## Streaming Examples

### 1. Client Streaming

**Client:**

```go
func recordUsers(ctx context.Context, client pb.UserServiceClient) {
    stream, err := client.RecordUsers(ctx)
    if err != nil {
        log.Fatalf("RecordUsers failed: %v", err)
    }
    
    users := []struct {
        name  string
        email string
        age   int32
    }{
        {"Alice", "alice@example.com", 30},
        {"Bob", "bob@example.com", 25},
        {"Charlie", "charlie@example.com", 35},
    }
    
    for _, u := range users {
        req := &pb.CreateUserRequest{
            Name:  u.name,
            Email: u.email,
            Age:   u.age,
        }
        
        if err := stream.Send(req); err != nil {
            log.Fatalf("Send failed: %v", err)
        }
        
        log.Printf("Sent user: %s", u.name)
        time.Sleep(500 * time.Millisecond)
    }
    
    resp, err := stream.CloseAndRecv()
    if err != nil {
        log.Fatalf("CloseAndRecv failed: %v", err)
    }
    
    log.Printf("Recorded response: %s", resp.User.Name)
}
```

### 2. Bidirectional Streaming

**Client:**

```go
func chatUsers(ctx context.Context, client pb.UserServiceClient) {
    stream, err := client.ChatUsers(ctx)
    if err != nil {
        log.Fatalf("ChatUsers failed: %v", err)
    }
    
    // Send and receive concurrently
    waitc := make(chan struct{})
    
    // Goroutine to receive
    go func() {
        for {
            user, err := stream.Recv()
            if err == io.EOF {
                close(waitc)
                return
            }
            if err != nil {
                log.Fatalf("Receive error: %v", err)
            }
            
            log.Printf("Received: %s", user.Name)
        }
    }()
    
    // Send messages
    users := []*pb.User{
        {Name: "Alice", Email: "alice@example.com", Age: 30},
        {Name: "Bob", Email: "bob@example.com", Age: 25},
        {Name: "Charlie", Email: "charlie@example.com", Age: 35},
    }
    
    for _, user := range users {
        if err := stream.Send(user); err != nil {
            log.Fatalf("Send error: %v", err)
        }
        log.Printf("Sent: %s", user.Name)
        time.Sleep(500 * time.Millisecond)
    }
    
    stream.CloseSend()
    <-waitc
}
```

---

## Middleware (Interceptors)

### 1. Server Interceptor (Logging)

```go
import (
    "google.golang.org/grpc"
)

func loggingInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    
    log.Printf("Method: %s, Request: %+v", info.FullMethod, req)
    
    // Call handler
    resp, err := handler(ctx, req)
    
    log.Printf("Method: %s, Duration: %v, Error: %v",
        info.FullMethod, time.Since(start), err)
    
    return resp, err
}

func main() {
    grpcServer := grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor),
    )
    
    // ... rest of server setup
}
```

### 2. Authentication Interceptor

```go
func authInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Get metadata from context
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }
    
    // Check authorization token
    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing token")
    }
    
    token := tokens[0]
    if !validateToken(token) {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }
    
    return handler(ctx, req)
}

func validateToken(token string) bool {
    // Implement your token validation
    return token == "valid-token"
}
```

### 3. Client Interceptor

```go
func clientLoggingInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    start := time.Now()
    
    log.Printf("Calling %s with request: %+v", method, req)
    
    err := invoker(ctx, method, req, reply, cc, opts...)
    
    log.Printf("Completed %s in %v, error: %v", method, time.Since(start), err)
    
    return err
}

func main() {
    conn, err := grpc.Dial(
        "localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithUnaryInterceptor(clientLoggingInterceptor),
    )
    // ... rest of client setup
}
```

---

## Error Handling

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    // Validation error
    if req.Id <= 0 {
        return nil, status.Error(codes.InvalidArgument, "invalid user ID")
    }
    
    s.mu.RLock()
    user, exists := s.users[req.Id]
    s.mu.RUnlock()
    
    // Not found error
    if !exists {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
    }
    
    // Internal error
    if user.Email == "" {
        return nil, status.Error(codes.Internal, "user data corrupted")
    }
    
    return &pb.GetUserResponse{User: user}, nil
}

// Client error handling
func handleGetUser(client pb.UserServiceClient, id int32) {
    resp, err := client.GetUser(context.Background(), &pb.GetUserRequest{Id: id})
    if err != nil {
        st, ok := status.FromError(err)
        if ok {
            log.Printf("Error code: %s, Message: %s", st.Code(), st.Message())
            
            switch st.Code() {
            case codes.NotFound:
                log.Println("User not found")
            case codes.InvalidArgument:
                log.Println("Invalid argument")
            case codes.Internal:
                log.Println("Internal server error")
            }
        }
        return
    }
    
    log.Printf("User: %+v", resp.User)
}
```

---

## TLS/SSL Security

### Server with TLS

```go
import (
    "google.golang.org/grpc/credentials"
)

func main() {
    // Load TLS credentials
    creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
    if err != nil {
        log.Fatalf("Failed to load TLS: %v", err)
    }
    
    grpcServer := grpc.NewServer(grpc.Creds(creds))
    
    // ... rest of setup
}
```

### Client with TLS

```go
func main() {
    creds, err := credentials.NewClientTLSFromFile("ca.crt", "")
    if err != nil {
        log.Fatalf("Failed to load TLS: %v", err)
    }
    
    conn, err := grpc.Dial(
        "localhost:50051",
        grpc.WithTransportCredentials(creds),
    )
    
    // ... rest of setup
}
```

---

## Complete Real-World Example

### Project Structure

```
myproject/
├── proto/
│   └── user/
│       ├── user.proto
│       ├── user.pb.go
│       └── user_grpc.pb.go
├── server/
│   └── main.go
├── client/
│   └── main.go
└── go.mod
```

---

## Summary

### Key Points

- ✅ Protocol Buffers define services & messages
- ✅ Four types: Unary, Server Streaming, Client Streaming, Bidirectional
- ✅ Use interceptors for cross-cutting concerns
- ✅ Handle errors with status codes
- ✅ Use TLS for production
- ✅ Context for timeouts & cancellation

### RPC Types

| Type | Description | Use Case |
|------|-------------|----------|
| Unary | Request → Response | Simple CRUD operations |
| Server Streaming | Request → Stream of Responses | Large data sets, real-time updates |
| Client Streaming | Stream of Requests → Response | Batch uploads, logging |
| Bidirectional Streaming | Stream ↔ Stream | Chat, real-time collaboration |

gRPC is perfect for microservices! 🚀
