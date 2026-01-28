# Go Kafka - Complete Guide

## Table of Contents
- [Setup & Installation](#setup--installation)
- [Kafka Basics](#kafka-basics)
- [Using kafka-go (Recommended for Beginners)](#using-kafka-go-recommended-for-beginners)
- [Using confluent-kafka-go (Production)](#using-confluent-kafka-go-production)
- [Real-World Example - Order Processing System](#real-world-example---order-processing-system)
- [Advanced Patterns](#advanced-patterns)
- [Best Practices](#best-practices)

---

## Setup & Installation

### 1. Install Kafka Client Libraries

```bash
# Confluent's Kafka Go Client (recommended, based on librdkafka)
go get github.com/confluentinc/confluent-kafka-go/v2/kafka

# OR Shopify's Sarama (pure Go)
go get github.com/IBM/sarama

# OR Segment's kafka-go (pure Go, simpler API)
go get github.com/segmentio/kafka-go
```

**Comparison:**
- **confluent-kafka-go**: Most feature-rich, C bindings (librdkafka), best performance
- **sarama**: Pure Go, widely used, complex API
- **kafka-go**: Pure Go, simple API, good for beginners

We'll use **kafka-go** (simplest) and **confluent-kafka-go** (production-grade).

---

## Kafka Basics

### Key Concepts

```
Producer → Topic (Partitions) → Consumer Group → Consumers
```

- **Producer**: Sends messages to topics
- **Topic**: Category/feed of messages
- **Partition**: Topic split for parallelism
- **Consumer**: Reads messages from topics
- **Consumer Group**: Multiple consumers sharing workload

---

## Using kafka-go (Recommended for Beginners)

### 1. Producer - Send Messages

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
    
    "github.com/segmentio/kafka-go"
)

func main() {
    // Create writer (producer)
    writer := kafka.NewWriter(kafka.WriterConfig{
        Brokers:  []string{"localhost:9092"},
        Topic:    "my-topic",
        Balancer: &kafka.LeastBytes{},
    })
    defer writer.Close()
    
    // Send messages
    ctx := context.Background()
    
    for i := 0; i < 10; i++ {
        message := kafka.Message{
            Key:   []byte(fmt.Sprintf("key-%d", i)),
            Value: []byte(fmt.Sprintf("Message %d - %s", i, time.Now())),
        }
        
        err := writer.WriteMessages(ctx, message)
        if err != nil {
            log.Fatal("Failed to write message:", err)
        }
        
        fmt.Printf("Sent: %s\n", message.Value)
        time.Sleep(500 * time.Millisecond)
    }
    
    fmt.Println("All messages sent!")
}
```

### 2. Consumer - Read Messages

```go
package main

import (
    "context"
    "fmt"
    "log"
    
    "github.com/segmentio/kafka-go"
)

func main() {
    // Create reader (consumer)
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:   []string{"localhost:9092"},
        Topic:     "my-topic",
        GroupID:   "my-consumer-group",
        Partition: 0,  // Or omit for auto-assignment
        MinBytes:  10e3, // 10KB
        MaxBytes:  10e6, // 10MB
    })
    defer reader.Close()
    
    ctx := context.Background()
    
    fmt.Println("Consumer started, waiting for messages...")
    
    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            log.Fatal("Error reading message:", err)
        }
        
        fmt.Printf("Received: Key=%s, Value=%s, Partition=%d, Offset=%d\n",
            string(msg.Key),
            string(msg.Value),
            msg.Partition,
            msg.Offset,
        )
    }
}
```

### 3. Batch Producer

```go
func batchProduce() {
    writer := kafka.NewWriter(kafka.WriterConfig{
        Brokers:  []string{"localhost:9092"},
        Topic:    "my-topic",
        Balancer: &kafka.LeastBytes{},
    })
    defer writer.Close()
    
    // Prepare batch
    messages := []kafka.Message{}
    for i := 0; i < 100; i++ {
        messages = append(messages, kafka.Message{
            Key:   []byte(fmt.Sprintf("key-%d", i)),
            Value: []byte(fmt.Sprintf("Batch message %d", i)),
        })
    }
    
    // Send batch
    ctx := context.Background()
    err := writer.WriteMessages(ctx, messages...)
    if err != nil {
        log.Fatal("Failed to write batch:", err)
    }
    
    fmt.Printf("Sent %d messages in batch\n", len(messages))
}
```

### 4. Consumer with Manual Commit

```go
func manualCommitConsumer() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:  []string{"localhost:9092"},
        Topic:    "my-topic",
        GroupID:  "my-group",
        MinBytes: 10e3,
        MaxBytes: 10e6,
    })
    defer reader.Close()
    
    ctx := context.Background()
    
    for {
        msg, err := reader.FetchMessage(ctx)  // Fetch without committing
        if err != nil {
            log.Fatal(err)
        }
        
        fmt.Printf("Processing: %s\n", msg.Value)
        
        // Process message...
        time.Sleep(100 * time.Millisecond)
        
        // Manually commit after processing
        err = reader.CommitMessages(ctx, msg)
        if err != nil {
            log.Printf("Failed to commit: %v", err)
        }
    }
}
```

---

## Using confluent-kafka-go (Production)

### 1. Producer

```go
package main

