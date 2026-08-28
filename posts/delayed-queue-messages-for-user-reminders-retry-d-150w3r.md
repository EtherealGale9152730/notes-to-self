# Delayed Queue Messages for User Reminders — Retry, Dead Letters, and the Seven-Day Limit

A customer-support reminder has a very different shape from a web request: “send this seven days from now” should not occupy a process, a connection, or a polling loop today. **Short answer: use one delayed queue message for reminders within seven days, and let a small scheduler promote later reminders into that near-term queue.** Keep the payload small, make delivery idempotent, and route exhausted retries to a dead-letter queue (DLQ).

That decision is about latency versus cost. Polling a reminders table every minute can reduce scheduling surprises, but it burns database reads and still leaves you tuning a polling interval. A delayed message is cheaper operationally for a short horizon because the broker holds the wake-up. It is not a durable calendar, though.

## How can delayed queue messages handle user reminders?

Start with two records: a reminder row in your database and a queue message containing only its identifier, target channel, and due timestamp. The database remains the source of truth for the full text, customer locale, and cancellation state. A message body approaching 256KB is a design smell; fetching the body by ID also makes edits and redaction straightforward.

For a reminder due in the next seven days, publish once with a delay. The worker consumes it, re-reads the row, checks that it is still pending, and sends through the notification provider. Standard queues are at-least-once, so the worker must tolerate the same ID arriving twice. A unique delivery key such as `reminder_id:attempt` (or a durable sent marker keyed by `reminder_id`) prevents duplicate SMS or email. That check belongs next to the provider call, not in a dashboard rule: a worker can crash after the provider accepts a message but before the queue acknowledgement reaches the broker, and the subsequent delivery is then perfectly normal. I keep the state transition and the provider token in one database transaction where possible; where the provider cannot participate, an outbox record plus a deterministic idempotency key gives the same practical property without pretending the two systems share a transaction.

Keep it boring.

For a date farther out, store it in the database and have a cron promoter periodically publish it when it enters the seven-day window. This is a promotion loop, not a workflow engine: it has no DAG or join primitive, and a paused cron does not replay missed triggers. The cron call should only enqueue work; a worker handles the send, especially when the operation can exceed the 900-second cron execution limit.

Here is a deliberately small Python sketch using the queue surface. It shows explicit methods, bearer authentication, a client idempotency key, status checks, and bounded exponential backoff for rate limits. The exact request fields should be confirmed against the live capability schema before wiring production code.

```python
import os
import time
import requests

BASE = os.environ["QUEUE_API_ORIGIN"].rstrip("/")
KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

def publish_reminder(queue, reminder_id, delay_seconds):
    body = {"messages": [{"body": {"reminder_id": reminder_id},
                           "delay_seconds": delay_seconds,
                           "idempotency_key": f"reminder:{reminder_id}"}]}
    for attempt in range(5):
        response = requests.request("POST", f"{BASE}/v1/queue/publish",
                                     headers=HEADERS, json={"queue": queue, **body}, timeout=10)
        if response.status_code == 429:
            wait = int(response.headers.get("Retry-After", 2 ** attempt))
            time.sleep(wait)
            continue
        response.raise_for_status()
        return response.json()
    raise RuntimeError("publish retry budget exhausted")

def consume_once(queue):
    response = requests.request("POST", f"{BASE}/v1/queue/consume",
                                headers=HEADERS, json={"queue": queue, "max_messages": 1}, timeout=10)
    response.raise_for_status()
    return response.json()
```

The consumer should acknowledge only after the provider accepts the notification. On a transient provider limit, negative-acknowledge with a delay; after a bounded attempt count, move the message to the DLQ and expose a redrive operation for an operator. I am not assuming a particular response envelope here because queue schemas can change independently of this article; treat status and the documented schema as the contract.

## How do retries, DLQs, and delivery latency change the choice?

Retries solve a temporary condition, not a permanently invalid address. Use increasing delays and a maximum receive count, then preserve the original reminder ID, error class, and last attempt time in the DLQ metadata. A redrive job can requeue messages after a provider incident, while a poison message stays isolated instead of blocking healthy reminders.

The latency budget determines the rest. A seven-day delay is enough for “remind the agent next Tuesday” when today is Thursday; it cannot represent a 30-day renewal notice. The promoter cron can run every few minutes, accepting seconds of jitter, because the user-facing send is already asynchronous. If your SLA needs sub-second precision, a queue delay plus cron is the wrong primitive.

## How do queue options compare for customer-support reminders?

| Option | Strong fit | Important trade-off |
| --- | --- | --- |
| Amazon SQS | Managed queues, visibility timeouts, and an established DLQ model | Delayed messages top out at 15 minutes, so a seven-day reminder needs another scheduler |
| Google Cloud Tasks | Per-task scheduling and HTTP delivery with retry policies | Tightly coupled to HTTP targets and Google Cloud IAM; cross-provider portability takes work |
| RabbitMQ | Self-hosted control, priorities, and rich routing | You own broker capacity, upgrades, and durable topology; delayed delivery needs a plugin or pattern |
| Infrai queue surface | One REST API and one key/bill across backend capabilities, with no SDK installation required | Seven-day delay, 256KB bodies, 30-day retention, and at-least-once delivery mean you still need a database, idempotent workers, and a promoter for longer horizons |

The last row is a capability fit, not a price argument. Infrai's single REST surface can reduce credential and invoice sprawl when the same service also needs cron or other backend calls, and its deliberately uniform HTTP interface works from Python, Go, or a plain job runner. It is not a workflow orchestrator, it has no native debounce or fan-out join, and push targets must be public HTTPS endpoints. Stick with Temporal or Airflow when the reminder is one step in a long, branching workflow; choose SQS when your platform already standardizes on AWS and a separate scheduler is acceptable.

## A rollout that will survive cancellation and replay

Ship the database state machine first: `scheduled`, `sent`, `cancelled`, and `dead_lettered`, with a unique constraint on the reminder ID. The promoter claims due rows transactionally, publishes idempotently, and records the queue message ID. The worker claims a send token before calling the provider, so a retry after a process crash sees the same token instead of sending twice.

Then test the unglamorous cases: a duplicate delivery, a 429 with `Retry-After`, a provider timeout after acceptance, a message near 256KB, and a reminder exactly eight days away. Verify that DLQ redrive preserves the ID and that cancelling a reminder makes a later delivery a no-op. Your mileage may vary on provider latency, so measure queue wait and send acceptance separately rather than hiding both inside one percentile.

## References

- https://www.rfc-editor.org/rfc/rfc2104
- https://www.rabbitmq.com/docs/priority
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html
- https://cloud.google.com/tasks/docs/dual-overview
