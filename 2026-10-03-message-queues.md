# Day 12: Message Queues Fundamentals

## What problem does it solve?

A user places an order. After saving it, your service must also: send a confirmation email, update inventory, notify the warehouse, and record analytics. If the order service calls each of these **directly and waits** for each one:

- The user waits for all of them before seeing "Order placed."
- If the email service is down, the whole order request fails, even though the order itself was fine.
- If 10,000 orders arrive in one minute, every downstream service gets hit with 10,000 calls at once.

A **message queue** sits in the middle. The order service drops a message ("order 42 was placed") into the queue and **immediately moves on**. Other services pick up messages from the queue whenever they're ready.

```
Order service ──► [ queue: msg msg msg ] ──► Email service
   (producer)                             ──► Inventory service
                                              (consumers)
```

Yesterday's DSA read was queues (FIFO). A message queue is the same idea, scaled up into a separate, durable system shared between services.

## Key vocabulary

- **Producer:** sends messages.
- **Consumer:** receives and processes messages.
- **Broker:** the queue system itself (RabbitMQ, Kafka, Amazon SQS).
- **Acknowledgment (ack):** the consumer tells the broker "I finished processing this message, you can delete it."

## What you gain

**1. Decoupling.** The order service doesn't know or care which services consume its message. Adding a new "loyalty points" consumer requires no changes to the order service.

**2. Asynchronous processing.** The user gets a fast response. Slow work, like sending emails or generating PDFs, happens in the background.

**3. Resilience.** If the email service is down for 10 minutes, messages wait safely in the queue. When it comes back, it catches up. Nothing is lost, and orders kept working the whole time.

**4. Load leveling.** A burst of 10,000 orders becomes 10,000 queued messages. Consumers work through them at their own steady pace, instead of being overwhelmed all at once.

## The catch: messages can arrive more than once

Most brokers guarantee **at-least-once delivery**. Here's why duplicates happen:

1. A consumer processes a message (sends the email).
2. It crashes **before** sending the acknowledgment.
3. The broker never got the ack, so it assumes the message wasn't handled, and redelivers it.
4. The email is sent twice.

So consumers must be **idempotent**: processing the same message twice must have the same effect as processing it once. A common technique is to give every message a unique ID, store processed IDs, and skip any you've already seen. This is the same idempotency idea as `PUT` and `DELETE` from Day 2.

**Ordering** isn't guaranteed by default either, especially once several consumers process messages in parallel. If order matters (such as "created" before "cancelled" for the same order), you need a broker feature for it, like Kafka's per-partition ordering.

## Poison messages and dead-letter queues

What if a message can never be processed, such as malformed data that crashes the consumer every time? Without a limit, the broker redelivers it forever, blocking or wasting work. The fix is a **dead-letter queue (DLQ)**: after N failed attempts, the message moves to a separate queue, where someone can investigate it.

## Two common styles

| | Traditional queue (e.g. RabbitMQ, SQS) | Event log (e.g. Kafka) |
|---|---|---|
| After a message is consumed | Removed | Kept for a retention period |
| Typical delivery | Each message goes to one consumer | Many independent consumer groups each read the full stream |
| Replay old messages | No | Yes, by re-reading from an earlier position |
| Good for | Task distribution, background jobs | Event streaming, multiple systems reacting to the same events |

## Java angle: Spring

With Spring Kafka:

```java
@Service
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderPlacedEvent> kafka;

    public OrderEventPublisher(KafkaTemplate<String, OrderPlacedEvent> kafka) {
        this.kafka = kafka;
    }

    public void publish(OrderPlacedEvent event) {
        kafka.send("orders", event.orderId(), event);   // returns immediately
    }
}

@Service
public class EmailConsumer {
    @KafkaListener(topics = "orders", groupId = "email-service")
    public void handle(OrderPlacedEvent event) {
        if (alreadyProcessed(event.eventId())) return;   // idempotency check
        sendConfirmationEmail(event);
        markProcessed(event.eventId());
    }
}
```

Using `orderId` as the message key sends all events for the same order to the same Kafka partition, so they stay in order. Spring AMQP offers the same pattern for RabbitMQ, with `@RabbitListener`.

## Common pitfalls

- **Non-idempotent consumers.** Duplicate deliveries cause double charges or duplicate emails.
- **Assuming messages arrive in order.** They often don't, unless you design for it.
- **No dead-letter queue.** One bad message gets retried forever.
- **The dual-write problem.** Saving the order to the database and publishing the message are two separate operations. If one succeeds and the other fails, they disagree. (The usual fix, the "outbox pattern," is a later topic.)
- **Queuing work that needs an immediate answer.** If the user must see the result right now, like "is this card valid?", a synchronous call is the right tool.

## Interviewer follow-ups

**"Why use a message queue instead of calling the service directly?"** Decoupling, faster user responses, resilience when a downstream service is down, and smoothing out traffic spikes. The trade-off is eventual consistency and extra moving parts.

**"How do you handle duplicate messages?"** Make consumers idempotent. Track processed message IDs, or design operations so applying them twice is harmless.

## Retention Check

1. What three problems arise when one service calls all its downstream services synchronously?
2. Define producer, consumer, and broker.
3. What does it mean for a consumer to acknowledge a message?
4. Why does at-least-once delivery produce duplicates?
5. How do you make a consumer idempotent?
6. What is a dead-letter queue for?
7. How does Kafka differ from a traditional queue after a message is consumed?
8. What is the dual-write problem?

**Key points to self-check against:**
- The user waits for everything; one failing service fails the whole request; traffic spikes hit every downstream service at once.
- Producer sends, consumer receives and processes, broker is the queue system in the middle.
- It tells the broker the message was fully processed and can be deleted.
- A consumer may finish the work but crash before acknowledging, so the broker redelivers it.
- Give each message a unique ID, record processed IDs, and skip any you've already handled.
- Holding messages that keep failing, so they don't retry forever.
- Kafka keeps messages for a retention period and allows replay; a traditional queue removes them once consumed.
- Writing to the database and publishing a message are separate steps that can partially fail and disagree.