import (
    "fmt"
    "log"
    
    "github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {
    // Create producer
    producer, err := kafka.NewProducer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "acks":              "all",  // Wait for all replicas
    })
    if err != nil {
        log.Fatal(err)
    }
    defer producer.Close()
    
    // Delivery report handler
    go func() {
        for e := range producer.Events() {
            switch ev := e.(type) {
            case *kafka.Message:
                if ev.TopicPartition.Error != nil {
                    fmt.Printf("Failed to deliver: %v\n", ev.TopicPartition)
                } else {
                    fmt.Printf("Delivered to %v\n", ev.TopicPartition)
                }
            }
        }
    }()
    
    // Send messages
    topic := "my-topic"
    for i := 0; i < 10; i++ {
        value := fmt.Sprintf("Message %d", i)
        
        err := producer.Produce(&kafka.Message{
            TopicPartition: kafka.TopicPartition{
                Topic:     &topic,
                Partition: kafka.PartitionAny,
            },
            Key:   []byte(fmt.Sprintf("key-%d", i)),
            Value: []byte(value),
        }, nil)
        
        if err != nil {
            log.Printf("Failed to produce: %v\n", err)
        }
    }
    
    // Wait for all messages to be delivered
    producer.Flush(15 * 1000)
    
    fmt.Println("All messages sent!")
}
```

### 2. Consumer

```go
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    
    "github.com/confluentinc/confluent-kafka-go/v2/kafka"
)

func main() {
    // Create consumer
    consumer, err := kafka.NewConsumer(&kafka.ConfigMap{
        "bootstrap.servers": "localhost:9092",
        "group.id":          "my-consumer-group",
        "auto.offset.reset": "earliest",  // Start from beginning
    })
    if err != nil {
        log.Fatal(err)
    }
    defer consumer.Close()
    
    // Subscribe to topics
    err = consumer.SubscribeTopics([]string{"my-topic"}, nil)
    if err != nil {
        log.Fatal(err)
    }
    
    // Setup signal handler
    sigchan := make(chan os.Signal, 1)
    signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)
    
    fmt.Println("Consumer started...")
    
    run := true
    for run {
        select {
        case sig := <-sigchan:
            fmt.Printf("Caught signal %v: terminating\n", sig)
            run = false
            
        default:
            // Poll for messages
            ev := consumer.Poll(100)
            if ev == nil {
                continue
            }
            
            switch e := ev.(type) {
            case *kafka.Message:
                fmt.Printf("Received: Topic=%s, Partition=%d, Offset=%d, Key=%s, Value=%s\n",
                    *e.TopicPartition.Topic,
                    e.TopicPartition.Partition,
                    e.TopicPartition.Offset,
                    string(e.Key),
                    string(e.Value),
                )
                
            case kafka.Error:
                fmt.Printf("Error: %v\n", e)
                
            default:
                fmt.Printf("Ignored event: %v\n", e)
            }
        }
    }
}
```

---

## Real-World Example - Order Processing System

### Producer: Order Service

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"
    
    "github.com/segmentio/kafka-go"
)

type Order struct {
    ID        string    `json:"id"`
    UserID    string    `json:"user_id"`
    Product   string    `json:"product"`
    Quantity  int       `json:"quantity"`
    Total     float64   `json:"total"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}

