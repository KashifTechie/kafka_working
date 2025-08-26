

## **Step 0: Initialization**

```python
producer = Producer({'bootstrap.servers': 'localhost:9092'})
```

* Producer object is created.
* Internal structures:

  * **Buffer**: holds messages in RAM before sending to broker.
  * **Queue for callbacks**: stores callback functions for messages.

---

## **Step 1: Call `produce()`**

```python
producer.produce("mytopic", payload, callback=delivery_report)
```

* Message **goes into producer’s internal buffer (RAM)**.
* Associated callback (`delivery_report`) is stored with the message in the buffer.
* At this point, message is **not yet on the broker**.

---

## **Step 2: Sending messages to the broker**

* The producer’s background thread sends messages in **batches** over TCP to the Kafka broker.
* The buffer may **temporarily block produce()** if full (BufferError).
* Once sent, the messages are **in flight**; they are not yet guaranteed to be on the broker.

---

## **Step 3: Broker appends messages**

* Broker receives the batch → writes messages **sequentially to partition log on disk**.
* Because Kafka is append-only, this is **very fast**.
* The broker now has the messages durably stored.

---

## **Step 4: Broker sends feedback (ACK)**

* Broker sends feedback over TCP to the producer:

  * If `acks=0`: no feedback.
  * If `acks=1`: leader acknowledges after write.
  * If `acks=all`: all replicas confirm.
* This feedback is exactly the **ACK we call “broker feedback”**.

---

## **Step 5: Feedback enters producer buffer**

* Producer receives feedback.
* Internal buffer marks the message as **delivered** (or failed if error).
* Feedback has **not yet triggered the callback**. It is stored as an event to process.

---

## **Step 6: Polling triggers callback**

```python
producer.poll(timeout)
```

* Checks for **events** in the producer:

  * Incoming feedback (ACKs).
  * Delivery errors.

* For each feedback event:

  * Marks message delivered/failed.
  * Calls **associated callback** (`delivery_report`).

* **Without poll()**:

  * Feedback is received but callback never fires.
  * Messages are still delivered to broker, buffer may eventually flush via `flush()`.

---

## **Step 7: Flush frees buffer**

```python
producer.flush()
```

* Calls `poll()` repeatedly until all messages are confirmed by broker.
* Clears messages from buffer → RAM freed.
* Ensures **all messages are delivered** before program exits.

---

### **Summary Timeline**

| Step | Action                        | Buffer                  | Broker          | Callback               |
| ---- | ----------------------------- | ----------------------- | --------------- | ---------------------- |
| 1    | `produce()`                   | Message added to buffer | -               | Stored for later       |
| 2    | Background thread sends batch | In flight               | -               | -                      |
| 3    | Broker writes message         | -                       | Appended to log | -                      |
| 4    | Broker sends ACK              | Received into buffer    | -               | -                      |
| 5    | `poll()` called               | Reads feedback          | -               | Fires callback         |
| 6    | `flush()` called              | Buffer cleared          | -               | All callbacks executed |

---

✅ **Key points**

* `produce()` → puts message in buffer.
* `poll()` → processes feedback & calls callback.
* `flush()` → blocks until all messages confirmed & callbacks fired.
* Callback is **never called automatically** without `poll()` (or `flush()` which internally polls).


