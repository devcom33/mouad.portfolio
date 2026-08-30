---
title: "How I Used RabbitMQ with Spring Boot to Handle Long-Running AI Tasks"
description: "How I used RabbitMQ, Redis, Spring Boot, and a Python worker to process long-running AI tasks asynchronously."
pubDatetime: 2025-10-09T18:00:00Z
author: "Mouad"
tags:
  - spring-boot
  - rabbitmq
  - redis
  - python
  - ai
  - backend
draft: false
---

# How I Used RabbitMQ with Spring Boot to Handle Long-Running AI Tasks

Recently, I was working on an AI application for reviewing employment contracts. My teammate handled the AI engineering side by building a REST API service, while I focused on the software engineering side using Java, Spring Boot, and React.

Once we needed to connect the two services, my first approach was straightforward: send the request directly from the Spring Boot backend to the AI service using HTTP.

It worked. I got a `200 OK` response and everything looked fine.

But there was a problem.

The AI processing could take a significant amount of time. Keeping an HTTP request open while waiting for the AI service wasn't a good fit for this kind of workload. If the request eventually moved through the frontend, timeouts could become another problem.

I needed a way to decouple the request from the actual processing.

That's where **RabbitMQ** came in.

## Why RabbitMQ?

RabbitMQ is a message broker that allows different services to communicate asynchronously.

A simple way to think about it is a post office:

- A **producer** sends a message.
- An **exchange** decides where the message should go.
- A **queue** stores the message until it can be processed.
- A **consumer** receives and processes the message.

Instead of making my Spring Boot application wait for the AI service to finish, I could send a message to RabbitMQ and let a separate worker process it.

The architecture became:

```text
React
  │
  ▼
Spring Boot
  │
  │ publish message
  ▼
RabbitMQ
  │
  │ consume message
  ▼
Python AI Worker
  │
  │ store result
  ▼
Redis
  │
  │ publish result
  ▼
Spring Boot
  │
  ▼
React
```

This gave me a much cleaner separation between the application handling user requests and the service performing the heavy AI processing.

## My Setup

In my application, each component had a specific responsibility:

- **Spring Boot** — receives the request and acts as the RabbitMQ producer.
- **RabbitMQ** — stores and delivers contract-processing messages.
- **Python worker** — consumes messages and performs the AI processing.
- **Redis** — stores the processing result and publishes an event when the result is ready.
- **React** — displays the result to the user.

The important part is that the Spring Boot application doesn't need to wait for the AI processing to finish before responding.

## Step 1: Add RabbitMQ to Spring Boot

First, I added Spring AMQP to my Maven project:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

This gives Spring Boot the integration needed to communicate with RabbitMQ.

## Step 2: Configure RabbitMQ

I created a queue, exchange, binding, JSON message converter, and `RabbitTemplate`:

```java
@Configuration
public class RabbitConfig {

    @Bean
    public Queue contractQueue() {
        return new Queue("contract-queue", true);
    }

    @Bean
    public DirectExchange contractExchange() {
        return new DirectExchange("contract-exchange");
    }

    @Bean
    public Binding binding(
            Queue contractQueue,
            DirectExchange contractExchange
    ) {
        return BindingBuilder
                .bind(contractQueue)
                .to(contractExchange)
                .with("contract-routing-key");
    }

    @Bean
    public Jackson2JsonMessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public RabbitTemplate rabbitTemplate(
            ConnectionFactory connectionFactory
    ) {
        RabbitTemplate rabbitTemplate =
                new RabbitTemplate(connectionFactory);

        rabbitTemplate.setMessageConverter(jsonMessageConverter());

        return rabbitTemplate;
    }
}
```

There are a few important pieces here.

### Queue

```java
new Queue("contract-queue", true);
```

The queue is durable, meaning RabbitMQ can preserve it across broker restarts.

### Exchange

```java
new DirectExchange("contract-exchange");
```

I used a `DirectExchange`, which routes messages based on an exact routing key.

### Binding

```java
.with("contract-routing-key");
```

This connects the exchange to the queue.

When a message is published to `contract-exchange` with the routing key `contract-routing-key`, RabbitMQ routes it to `contract-queue`.

### JSON message converter

The `Jackson2JsonMessageConverter` allows Java objects to be converted to JSON when sending messages.

## Step 3: Send the Message from Spring Boot

Next, I created a producer responsible for sending contract-processing requests to RabbitMQ:

```java
@Service
@RequiredArgsConstructor
public class ContractReviewProducer {

    private final RabbitTemplate rabbitTemplate;

    public String sendToQueue(ContractResponse contract) {

        String id = UUID.randomUUID().toString();

        ContractMessage message = new ContractMessage(
                id,
                contract.filename(),
                contract.extractedText(),
                contract.header()
        );

        rabbitTemplate.convertAndSend(
                "contract-exchange",
                "contract-routing-key",
                message
        );

        return id;
    }
}
```

The UUID is important here.

Instead of simply sending the contract and forgetting about it, I generate a unique identifier for each processing request.

That identifier can later be used to associate the AI result with the original request.

The important line is:

```java
rabbitTemplate.convertAndSend(
        "contract-exchange",
        "contract-routing-key",
        message
);
```

Spring sends the message to the exchange, and RabbitMQ takes care of routing it to the appropriate queue.

At this point, the Spring Boot application doesn't need to perform the AI processing itself.

## Step 4: Process the Message with a Python Worker

The AI processing was handled by a separate Python worker.

For RabbitMQ and Redis, I used:

```bash
pip install pika redis
```

The worker listens for messages from the RabbitMQ queue:

```python
import pika
import redis
import json

r = redis.Redis(
    host="localhost",
    port=6379,
    db=0
)


def process_message(ch, method, properties, body):

    message = json.loads(body)

    print("Processing a new contract message...")

    # Perform AI processing here
    result = "AI processing result"

    r.set(
        "contract_id",
        json.dumps(result)
    )

    r.publish(
        "contract_results",
        json.dumps({
            "id": "contract_id",
            "result": result
        })
    )

    ch.basic_ack(
        delivery_tag=method.delivery_tag
    )


def start_worker():

    connection = pika.BlockingConnection(
        pika.ConnectionParameters("localhost")
    )

    channel = connection.channel()

    channel.queue_declare(
        queue="contract-queue",
        durable=True
    )

    channel.basic_qos(
        prefetch_count=1
    )

    channel.basic_consume(
        queue="contract-queue",
        on_message_callback=process_message
    )

    print("Worker started. Waiting for messages...")

    channel.start_consuming()


if __name__ == "__main__":
    start_worker()
```

The worker does three main things:

1. Receives a message from RabbitMQ.
2. Performs the AI processing.
3. Stores and publishes the result through Redis.

### Why `basic_ack` matters

After successfully processing the message, the worker acknowledges it:

```python
ch.basic_ack(
    delivery_tag=method.delivery_tag
)
```

This tells RabbitMQ that the message has been successfully handled.

I also used:

```python
channel.basic_qos(
    prefetch_count=1
)
```

This limits the worker to receiving one unacknowledged message at a time, which is useful when each task can be computationally expensive.

## Why Redis?

RabbitMQ handles the task queue, but I also needed a way to make the completed result available to the rest of the application.

That's where Redis came in.

The worker stores the result:

```python
r.set(
    "contract_id",
    json.dumps(result)
)
```

and publishes an event:

```python
r.publish(
    "contract_results",
    json.dumps({
        "id": "contract_id",
        "result": result
    })
)
```

The Spring Boot application can listen for these events and then make the result available to the frontend.

This separates two different responsibilities:

**RabbitMQ → task delivery**

**Redis → result storage and notification**

## What I Learned

This project taught me an important lesson about distributed systems:

> Not every operation belongs inside a synchronous HTTP request.

For short operations, a normal REST call is often perfectly fine.

But when a task involves potentially long-running work such as AI inference, document processing, video processing, or other computationally expensive operations, asynchronous processing can be a much better architecture.

In my case, RabbitMQ allowed me to decouple the Spring Boot application from the AI worker.

The final flow became:

```text
User
 │
 ▼
React
 │
 ▼
Spring Boot
 │
 │ 1. Create task
 │
 ▼
RabbitMQ
 │
 │ 2. Deliver task
 │
 ▼
Python AI Worker
 │
 │ 3. Process
 │
 ▼
Redis
 │
 │ 4. Publish result
 │
 ▼
Spring Boot
 │
 │ 5. Notify frontend
 │
 ▼
React
```

This architecture made the system more suitable for long-running AI workloads and gave each service a clear responsibility.

If you're building a system where one service needs to perform work that can take seconds or minutes, I'd definitely consider asynchronous messaging instead of keeping the original HTTP request open.

You can find more of my projects on [GitHub](https://github.com/devcom33).