type OrderProducer struct {
    writer *kafka.Writer
}

func NewOrderProducer(brokers []string, topic string) *OrderProducer {
    return &OrderProducer{
        writer: kafka.NewWriter(kafka.WriterConfig{
            Brokers:  brokers,
            Topic:    topic,
            Balancer: &kafka.Hash{},  // Same key → same partition
        }),
    }
}

func (p *OrderProducer) PublishOrder(ctx context.Context, order Order) error {
    orderJSON, err := json.Marshal(order)
    if err != nil {
        return err
    }
    
    message := kafka.Message{
        Key:   []byte(order.ID),
        Value: orderJSON,
        Headers: []kafka.Header{
            {Key: "event_type", Value: []byte("order_created")},
        },
    }
    
    err = p.writer.WriteMessages(ctx, message)
    if err != nil {
        return fmt.Errorf("failed to publish order: %w", err)
    }
    
    log.Printf("Published order: %s", order.ID)
    return nil
}

func (p *OrderProducer) Close() error {
    return p.writer.Close()
}

func main() {
    producer := NewOrderProducer(
        []string{"localhost:9092"},
        "orders",
    )
    defer producer.Close()
    
    ctx := context.Background()
    
    // Simulate creating orders
    for i := 1; i <= 5; i++ {
        order := Order{
            ID:        fmt.Sprintf("ORDER-%d", i),
            UserID:    fmt.Sprintf("USER-%d", i%3+1),
            Product:   "Laptop",
            Quantity:  i,
            Total:     float64(i) * 999.99,
            Status:    "pending",
            CreatedAt: time.Now(),
        }
        
        err := producer.PublishOrder(ctx, order)
        if err != nil {
            log.Printf("Error publishing order: %v", err)
        }
        
        time.Sleep(1 * time.Second)
    }
    
    fmt.Println("All orders published!")
}
```

### Consumer: Payment Service

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "time"
    
    "github.com/segmentio/kafka-go"
)

type PaymentService struct {
    reader  *kafka.Reader
    writer  *kafka.Writer
}

func NewPaymentService(brokers []string) *PaymentService {
    return &PaymentService{
        reader: kafka.NewReader(kafka.ReaderConfig{
            Brokers:  brokers,
            Topic:    "orders",
            GroupID:  "payment-service",
            MinBytes: 10e3,
            MaxBytes: 10e6,
        }),
        writer: kafka.NewWriter(kafka.WriterConfig{
            Brokers: brokers,
            Topic:   "payments",
        }),
    }
}

func (s *PaymentService) processOrder(order Order) error {
    // Simulate payment processing
    log.Printf("Processing payment for order %s: $%.2f", order.ID, order.Total)
    time.Sleep(500 * time.Millisecond)
    
    // Create payment event
    payment := map[string]interface{}{
        "order_id":    order.ID,
        "user_id":     order.UserID,
        "amount":      order.Total,
        "status":      "completed",
        "processed_at": time.Now(),
    }
    
    paymentJSON, _ := json.Marshal(payment)
    
    // Publish payment event
    err := s.writer.WriteMessages(context.Background(), kafka.Message{
        Key:   []byte(order.ID),
        Value: paymentJSON,
    })
    
    if err != nil {
        return fmt.Errorf("failed to publish payment: %w", err)
    }
    
    log.Printf("Payment completed for order %s", order.ID)
    return nil
}

func (s *PaymentService) Start(ctx context.Context) error {
    log.Println("Payment service started...")
    
    for {
        msg, err := s.reader.ReadMessage(ctx)
        if err != nil {
            return err
        }
        
        var order Order
        err = json.Unmarshal(msg.Value, &order)
        if err != nil {
            log.Printf("Failed to unmarshal order: %v", err)
            continue
        }
        
        err = s.processOrder(order)
        if err != nil {
            log.Printf("Failed to process order: %v", err)
        }
    }
}

func (s *PaymentService) Close() {
    s.reader.Close()
    s.writer.Close()
}

func main() {
    service := NewPaymentService([]string{"localhost:9092"})
    defer service.Close()
    
    ctx := context.Background()
    
    if err := service.Start(ctx); err != nil {
        log.Fatal(err)
    }
}
```

---

## Advanced Patterns

### 1. Error Handling & Retry

```go
func consumeWithRetry() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        Topic:   "my-topic",
        GroupID: "my-group",
    })
    defer reader.Close()
    
    ctx := context.Background()
    
    for {
        msg, err := reader.FetchMessage(ctx)
        if err != nil {
            log.Printf("Error fetching: %v", err)
            continue
        }
        
        // Try to process with retries
        maxRetries := 3
        for i := 0; i < maxRetries; i++ {
            err = processMessage(msg)
            if err == nil {
                break
            }
            
            log.Printf("Retry %d/%d: %v", i+1, maxRetries, err)
            time.Sleep(time.Second * time.Duration(i+1))
        }
        
        if err != nil {
            // Send to dead letter queue
            sendToDeadLetter(msg)
        }
        
        // Commit only after successful processing
        reader.CommitMessages(ctx, msg)
    }
}

func processMessage(msg kafka.Message) error {
    // Your processing logic
    return nil
}

func sendToDeadLetter(msg kafka.Message) {
    // Send failed message to DLQ
}
```

### 2. Graceful Shutdown

```go
func gracefulConsumer() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        Topic:   "my-topic",
        GroupID: "my-group",
    })
    defer reader.Close()
    
    ctx, cancel := context.WithCancel(context.Background())
    
    // Handle signals
    sigchan := make(chan os.Signal, 1)
    signal.Notify(sigchan, os.Interrupt, syscall.SIGTERM)
    
    go func() {
        <-sigchan
        log.Println("Shutting down gracefully...")
        cancel()
    }()
    
    for {
        select {
        case <-ctx.Done():
            log.Println("Consumer stopped")
            return
        default:
            msg, err := reader.ReadMessage(ctx)
            if err != nil {
                if ctx.Err() != nil {
                    return  // Context cancelled
                }
                log.Printf("Error: %v", err)
                continue
            }
            
            log.Printf("Received: %s", msg.Value)
        }
    }
}
```

---

## Summary

| Library | Pros | Cons |
|---------|------|------|
| kafka-go | Simple API, pure Go, easy to learn | Fewer features |
| confluent-kafka-go | Feature-rich, high performance | C dependency (librdkafka) |
| sarama | Pure Go, widely used | Complex API, verbose |

---

## Best Practices

### ✅ DO:

- Use consumer groups for scalability
- Commit after processing, not before
- Handle errors and retries
- Use dead letter queues for failed messages
- Implement graceful shutdown
- Monitor consumer lag
- Use idempotent producers (avoid duplicates)
- Set appropriate timeouts

### ❌ DON'T:

- Don't commit before processing
- Don't ignore errors
- Don't use same consumer group ID for different services
- Don't process messages synchronously if not needed

---

## Common Patterns

- Event sourcing
- CQRS (Command Query Responsibility Segregation)
- Microservices communication
- Real-time analytics
- Log aggregation

Go + Kafka = Scalable event-driven systems! 🚀📨
